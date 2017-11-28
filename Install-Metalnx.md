# Installation of Metalnx on Ubuntu 14.04 and Ubuntu 16.04
This document describes how to install Metalnx on a Ubuntu machine with a working iRODS instance on the same machine. Note that this document closely follows the "Getting Started" page on the GitHub of metalnx-web with some adjustments: 

https://github.com/Metalnx/metalnx-web/wiki/Getting-Started

## Environment
Ubuntu 14.04 server

## Prerequisites

### 1. Update and upgrade if necessary
```sh
sudo apt-get update
sudo apt-get upgrade
```

### 2. Firewall settings and hostname settings
#### Firewall

Edit /etc/iptables/rules.v4:

```sh
sudo vim /etc/iptables/rules.v4

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [4538:480396]
-A INPUT -m state --state INVALID -j DROP
-A INPUT -p tcp -m tcp ! --tcp-flags FIN,SYN,RST,ACK SYN -m state --state NEW -j DROP
-A INPUT -f -j DROP
-A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP
-A INPUT -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG FIN,SYN,RST,PSH,ACK,URG -j DROP
-A INPUT -p icmp -m limit --limit 5/sec -j ACCEPT
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
# ssh
-A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
# HTTP
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
# HTTPS
-A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
# iRODS
-A INPUT -p tcp -m tcp --dport 1248 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 1247 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 20000:20199 -j ACCEPT
# postgrsql
-A INPUT -p tcp -m tcp --dport 5432 -j ACCEPT
# Tomcat HTTP and HTTPS
-A INPUT -p tcp -m tcp --dport 8080 -j ACCEPT
-A INPUT -p tcp -m tcp --dport 8443 -j ACCEPT
-A INPUT -j LOG
-A INPUT -j DROP
COMMIT
```

In Ubuntu 14 it is enough to do a:

```sh
sudo /etc/init.d/iptables-persistent restart
```

In Ubuntu 16 you need to do the following to save your iptables comfiguration:
- Store the firewall settinigs above in a file `~/iptables-rules/ruleset-v4`
 
 ```sh
 sudo iptables-restore < iptables-rules/ruleset-v4
 ```
- Verify the settings with 
 
 ```sh
 sudo iptables -L
 ```
 and save them
 
 ```
 sudo su 
 iptables-save > /etc/iptables/rules.v4
 cat /etc/iptables/rules.v4
 reboot
 ```
 
### 2. Dependencies
#### Java 8
Test whether your operating system already comes with Java 8:

```sh
which java
java -version
```
E.g.






```sh
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
```

If not you can install Java 8 with the system package manager which for Ubuntu 16.04 is:

```sh
sudo apt-get install default-jre
sudo apt-get install default-jdk
```

And for Ubuntu 14.04:

```
sudo apt-add-repository ppa:webupd8team/java
sudo apt-get update
sudo apt-get install oracle-java8-installer
sudo apt-get install oracle-java8-set-default
sudo ln -s /usr/lib/jvm/java-8-oracle /usr/lib/jvm/default-java
```

Export ```JAVA_HOME```:

```sh
export JAVA_HOME=/usr/lib/jvm/java-8-<provider>/
```

By default Java will be installed under */usr/lib/jvm*. If you installed Java manually you might have manually set a different ```JAVA_HOME```.

For CentOS 7 do:

```sh
sudo yum install jre
```


#### Python 2.7

```sh
which python
python --version
```

#### Tomcat Installation

```sh
sudo apt-get install -y tomcat7
```


Try to restart Tomcat:

```sh
sudo service tomcat7 restart
```

For CentOS do:

```sh
sudo yum install tomcat
sudo yum install tomcat-webapps tomcat-admin-webapps
sudo systemctl -l restart tomcat
```

Test whether Tomcat7 is installed and reachable by openening a  browser and going to the page:
`http://<ip-address of server\>:8080`


#### PostgreSQL 9.2 or higher
If you install Metalynx on the same server as where the iRODS server is running on, you probably already have a PostgreSQL data base version 9.3.

If you are working on a different machine then please install:

```sh
sudo apt-get install postgresql postgresql-contrib
sudo postgresql-setup initdb
```


```sh
sudo apt-get install python-psycopg2
```

The ```pg_hba.conf``` maybe needs to be edited which could be in ```/home```,```/var/lib/pgsql```, ```/var/lib/postgresql/[version]/```, ```/opt/postgres/```, ```/etc/postgresql/[version]/main```, etc. 

