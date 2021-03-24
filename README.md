[TOC]

# React

## Deployment

Install nodejs and NPM

```text
apt install nodejs npm
```

Clone project

```text
git clone  https://github.com/htw-db/htw_db_frontend.git
```

Install npm packages

```text
npm install
```

Setup environment _.env_

```text
REACT_APP_BASE_URL = 'https://db1.f4.htw-berlin.de:9000'
REACT_APP_ENDPOINT_LOGIN = 'login'
REACT_APP_ENDPOINT_INSTANCES = 'instances'

// Database Server
REACT_APP_DB_HOSTNAME = 'db1.f4.htw-berlin.de'

// phppgadmin
REACT_APP_PHPPGADMIN_URL = 'https://db1.f4.htw-berlin.de/phppgadmin/'
```

Build project

```text
npm run build
```

Move build files to apache

```text
cp -r build/* /var/www/html
```

## Handling React Routes with Apache

Make sure that RewriteEngine is running

```text
sudo a2enmod rewrite
```

Add lines to enable RewriteEngine

```text
nano /etc/apache2/sites-enabled/000-default.conf

<Directory "/var/www/html">
    RewriteEngine on
    # Don't rewrite files or directories
    RewriteCond %{REQUEST_FILENAME} -f [OR]
    RewriteCond %{REQUEST_FILENAME} -d
    RewriteRule ^ - [L]
    # Rewrite everything else to index.html to allow html5 state links
    RewriteRule ^ index.html [L]
</Directory>
```

Restart apache2 to reload configurations

```text
systemctl restart apache2
```

# Play Framework

## Requirements

### Install Scala

Install the default JDK package

```text
apt-get install default-jdk -y
```

Download the necessary scala .deb file

```text
wget www.scala-lang.org/files/archive/scala-2.13.0.deb
```

Install the Scala .deb file

```text
sudo dpkg -i scala*.deb
```

### Install SBT

Add necesssary repository

```text
echo "deb https://dl.bintray.com/sbt/debian /" | sudo tee -a /etc/apt/sources.list.d/sbt.list
```

Add public key for installation

```text
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2EE0EA64E40A89B84B2DF73499E82A75642AC823
apt-get update
```

Finally install sbt

```text
apt-get install sbt -y
```

Once the installation is complete, test to make sure all is working

```text
sbt test
```

## Deploy Play project

Clone project

```text
git clone https://github.com/htw-db/htw_db_backend.git
```

Configure environment

```text
nano conf/application.conf
```

Setup following environment variables

```text
play.http.secret.key=${?APPLICATION_SECRET}
jwt.secretKey = "secretKey"
slick.dbs.default.db.url="jdbc:postgresql://localhost:5432/production"
slick.dbs.default.db.user="postgres"
slick.dbs.default.db.password="1234"
```

Compile files

```text
sbt dist
```

Run project and override the http secret key

```text
cd target/universal
unzip htw_db_backend-1.0-SNAPSHOT.zip
 ./htw_db_backend-1.0-SNAPSHOT/bin/htw_db_backend -Dplay.http.secret.key='KEY'
```

Run as daemon process

```text
screen -S backend
```

## Integrate HTTPs

The organization has give following files

```bash
root@db1:/home/local# ls -la cert
total 24
drwx------ 2 root  root  4096 Jan 14 10:48 .
drwxr-xr-x 8 local local 4096 Feb 19 15:59 ..
-rw-r--r-- 1 root  root  5575 Jan 14 10:48 cacert-HTW.pem
-rw-r--r-- 1 root  root  2813 Jan 14 10:46 db1.f4.htw-berlin.de.cert
-rw-r--r-- 1 root  root  1675 Jan 14 10:47 db1.f4.htw-berlin.de.key
```

Create a keystore with given credentials

```bash
openssl pkcs12 -export -in db1.f4.htw-berlin.de.cert -inkey db1.f4.htw-berlin.de.key -out keystore.p12
```

The process will asked for a Keystore password. Note that password. Now we can start our project with following command

