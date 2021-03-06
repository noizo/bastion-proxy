## Configure Bastion

Finally the bastion needs to be update with the MySQL servers that it will be servicing.  Then Vault will need to be configured with the roles for the users to request credentials.

**Add MySQL Servers to the MySQL Inventory**

Login to the MySQL Inventory server and insert the MySQL servers that will be accessed with the Bastion

``` bash
mysql
USE mysql_inventory;

INSERT INTO hosts(host,ip) VALUES('server1.fqdn.com','192.168.0.20');
INSERT INTO hosts(host,ip,port) VALUES('server2.fqdn.com','192.168.0.21',3301);
```

!! The sync user will be used to access the local MySQL server as well as the remote MySQL servers that the credentials will sync to. You will need to create the sync user account on each MySQL server that requesting users will access from using the Bastion/Proxy server. Account should have full privileges with GRANT OPTION !!

**Create Vault Roles**

In order to make the vault changes, you will have to make sure that the two environment variables have been exported for VAULT_ADDR and VAULT_TOKEN, and that the vault is unsealed.  If you had to reboot and you need to unseal your vault, please see https://github.com/kevinmarkwardt/mysql-bastion/blob/master/docs/Vault.md

**DB Config**

First we need to configure the connectivity from Vault to the local mysql instance, so it can store the credneitals that are requested for MySQL.  Then the sync script will sync these credentials to ProxySQL and the MySQL server that the user is connecting to.  Make sure you update the code below with the "MySQL local root password" ROOT password that was changed previously.  First mount the database plugin, and then run configuration script

For older MySQL version that require a shorter username, replace "mysql-database-plugin" with "mysql-legacy-database-plugin"

```bash
vault mount database

vault write database/config/mysql \
    plugin_name=mysql-database-plugin \
    connection_url="root:<PASSWORD GOES HERE FOR ROOT>@tcp(mysql_local:3306)/" \
    allowed_roles="*"
```

**OLDER MySQL Versions** with shorter username restrictions.  If you decide to use the legacy database plugin, you will have to make sure you update the prefix in the mysql-proxy-credential-sync script as it's currently default set to v-ldap.  When using this plugin the prefix for the user accounts will start with v-serv

```bash
vault mount database

vault write database/config/mysql \
    plugin_name=mysql-legacy-database-plugin \
    connection_url="root:<PASSWORD GOES HERE FOR ROOT>@tcp(mysql_local:3306)/" \
    allowed_roles="*"
```

**Roles**

Next is time to create the roles within vault where the user will request access to.  Below is an example of recreating two roles for a specific server.  The first role is a read only role for the server.  You will want to update the commands with the following information .  You will have to create roles that will fit your needs.  

- **Role Name** :   In the below example it's service_ro_servername and service_rw_servername.  Update with what makes sense for your environment
- **MYSQL_SERVER_NAME** : Please uses the exact name or IP that was inserted into the mysql_inventory. If you use % as the MYSQL_SERVER_NAME, then you do NOT need the revocation_statements line for dropping the user.  
- **TTL** : Update the time to live for the connections accordinly to how long you want the credentials to stay around.

```bash
vault write database/roles/service_ro_server1 \
  db_name=mysql \
  default_ttl="1h" max_ttl="24h" \
  revocation_statements="DROP USER '{{name}}'@'MYSQL_NAME_OR_IP';" \
  creation_statements="CREATE USER '{{name}}'@'MYSQL_NAME_OR_IP' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'MYSQL_NAME_OR_IP';"
  
vault write database/roles/service_rw_server1 \
  db_name=mysql \
  default_ttl="1h" max_ttl="24h" \
  revocation_statements="DROP USER '{{name}}'@'MYSQL_NAME_OR_IP';" \
  creation_statements="CREATE USER '{{name}}'@'MYSQL_NAME_OR_IP' IDENTIFIED BY '{{password}}';GRANT INSERT ON *.* TO '{{name}}'@'MYSQL_NAME_OR_IP';"
  
vault write database/roles/service_ro_server2 \
  db_name=mysql \
  default_ttl="1h" max_ttl="24h" \
  revocation_statements="DROP USER '{{name}}'@'MYSQL_NAME_OR_IP';" \
  creation_statements="CREATE USER '{{name}}'@'MYSQL_NAME_OR_IP' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'MYSQL_NAME_OR_IP';"
  
vault write database/roles/service_rw_server2 \
  db_name=mysql \
  default_ttl="1h" max_ttl="24h" \
  revocation_statements="DROP USER '{{name}}'@'MYSQL_NAME_OR_IP';" \
  creation_statements="CREATE USER '{{name}}'@'MYSQL_NAME_OR_IP' IDENTIFIED BY '{{password}}';GRANT INSERT ON *.* TO '{{name}}'@'MYSQL_NAME_OR_IP';"
```