Open in text editor and find the following lines:

```sh
host	all 	all 	127.0.0.1/32		ident
host	all 	all 	::1/128				ident
```

Replace ```ident``` with ```md5``` or ```trust```:

```sh
host	all 	all 	127.0.0.1/32		md5
host	all 	all 	::1/128				md5
```

Again, it could be that this is already done correctly during the iRODS installation.


#### MySQL installation
During the Metalnx setup script you can choose to either use PostgreSQL or MySQL for the Metalnx Database. Here we choose PostgreSQL. However, if you want to use MySQL, you should install MySQL server 5.6 or higher. Note that the instructions to set up the MySQL database can be followed on the Metalnx 'Getting Started' pages. In Ubuntu 16 these packages are not part of the standard software packages. So for Ubuntu 16 you have to add it to the repository:

```sh
#Ubuntu 16
apt-get install software-properties-common 
sudo add-apt-repository 'deb http://archive.ubuntu.com/ubuntu trusty universe'
sudo apt-get update
```

And subsequently install MySQL server and client, which is the same command for Ubuntu 14 and 16:

```sh
#Ubuntu 14 and 16
sudo apt install mysql-server-5.6
sudo apt install mysql-client-5.6
```

Also install mysqldv:

```sh
sudo apt-get install python-mysqldb
```






### 3. Install Metalnx

#### PostgreSQL
Become the user ```postgres``` using the command:

```sh
sudo su - postgres
```

Now create user and database 'metalnx' with the ```psql``` utility:

```sh
psql
CREATE USER metalnx WITH PASSWORD 'metalnx';
CREATE DATABASE "metalnx";
GRANT ALL PRIVILEGES ON DATABASE "metalnx" TO metalnx;
\q
exit
```

#### Set iRODS Negotiation
Before running the Metalnx setup script, make sure your iRODS negotiation parameters are correct. By default, iRODS is configured as  ```CS_NEG_DONT_CARE``` in the ```core.re``` file, which means that the server can use SSL or not to communicate with the client. ```CS_NEG_REQUIRE``` and ```CS_NEG_REFUSE``` can also be used. ```CS_NEG_REQUIRE``` means that iRODS will always use SSL communication while ```CS_NEG_REFUSE``` tells iRODS not to use SSL at all. 

<!--
For now we will not use SSL. We will change the ```core.re``` file directly (note that in the "Getting Started" page of the Metalnx wiki, other options are presented). 
-->

##### No SSL
Open ```/etc/irods/core.re``` in a text editor. Warning: be very carefull with the file as it can break your iRODS instance very easily. Replace:

```acPreConnect(*OUT) { *OUT="CS_NEG_DONT_CARE"; }```

with 

```acPreConnect(*OUT) { *OUT="CS_NEG_REFUSE"; }```


Also, adjust the environment-json:

```sh
 vi /home/rods/.irods/irods_environment.json
 
 ...
 "irods_client_server_policy": "CS_NEG_REFUSE",
 ...
 ```

##### Using SSL
If you want to use SSL, you first have to configure iRODS with SSL.

1. **Generate the SSL key and certificate** on the server that runs iRODS.
Remember the names you gave to the files: `irods.key`, `irods.crt`. Also note that you can change the number of days (now 365) for the certificate. In this example, the user is called 'rods'. 

```sh
 sudo su - rods
 mkdir /etc/irods/ssl
 cd /etc/irods/ssl
 openssl genrsa -out irods.key 2048
 chmod 600 irods.key
 openssl req -new -x509 -key irods.key -out irods.crt -days 365
 ```
 
  
 You are asked to provide some details. Upon login your users will have to use the common name of the server:
 
 ```sh
 Country Name (2 letter code) [XX]:XX
State or Province Name (full name) []:<your state>
Locality Name (eg, city) [Default City]:<your city>
Organization Name (eg, company) [Default Company Ltd]:<company>
Organizational Unit Name (eg, section) []:<group>
Common Name (eg, your name or your server's hostname) []:<ip address or fqdn>
Email Address []:<email>
 ```
And finally create the dhparams.pem file. 
 
 ```sh
 openssl dhparam -2 -out dhparams.pem 2048
 ```
 
2. **Adjust the /etc/irods/core.re** with
 
 ```sh
 acPreConnect(*OUT) { *OUT="CS_NEG_REQUIRE"; }
 ```