```bash
./htw_db_backend-1.0-SNAPSHOT/bin/htw_db_backend -Dplay.http.secret.key='SECRET KEYT' -Dhttps.port=9443 -Dplay.server.https.keyStore.path='/home/local/cert/keystore.p12' -Dplay.server.https.keyStore.password='KEYSTORE_PW' -Dplay.server.https.keyStore.type='pkcs12'
```

## Migrate new code changes

First we stash our local changes (.env file)

```bash
git stash
```

Then we pull changes

```bash
git pull
```

Then pop our local changes back

```bash
git stash pop
```

# phpPgAdmin

## Deployment

phpPgAdmin is available in the base Debian repository, so it can be easily installed using the following command

```text
apt-get install phppgadmin
```

Change the configuration of phppgadmin

```text
 nano /etc/phppgadmin/config.inc.php
 $conf['owned_only'] = true;
```

Change the apache2 specific configuration

```text
nano /etc/apache2/conf-enabled/phppgadmin.conf

# require local
Require all granted
```

# Apache HTTP Server

## Deployment

Installing Apache

```text
 apt install apache2
```

Make sure the service is running

```text
systemctl status apache2
```

## HTTPS Mode

Edit the apache enabled config

```
/etc/apache2/sites-enabled/000-default.conf
```

Add following lines

```bash
<VirtualHost *:443>
    SSLEngine On
    SSLCertificateFile /home/local/cert/db1.f4.htw-berlin.de.cert
    SSLCertificateKeyFile /home/local/cert/db1.f4.htw-berlin.de.key
    SSLCACertificateFile /home/local/cert/cacert-HTW.pem

    ServerAdmin info@db1.f4.htw-berlin.de
    ServerName db1.f4.htw-berlin.de
    ServerAlias db1.f4.htw-berlin.de
    DocumentRoot /var/www/html/
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    
    <Directory "/var/www/html">
        RewriteEngine on
        # Don't rewrite files or directories
        RewriteCond %{REQUEST_FILENAME} -f [OR]
        RewriteCond %{REQUEST_FILENAME} -d
        RewriteRule ^ - [L]
        # Rewrite everything else to index.html to allow html5 state links
        RewriteRule ^ index.html [L]
    </Directory>

</VirtualHost>
```

## Redirect HTTP to HTTPS

Change lines in VirtualHost for Port 80

```bash
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
```

Enable redirect

```
a2enmod rewrite
service apache2 restart
```

# PostgreSQL 

**PostgreSQL** is an open-source object-relational database system.

## Setup PostgreSQL PPA

