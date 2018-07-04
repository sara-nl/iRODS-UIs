# Davrods install guide
**Authors**

Matthew Saum (SURFsara)

Christine Staiger (SURFsara)

**Contributors**

Merret Buurman (DKRZ)

## Synopsis
This guide will take you through all steps to deploy an SSL-enabled Davrods instance connected to an SSL-enabled iRODS instance. 
In principle you also only enable Davrods with SSL and direct it to an iRODS instance that is not SSL-enabled and vice-versa. In this guide we will also give hints which steps to omit to skip the SSL-enabling for Davrods and iRODS.

With Davrods users can get access to iRODS by the webdav protocol and mount the iRODS logical filesystem to their workstations' filesystems or access iRODS with tools like Cyberduck or Filezilla.

Davrods uses an Apache HTTP server.
Since with a normal HTTP users would send passwords in an unencrypted way over the internet we show here how to first enable your iRODS instance with SSL encryption and subsequently how to deploy Davrods using a secure Webdav-SSL connection employing HTTPS. So both, the iRODS and the HTTP Server make use of SSL certificates.

In this tutorial we will deploy all software, i.e. Apache HTTP server and Davrods on the same server that also runs iRODS. In principal you can deploy Davrods on a separate server which runs the icommands.

Since there is no installation package of Davrods for Ubuntu yet, we use a Centos 7 machine.
 
We are using an iRODS 4.1.11 server, please note that the installation of Davrods slightly differs for iRODS 4.2.1

### Compatability
|iRODS | Davrods
|----|----|
|4.1.10 or higher | 4.1_X |
|4.2.X | 4.2_X |

### Abbreviations
| Abbr  | Meaning | 
|---|---|
| fqdn  | Fully qualified domain name, e.g. *hostname.domain.org*|


### SSL enabling
You can decide on SSL encryption. Both servers, the http server running Davrods and the iRODS server can be SSL-neabled:
  
  | Apache HTTP  | iRODS |  Configuration |
|---|---|---|
| SSL-enabled  | SSL-enabled | Follow all steps in this guide, use `irods_environment.json` 2.5 a) |
| SSL-enabled  | no SSL | Skip Section 1, create certificates for the HTTP server and use them in 2.4, use `irods_environment.json` 2.5 b) |
| no SSL (not advised) | SSL-enabled | Do not change port in 2.4, do not link to SSL certificates in 2.4, use `irods_environment.json` 2.5 a) |
| no SSL (not advised) | no SSL | Do not change port in 2.4, do not link to SSL certificates in 2.4, use `irods_environment.json` 2.5 b) |

## Prerequisites