3. **Adjust the environment-json for the irods service account.**

 You need to set the server certificate (*irods.crt*) and its corresponding key (*irods.key*) and the certificate from the "Certificate Authority" (here we use again *irods.crt* (usually you would have a *chain.pem*), if you use a different authority make sure all machines that run clients have this file installed). We also need to set the file defining how keys are exchanged (*dhparams.pem*). Finally we need to tell iRODS that we are using ssl verification by certificate.
 
 ```sh
 vi /home/rods/.irods/irods_environment.json
 
 "irods_client_server_policy": "CS_NEG_REQUIRE",
 "irods_ssl_certificate_chain_file": "/etc/irods/ssl/irods.crt",
 "irods_ssl_certificate_key_file": "/etc/irods/ssl/irods.key",
 "irods_ssl_dh_params_file": "/etc/irods/ssl/dhparams.pem",
 "irods_ssl_ca_certificate_file": "/etc/irods/ssl/irods.crt",
 "irods_ssl_verify_server": "cert"
 ```
 Make sure common name of the server in the certificate and the *irods_host* in the environment json file match.
 Then try as a rodsuser/rodsadmin whether you can login:
 
 ```sh
 iinit
 ils
 ```
 
 If the connections fails due to the SSL connection, check if all files are set correctly in the environment-json and core.re file.
 
 ##### SSL and Metalnx
 For Metalnx, we need to tell the Java Virtual Machine to trust the irods certificate created earlier. This can be done by running the following command (note that you will be asked to create a password):

```sh
cd /etc/irods/ssl
keytool -import -alias irodscertificate -file /etc/irods/ssl/irods.crt -keystore
```

The command above will create a keystore called *irodskeystore* in ```/etc/irods/ssl```. This keystore is necessary for the ```setup_script.py``` to check whether or not Metalnx can securely connect to iRODS.

Finally, there is only one configuration to be done. Specify to Tomcat where the *irodskeystore* file is, so the Metalnx Web server is able to communicate with iRODS using SSL. To do so, modify the ```JAVA_OPTS``` option in the file ```/etc/default/tomcat7```:

```sh
JAVA_OPTS="-Djavax.net.ssl.trustStore=/etc/irods/ssl/irodskeystore -Djavax.net.ssl.trustStorePassword=<keystore-password>"
```

Don't forget to replace ```<keystore-password>``` with the password you created earlier.


#### Package Installation

```sh
wget -O emc-metalnx-webapp-1.1.1-3.deb https://bintray.com/metalnx/deb/download_file?file_path=pool%2Fe%2Femc-metalnx-web%2Femc-metalnx-webapp-1.1.1-3.deb
```

Install the Metalnx application using the command:

```sh
sudo dpkg -i emc-metalnx-webapp-1.X.X-X.noarch.deb
```

Note the 'X' in the name of the ```.deb``` file, which should be replaced with the correct version number of Metalnx you downloaded.

#### Setup Metalnx
The Metalnx installation package comes with a setup script. The script will help to setup the Metalnx in according to your environment. Once the rpm or deb package is installed, run the Metalnx setup script, as root:

```sh
sudo python /opt/emc/setup_metalnx.py
```

The script will follow 13 steps:

```sh
Executing config_java_devel (1/13)
...
Executing config_tomcat_home (2/13) 
...
Executing config_tomcat_shutdown (3/13)   # Shutdown the tomcat service
...
Executing config_metalnx_package (4/13)   # Checks if any Metalnx package already exists in your environment
 ...
Executing config_existing_setup (5/13)    # Saves current installed Metalnx configuration, if any
 ...
Executing config_war_file (6/13)          # Installs the Metalnx WAR file into Tomcat
```

In the seventh step it will ask you for the Metalnx Database configuration parameters as follows:

```sh
Executing config_database (7/13)                                            # Configures database access
 Enter the Metalnx Database type (mysql, postgresql) [mysql]: postgresql    # Metalnx database type. By default, it is MySQL
 Enter Metalnx Database Host [localhost]: <server-name>                     # Database hostname. By default, it is localhost.
 Enter Metalnx Database Port [3306]: 5432                                   # Database port. The default port is 3306 for MySQL and 5432 for PostgreSQL
 Enter Metalnx Database Name [metalnx]:                                     # Database name. By default, it is "metalnx".
 Enter Metalnx Database User [metalnx]:                                     # Database user. By default, it is "metalnx"
 Enter Metalnx Database Password (it will not be displayed):                # Database password. We do not provide any default value for that.
```