See [https://www.postgresql.org/download/linux/debian/](https://www.postgresql.org/download/linux/debian/)

Create the file repository configuration

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
```

Import the repository signing key

```bash
apt-get install gnupg2
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```

Update the package lists

```bash
sudo apt-get update
```

Install the latest version of PostgreSQL

```text
sudo apt-get -y install postgresql
```

The service is usually started after installation.

```text
systemctl status postgresql
```

After installing the PostgreSQL database server by default, it **creates a user postgres** with role postgres. It also creates a system account with the same name postgres. So to connect to postgres server, login to your system as user postgres and connect the database.

```text
su - postgres
psql -c "alter user postgres with password 'StrongDBPassword'"
ALTER ROLE
```

## Enable remote access

By default, access to PostgreSQL database server is only from localhost. Edit PostgreSQL configuration file if you want to change listening address. 

```text
nano /etc/postgresql/13/main/postgresql.conf
```

Add the line under connections and authentication

```text
listen_addresses = '*' 
```

Restart postgresql service to enable changes

```text
sudo systemctl restart postgresql
```

Add iptables rules

```text
nano firewall.sh
```

PostgreSQL listens for client connections on port 5432.  To allow incoming PostgreSQL connections from a specific IP address or subnet

```text
iptables -A INPUT -p tcp -s 0.0.0.0/0 --dport 5432 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -p tcp --sport 5432 -m conntrack --ctstate ESTABLISHED -j ACCEPT
```

## Client authentication with LDAP

Client authentication is controlled by a configuration file. which is named pg\_hba.conf

```text
nano /etc/postgresql/13/main/pg_hba.conf
```

Add the following line

```text
host    all   all     all  ldap ldapscheme="ldaps" ldapserver="login-dc-01.login.htw-berlin.de" ldapprefix="cn=" ldapsuffix=", ou=idmusers,dc=login,dc=htw-berlin,dc=de" ldapport=636
```

The service need be restarted

```text
systemctl restart postgresql
```

## Understand the workflow

### Gathering general information

Login to your system as user postgres and connect the database.

```text
$ psql
psql (13.2 (Debian 13.2-1.pgdg110+1))
Type "help" for help.
```

To check login info use following command from the database command prompt.

```text
postgres=# \conninfo
You are connected to database "postgres" as user "postgres" via socket in "/var/run/postgresql" at port "5432".
```

List roles

```text
postgres=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

There is another way to display all roles.  Notice that the roles that start with with `pg_` are system roles.

```text
postgres=# SELECT rolname FROM pg_roles;
          rolname
---------------------------
 pg_monitor
 pg_read_all_settings
 pg_read_all_stats
 pg_stat_scan_tables
 pg_read_server_files
 pg_write_server_files
 pg_execute_server_program
 pg_signal_backend
 postgres
```

List databases

```text
postgres=# \l
                             List of databases
   Name    |  Owner   | Encoding  | Collate | Ctype |   Access privileges
-----------+----------+-----------+---------+-------+-----------------------
 postgres  | postgres | SQL_ASCII | C       | C     |
 template0 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
           |          |           |         |       | postgres=CTc/postgres
 template1 | postgres | SQL_ASCII | C       | C     | =c/postgres          +
           |          |           |         |       | postgres=CTc/postgres
```

### The actual workflow

Authentication takes place via LDAP. So that these users are all assigned to one group. This group has already been created in the deployment in the form of a role.

```text
postgres=# CREATE ROLE ldap_users;
CREATE ROLE
```

When a user authenticates for the first time, a new role is created in the database and added to that specific group. `CONNECTION LIMIT` specifies the number of concurrent connections a role can make.

```text
postgres=# CREATE ROLE s0558151 WITH NOSUPERUSER LOGIN CONNECTION LIMIT 10 IN ROLE ldap_users;
CREATE ROLE
```

Lets verify that we added a role to a specific group

```text
postgres=# \du
                                     List of roles
 Role name  |                         Attributes                         |  Member of
------------+------------------------------------------------------------+--------------
 ldap_users | Cannot login                                               | {}
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
 s0558151   | 10 connections 
```

As soon as a user creates a database, the following command is executed. The user is entered as the owner of the database.

```text
 CREATE DATABASE s0558151_firstdb WITH OWNER s0558151;
```

If the user deletes his database, the following command will be executed.

```text
DROP DATABASE IF EXISTS s0558151_firstdb;
```

# Appendix

/etc/apache2/sites-enabled/000-default.conf

```bash

<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        #ServerName www.example.com

        ServerAdmin webmaster@localhost
        RewriteEngine On
        RewriteCond %{HTTPS} off
        RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        #Include conf-available/serve-cgi-bin.conf
</VirtualHost>

<VirtualHost *:443>
    SSLEngine On
    SSLCertificateFile /home/local/cert/db1.f4.htw-berlin.de.cert
    SSLCertificateKeyFile /home/local/cert/db1.f4.htw-berlin.de.key
    SSLCACertificateFile /home/local/cert/cacert-HTW.pem

    ServerAdmin info@db1.f4.htw-berlin.de
    ServerName db1.f4.htw-berlin.de
    ServerAlias db1.f4.htw-berlin.de
    DocumentRoot /var/www/html/
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined


    <Directory "/var/www/html">
        RewriteEngine on
        # Don't rewrite files or directories
        RewriteCond %{REQUEST_FILENAME} -f [OR]
        RewriteCond %{REQUEST_FILENAME} -d
        RewriteRule ^ - [L]
        # Rewrite everything else to index.html to allow html5 state links
        RewriteRule ^ index.html [L]
    </Directory>

</VirtualHost>
```