- Running iRODS instance on Centos 7 ([Guide](https://github.com/EUDAT-Training/B2SAFE-B2STAGE-Training/blob/develop/ExampleTrainings/iRODS-SysAdmin-Training/iRODS-CentOS-install.md))
- *sudo* rights the machine iRODS runs on
- Network address mapping:
 - Map the ip-address to the hostname and the fully qualified domain name (last line)
 
 ```sh
 vi /etc/hosts
 127.0.0.1   localhost irods-centos 
 ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

 <ip-address>	irods-centos	<fqdn>
 ```
 
 - Configure the firewall
Ports 80 (HTTP) and 443 (HTTPS) need to be open for outside connections.

 ```sh
    sudo iptables -A INPUT -i lo -j ACCEPT
    sudo iptables -A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    sudo iptables -A INPUT -p tcp -m tcp --dport 22 -j ACCEPT
    # irods
    sudo iptables -A INPUT -p tcp -m tcp --dport 1247 -j ACCEPT
    sudo iptables -A INPUT -p tcp -m tcp --dport 1248 -j ACCEPT
    sudo iptables -A INPUT -p tcp -m tcp --dport 20000:20199 -j ACCEPT
    sudo iptables -A INPUT -p icmp -j ACCEPT
    # http
    sudo iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
    # https
    sudo iptables -A INPUT -p tcp -m tcp --dport 443 -j ACCEPT
 ```
 - Check the firewall and save it
 
 ```sh
   sudo iptables -L
   sudo service iptables save
 ```
 - Make sure there is no `DROP	all  -- anywhere	anywhere` somewhere in the middle. This might happen when you extend firewalls. 
  To delete the rule you can use:
  ```sh
  iptables -L INPUT --line-numbers
  sudo iptables -D INPUT <line>
  ```
 - Port 443 is only needed if you want to use HTTPS and SSL encryption for the HTTP server (Davrods).

## 1. Enable iRODS with SSL 
### (Skip if you do not want to use SSL encryption for your iRODS server)
In this section we will follow the [iRODS documentation - Section Server SSL Setup](https://docs.irods.org/4.1.10/manual/authentication/).

1. **Generate the SSL key and certificate** on the server that runs iRODS
 ```sh
 sudo su - irods
 mkdir /etc/irods/ssl
 cd /etc/irods/ssl
 openssl genrsa -out irods.key 2048
 chmod 600 irods.key
 openssl req -new -x509 -key irods.key -out irods.crt -days 365
 ```
 
 You are asked to provide some details.
 
 ```sh
 Country Name (2 letter code) [XX]:XX
State or Province Name (full name) []:<your state>
Locality Name (eg, city) [Default City]:<your city>
Organization Name (eg, company) [Default Company Ltd]:<company>
Organizational Unit Name (eg, section) []:<group>
Common Name (eg, your name or your server's hostname) []:<ip address or fqdn>
Email Address []:<email>
 ```
 The common name that you set here will also be used by all user clients and Davrods to address the iRODS server. It should correspond to the fqdn or the hostname you set in the */etc/hosts* file (see above).
 
 ```sh
 openssl dhparam -2 -out dhparams.pem 2048
 ```
 
2. **Adjust the /etc/irods/core.re** with
 
 ```sh
 acPreConnect(*OUT) { *OUT="CS_NEG_REQUIRE"; }
 ```
3. **Adjust the environment-json for the irods service account.**

 You need to set the server certificate (*irods.crt*) and its corresponding key (*irods.key*) and the certfificate from the "Certificate Authority" (here we use again *irods.crt* (usually you would have a *chain.pem*), if you use a different authority make sure all machines that run clients have this file installed). We also need to set the file defining how keys are exchanged (*dhparams.pem*). Finally we need to tell iRODS that we are using ssll verification by certificate.
 
 The irods environment file for the unix service account can be in several locations, please check what is applicable to you:
 ```sh
 vi /var/lib/irods/.irods/irods_environment.json # default
 vi /home/irods/.irods/irods_environment.json # if irods service account has a home
 ```
 
 ``` sh
 "irods_client_server_policy": "CS_NEG_REQUIRE",
 "irods_ssl_certificate_chain_file": "/etc/irods/ssl/irods.crt",
 "irods_ssl_certificate_key_file": "/etc/irods/ssl/irods.key",
 "irods_ssl_dh_params_file": "/etc/irods/ssl/dhparams.pem",
 "irods_ssl_ca_certificate_file": "/etc/irods/ssl/irods.crt",
 "irods_ssl_verify_server": "cert"
 ```
 Make sure common name of the server in the certificate and the *irods_host* in the environment json file match.
 Then try as the user 'irods' whether you can login:
 
 ```sh
 iinit
 ils
 ```
 Turn back to your normal unix user account
 
 ```sh
 exit
 ```

4. **Enabling other user clients with SSL.**

 Clients on other servers need to have a copy of the chain of trust pem-file (here the *irods.crt*) file.
 All your iRODS users need to extend their *irods_environment.json* with
 
 ```sh
 "irods_client_server_negotiation": "request_server_negotiation",
 "irods_client_server_policy": "CS_NEG_REQUIRE",
 "irods_ssl_ca_certificate_file": "</path/to>/irods.crt",
 "irods_encryption_key_size": 32,
 "irods_encryption_salt_size": 8,
 "irods_encryption_num_hash_rounds": 16,
 "irods_encryption_algorithm": "AES-256-CBC"
 ```
 
## 2. Create certificates for HTTP server
If you are installing Davrods on a different server than the iRODS server (or you did not enable your iRODS verser with SSL) and you would like to use HTTPS instead of HTTP you will need to create certificates for the HTTP server.

- Create certificate and key
 ```sh
 cd /etc/ssl/certs
 sudo openssl req \
 -newkey rsa:2048 -nodes -keyout davrods.key \
 -x509 -days 365000 -out davrods.crt
 ```
 Later in the webdav configuration we need to provide these two files.
  
## 3. Install Davrods
1. **Installation requirements:**
 
 iRODS runtime 4.1.11 (only for iRODS 4.1.11, skip when using iRODS 4.2.X)

 ```sh
 export SERVERPATH='ftp.renci.org/pub/irods/releases'
 wget \
 ftp://$SERVERPATH/4.1.11/centos7/irods-runtime-4.1.11-centos7-x86_64.rpm
 sudo yum install irods-runtime-4.1.10-centos7-x86_64.rpm
 ```
 
 Note if you do not install Davrods on the same server that also runs the iRODS server you need one of the following packages to be installed (only for iRODS 4.1.X):
 - icommands
 - iRODS resource server
 - iRODS icat

2. **Install the Apache HTTP server**

 ```sh
 sudo yum install httpd
 sudo service httpd start
 ```
 Test whether the httpd service runs by opening a browser and typing in the fully qualified domain name of your server or its ip address. You should see the HTTP testing page. Make sure port 80 is open.
 
 **Trouble shooting: SELinux keeps the HTTP server to connect to port 1247**
 In case you use SELinux, you need to make sure that https may connect to 1247. Otherwise, you may run into an Internal Server Error. If an Internal Server Error occurs, please check the SELinux audit logs (e.g. in /var/log/audit/audit.log) for a line similar to this:

```sh
type=AVC msg=audit(1504009961.761:1109613): avc:  denied  { name_connect } for  pid=7658 comm="httpd" dest=1247 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket
```
 
3. **Download and install Davrods**
 
 ```sh
 export SERVERPATH='github.com/UtrechtUniversity/davrods/releases'
 wget https://$SERVERPATH/download/4.1_X/davrods-4.1_X.el7.centos.x86_64.rpm
 sudo yum install davrods-4.1_X.el7.centos.x86_64.rpm
 ```
 For iRODS 4.2.X you need to download davrids-4.2_X.
 
 Since our Davrods will work with SSL encryption you will also need to install the corresponding SSL package for the Apache HTTP server (skip if you do not use encryption):
 
 ```sh
 sudo yum install mod_ssl
 ```

4. **Edit the /etc/httpd/conf.d/davrods-vhost.conf**
 Remove the first comment of each line starting at line 8.
 
 - Since we would like to use HTTPS instead of HTTP we need to tell the HTTP server to use port 443:
  ```sh
  <VirtualHost *:80> # for normal http
  ```
  needs to become 
  
  ```sh
  <VirtualHost *:443> # for https, when you enabled iRODS with SSL
  ```
 - We need to set the server name for the HTTP server in the next line
  
  ```sh
  ServerName <FQDN or IP address of the HTTP server>
  ```
 - Next we need to tell the httpd server where to find the certificates for the SSL encryption.
  ```sh
  SSLEngine on # only when using SSL
  SSLCertificateFile "/etc/irods/ssl/irods.crt"
  SSLCertificateKeyFile "/etc/irods/ssl/irods.key"
  ```
  **NOTE:** We are reusing the certificates we generated for iRODS since iRODS and the HTTP server are running on the same machine. If you deploy Davrods on a separate machine, you need to provide the certificates from Step 2 (matching the fqdn or IP address of that machine and not the iRODS server).
   
 - If you used a different zone name than the standard zone name you need to change
  ```sh
          #DavRodsZone tempZone
  ```
  to
  
  ```sh
          DavRodsZone <yourZone>
  ```
 - Uncomment the line
  ```sh
  #DavRodsExposedRoot User
  ```
  
 - If your HTTP server is running on a different machine than the iRODS server you will also need to set
  ```sh
          DavrodsServer localhost 1247
  ```
  to
  ```
      DavRodsServer <fqdn or ip of iRODS server> 1247
  ```
  
5. **Edit the /etc/httpd/irods/irods_environment.json**
 - Replace the "irods\_host" with the server name defined in the SSL step
 - Replace the "irods\_zone\_name" with your zone name
 - "irods\_client\_server\_policy" should be "CS\_NEG\_REQUIRE" (only for SSL, otherwise )
 - irods\_ssl\_verify\_server" should be "cert" (only for SSL)
 - "irods\_client\_server\_negotiation": "request\_server\_negotiation" (only for SSL)
 - The irods user you specify here is a proxy user that will act on behalf of your real users. Hence this user needs to have 'rodsadmin' rights.
 
 a) Example iRODS environment file for an SSL-enabled iRODS server:
 
 ```sh
 {
    "irods_host": "<FQDN or IP address>",
    "irods_port": 1247,
    "irods_default_resource": "demoResc",
    "irods_home": "/homeZone/home/rods",
    "irods_cwd": "/homeZone/home/rods",
    "irods_user_name": "rods",
    "irods_zone_name": "homeZone",
    "irods_client_server_negotiation": "request_server_negotiation",
    "irods_client_server_policy": "CS_NEG_REQUIRE",
    "irods_encryption_key_size": 32,
    "irods_encryption_salt_size": 8,
    "irods_encryption_num_hash_rounds": 16,
    "irods_encryption_algorithm": "AES-256-CBC",
    "irods_default_hash_scheme": "SHA256",
    "irods_match_hash_policy": "compatible",
    "irods_server_control_plane_port": 1248,
    "irods_server_control_plane_key": "TEMPORARY__32byte_ctrl_plane_key",
    "irods_server_control_plane_encryption_num_hash_rounds": 16,
    "irods_server_control_plane_encryption_algorithm": "AES-256-CBC",
    "irods_maximum_size_for_single_buffer_in_megabytes": 32,
    "irods_default_number_of_transfer_threads": 4,
    "irods_transfer_buffer_size_for_parallel_transfer_in_megabytes": 4,
    "irods_ssl_ca_certificate_file": "/etc/irods/ssl/irods.crt",
    "irods_ssl_verify_server": "cert"
 }
 ```
 
 b) Example iRODS environment file for an iRODS server not enabled with SSL:
 
 ```sh
 {
    "irods_host": "<FQDN or IP address>",
    "irods_port": 1247,
    "irods_default_resource": "",
    "irods_home": "/homeZone/home/rods",
    "irods_cwd": "/homeZone/home/rods",
    "irods_user_name": "rods",
    "irods_zone_name": "homeZone",
    "irods_client_server_policy": "CS_NEG_DONT_CARE",
    "irods_encryption_key_size": 32,
    "irods_encryption_salt_size": 8,
    "irods_encryption_num_hash_rounds": 16,
    "irods_encryption_algorithm": "AES-256-CBC",
    "irods_default_hash_scheme": "SHA256",
    "irods_match_hash_policy": "compatible",
    "irods_server_control_plane_port": 1248,
    "irods_server_control_plane_key": "TEMPORARY__32byte_ctrl_plane_key",
    "irods_server_control_plane_encryption_num_hash_rounds": 16,
    "irods_server_control_plane_encryption_algorithm": "AES-256-CBC",
    "irods_maximum_size_for_single_buffer_in_megabytes": 32,
    "irods_default_number_of_transfer_threads": 4,
    "irods_transfer_buffer_size_for_parallel_transfer_in_megabytes": 4,
    "irods_ssl_verify_server": "hostname"
 }
 ```
 
 Now restart the Apache HTTP server:
 
 ```sh
 sudo service httpd restart
 ```
6. **Install selinux**

 ```sh
 sudo setsebool -P httpd_can_network_connect true
 sudo chcon -t lib_t /var/lib/irods/plugins/network/lib*.so
 ```

7. **Test Davrods.** 
 Try to connect to the iRODS server with a webdav client like cyberduck or try to mount it from a different unix machine.
 You can mount filesystems supporting webDav with:
 
 ```sh
 sudo apt-get install davfs2 #ubuntu
 sudo yum install davfs2 #centos
 ```
 
 And then mount your iRODS logical namespace with:
 
 - FQDN: common name of the server running Apache HTTP (in case u you enbled it with SSL) or the fully qualified domain name of the server running Apache HTTP (for HTTP only)
 
 ```sh
 sudo mkdir /mnt/mytest/webdav/foo
 mount -t davfs http://<FQDN>/ /mnt/mytest/webdav/foo # http-enabled Davrods
 mount -t davfs https://<FQDN>/ /mnt/mytest/webdav/foo # https-enabled Davrods
 ```
 
   
## Configurations and remarks

1. **iRODS file system views.** 
 Davrods provides different options for views on the filesystem. By default a user who connects to iRODS via Davrods will only see his own *home* collection and will not be able to see higher collections.
 To enable other views you can choose between the options 'Zone', 'Home' and 'User' for the parameter *DavRodsExposedRoot*. Or you can define the highest collection a user sees yourself.
 Make sure that those collections ans subcollections till the user's home collection were granted 'read' rights to 'public' in iRODS, e.g.:
 
 ```sh
 ichmod read public /<yourZone>
 ichmod read public /<yourZone>/home
 ```
 grants read access to the root collection in iRODS and users can now navigate from there (test with a rodsuser account using the *icommands*). 
 
 Now we can set 
 
 ```sh
 DavRodsExposedRoot Zone
 ```
 in the *davrods-vhost.conf* and restart the Apache HTTP server
 
 ```sh
 sudo service httpd restart
 ```

2. **Remark**
 Every time you change the user data base you need to do a restart of the httpd server to fetch the changes:
 
 ```sh
 sudo service httpd restart
 ```
3. **Remark**
 After a reboot you might have to restart the postgresql, Apache HTTP server and the iRODS server manually if you did not configure your servers accordingly.
 
4. **Remark**
 In general when working with federated iRODS instances, your users would need read access to the root (*/*), zone (*/<myZone>* and other zones) and home (*/<myZone>/home*) collections to list all possible federated zones they might have access to. This does not work with Davrods. When changing the view to 
 
 ```sh
 DavRodsExposedRoot  /
 ```
  users willl just see an empty folder. Issue [#7](https://github.com/UtrechtUniversity/davrods/issues/7).
  
  Open question: Can Davrods support federations at all? There needs to be a mapping from *user* to *user#<myZone>* for the federated zone. How is that prpagated in Davrods?
  
5. **Remark**
  You can install Davrods on a different server than the iRODS server. 
  
  You simply need to adjust the davrods-vhost.conf:
  - ServerName (server where Apache HTTP runs on)
  - DavRodsServer (server where iRODS runs)
  - In case you want to enable the HTTP server with certificates you will have to create new ones matching the common name to the HTTP server's name.

6. **SSL enabling** You can also mix the SSL eabling of the two servers
  
  | Apache HTTP  | iRODS |  Configuration |
|---|---|---|
| SSL-enabled  | SSL-enabled | Follow all steps in this guide, use `irods_environment.json` 2.5 a) |
| SSL-enabled  | no SSL | Skip Section 1, create certificates for the HTTP server and use them in 2.4, use `irods_environment.json` 2.5 b) |
| no SSL (not advised) | SSL-enabled | Do not change port in 2.4, do not link to SSL certificates in 2.4, use `irods_environment.json` 2.5 a) |
| no SSL (not advised) | no SSL | Do not change port in 2.4, do not link to SSL certificates in 2.4, use `irods_environment.json` 2.5 b) |

7. **Matching hostnames**
 When working with certificates for your servers, note that you will have to address the servers the same way that you specified in the certificate. It is safe to use the fully qualified domain name or the IP address. If you insist on local names (for testing), set the /etc/host mapping accordingly.