Note that in the previous steps we have configured the PostgreSQL database, not the MySQL. However, it seems that the setup script also checks for MySQL. Therefore, also install MySQL, see Dependencies above, to avoid warning messages such as:

```sh
No DB connection test modules detected. Skipping DB connection test.
Notice that if your DB params are incorrect, Metalnx will not work as expected.
To change these parameters, execute the configuration script again.
```

After the Metalnx database configuration, the script tests whether or not the credentials are valid by connecting to the database. If it is successful, the following message is shown:

```sh
* Testing database connection...
* Database connection successful.
```

Now, you will be configuring the iRODS parameters to allow Metalnx connect to the grid.

```sh
Enter iRODS Host [localhost]: <ip-address>                                             # iRODS machine's hostname. By default, it is localhost
Enter iRODS Port [1247]:                                                  # port number used to communicate with the iCAT. By default, it is 1247
Enter iRODS Zone [tempZone]: <yourZone>                                             # iRODS Zone Name. By default, it is tempZone
Enter Authentication Schema (STANDARD, GSI, PAM, KERBEROS) [STANDARD]:    # iRODS authentication mechanism. By default, it is STANDARD
Enter iRODS Admin User [rods]: <irods_admin>                                           # iRODS Admin Username. By default, it is rods.
Enter iRODS Admin Password (it will not be displayed):                    # iRODS Admin Password. We do not provide any default value for that.
```

Almost there:

```sh
Executing config_restore_conf (9/13)
Executing config_set_https (10/13)
Do you want to enable HTTPS on Tomcat and Metalnx? [Y/n]: Y
```

If you respond n Tomcat and Metalnx will be setup to respond to http protocol on port 8080. If you press Y or y it will configure Tomcat to use https for its connections on port 8443.

```sh
Executing config_confirm_props (11/13)
    - Confirm configuration parameters

Metalnx Database Parameters:
    * db.type = mysql
    * db.db_name = metalnx
    * db.port = 3306
    * db.host = < FQDN >
    * db.username = metalnx

iRODS Paramaters:
    * irods.port = 1247
    * irods.auth.scheme = STANDARD
    * irods.zoneName = <yourZone>
    * jobs.irods.username = <irods_admin>
    * irods.host = <ip address server>

Do you accept these configuration parameters? (yes, no) [yes]:
```

The 11th step asks you to confirm if all params you set for both iRODS and the Metalnx database are correct.

```sh
Executing config_confirm_props (11/13)
    - Confirm configuration parameters

Metalnx Database Parameters:
    * db.type = postgresql
    * db.db_name = metalnx
    * db.port = 5432
    * db.host = <your-server>
    * db.username = metalnx

iRODS Paramaters:
    * irods.port = 1247
    * irods.auth.scheme = STANDARD
    * irods.zoneName = <yourZone>
    * jobs.irods.username = <irods_admin>
    * irods.host = <ip address server>

Do you accept these configuration parameters? (yes, no) [yes]:

* Creating Database properties file...
* Database properties file created.
* Creating iRODS properties file...
* iRODS properties file created.

[*] Executing config_tomcat_startup (12/13)
   - Starting tomcat back again
Redirecting to /bin/systemctl start  tomcat.service

[*] Executing config_displays_summary (13/13)
   - Metalnx configuration finished

Metalnx configuration finished successfully!
You can access your Metalnx instance at:
    http://<ip-address>:8443/emc-metalnx-web/login/

For further information and help, refer to:
    https://github.com/Metalnx/metalnx-web
```

If everything is correctly installed and configured, you can view your iRODS instance via Metalnx on your browser with either of the following URLs:
```
http://<ip-address>:8080/emc-metalnx-web/login/
```
```
https://<ip-address>:8443/emc-metalnx-web/login/
```

### 4. Usage

If you go to the URL you are presented a simple login screen which asks for your iRODS username and password. 

Normal rodsusers will have access to all their collections and files. Users will also be able to search for files based on their (user defined or system) metadata, like ```iquest```. Metadata can also be added to files manually per file. 

Admin users have more functionaly and can view the resource tree, add resources, add/modify users, add/modify groups and of course everything normal users can do.

#### Some quirks
- Folders can not be added at once via the Metalnx interface. 
- When selecting a data object, almost every action does not work except for the action 'remove'. 
- Download (iget) of data objects also does not work and only shows a new empty tab with the suffix 'fileOperation/download/'. 
- Metadata templates seem nice. You can specify a certain list of metadata that in principle you can immediately add to a file. However,this does not seem to work