List Roles

```bash
root@bastion:~/mysql-bastion# vault list database/roles
Keys
----
service_ro_server1
service_ro_server2
service_rw_server1
service_rw_server2
```

**LDAP Authentication**

You can configure Vault to validate request authentication using LDAP.  The groupfilter and all configuration that were used are based upon OpenLDAP.  Some modifications may be needed if you are using Active Directory.  

The url for the configuration below is if you are using the openldap docker instance.  If you are using a remote LDAP configuration, update the url accordingly.  

You will also need to update the account that will be used to authenticate to LDAP using the binddn and bindpass

Finally update the userdn and groupdn where the users and groups will be stored that Vault will need to authenticate with.

```bash
vault auth-enable ldap

vault read auth/ldap/config

vault write auth/ldap/config \
  url="ldap://openldap" \
  binddn="cn=admin,dc=proxysql,dc=com" \
  bindpass="password" \
  userdn="ou=users,dc=proxysql,dc=com" \
  userattr="uid" \
  groupdn="ou=groups,dc=proxysql,dc=com" \
  groupattr="cn" \
  groupfilter="(|(memberUid={{.Username}})(member={{.UserDN}})(uniqueMember={{.UserDN}}))" \
  insecure_tls=true
  
vault read auth/ldap/config
```

**Policies**

Finally the policies will need to be configured.  Policies tell vault who to allow access to read credentials from specific roles.  If you dont' configure policies then no one will have access to request credentials.  

You can use wildcards to allow multiple roles to be associated to a single policy easier.  In the example below the database/* is given list permission, so that an end user can view the policies that are available.  But the user will only be able to request credentials from policies that they are granted access to.

```bash
vault policy-write env_service_mysql_ro -<<EOF
path "database/creds/service_ro_*" {
  capabilities = ["list", "read"]
}

path "database/*" {
  capabilities = ["list"]
}
EOF

vault policy-write env_service_mysql_rw -<<EOF
path "database/creds/service_rw_*" {
  capabilities = ["list", "read"]
}

path "database/*" {
  capabilities = ["list"]
}
EOF
```

List Policies

```bash
root@bastion:~/mysql-bastion# vault policies
default
env_service_mysql_ro
env_service_mysql_rw
root
```

**LDAP Group Auth to Policy Commands**

These will map the LDAP group to the Vault Policy.  So users can be managed within LDAP.

```bash
vault write auth/ldap/groups/<LDAP GROUP> policies=<POLICY NAME>

examples:

vault write auth/ldap/groups/prod_service_server1_ro policies=env_service_mysql_ro
vault write auth/ldap/groups/prod_service_server1_rw policies=env_service_mysql_rw
```

List Auth Group Mappings, and read the policies that it's associated to

```bash
root@bastion:~/mysql-bastion# vault list auth/ldap/groups
Keys
----
prod_service_server1_ro
prod_service_server1_rw

root@bastion:~/bastion-proxy# vault read auth/ldap/groups/prod_service_server1_ro
Key     	Value
---     	-----
policies	[env_service_mysql_rw]

```

## Starting Sync Script

mysql-proxy-credential-sync is the script that will sync the credentials that Vault creates in the local MySQL instance to ProxySQL and the MySQL server where the user is trying to access.  

- Make sure the MySQL configuration is complete with both of these logins working.
mysql --defaults-file=~/.proxysql.cnf
mysql --defaults-file=~/.my.cnf
- Make sure the remote credentials have been created on the remote servers with full privileges with GRANT OPTION.
- You can run the script on the command line to see the interaction it has and to trouble shoot any problems that it may encounter.  After that, it can set to start on server boot by placing it in /etc/rc.local

## Using the Bastion/Proxy

Now that the bastion is configured, it's ready to be used. Here is an example of using the bastion proxy to grain credentials and login.

https://github.com/kevinmarkwardt/bastion-proxy/blob/master/docs/Using.md

## Manual Configuration

https://github.com/kevinmarkwardt/bastion-proxy/blob/master/docs/Manual_Install.md
