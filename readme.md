# Full Gucamole Install (version 1.2.0 with fixes for Debian 10 install)

### Prerequisites installation

#### Add the repo for Debian stretch 
Some of the supporting binaries for guacamole are not availabe in Debian 10 but the legacy components from Debian Stretch work fine, so we are going to add the stretch repo so guacamole will build properly.<br>
``` bash
echo deb http://deb.debian.org/debian/ stretch main >> /etc/apt/sources.list
echo deb-src http://deb.debian.org/debian/ stretch main >> /etc/apt/sources.list
apt-get update
```

####  Install Prerequisite Software<BR>
Copy and execute the apt-get command below to install the necessary supporting software required to build guacamole.<br>
``` bash
apt-get install build-essential autoconf libtool m4 libpng-dev libjpeg-dev libcairo-dev libossp-uuid-dev libtelnet-dev libpango1.0-dev libfreerdp-dev libssh2-1-dev libwebp-dev libvncserver-dev libpulse-dev libjpeg-dev libcairo-dev libvorbis-dev freerdp2-dev libfreerdp-dev libcairo-dev libpango1.0-dev gcc make uuid curl uuid-dev libossp-uuid-dev libssh2-1-dev libpulse-dev libssl-dev libvorbis-dev ffmpeg maven libvncserver-dev libavutil-dev libavcodec-dev libswscale-dev  software-properties-common libvorbis-dev libwebp-dev gcc make tomcat8 tomcat8-admin tomcat8-common tomcat8-user libpng-dev libjpeg-dev jaxb/stable libmaven-jaxb2-plugin-java/stable tomcat8 tomcat8-admin tomcat8-common tomcat8-user libpng-dev libjpeg-dev jaxb/stable libmaven-jaxb2-plugin-java/stable wget git -y 
  ```

#### Download and Install Guacamole Server Components<br>
Both the server and the client components must be installed on each guacamole server, so install the server by executing the following commands: <br>
<br>
``` bash
cd ~/
curl -O -J -L "http://apache.org/dyn/closer.cgi?action=download&filename=guacamole/1.2.0/source/guacamole-server-1.2.0.tar.gz" --output guacamole-server-1.2.0.tar.gz
tar -zxvf guacamole-server-1.2.0.tar.gz
cd guacamole-server-1.2.0
env CFLAGS="-g -O2 -Wno-stringop-truncation -Wno-deprecated-declarations" ./configure --with-init-dir=/etc/init.d
make
(IF a pre-existing install is being upgraded then 'mv /usr/local/sbin/guacd /usr/local/sbin/guad.bak' prior to running make install)
make install
/usr/sbin/ldconfig
```


#### Download and Install Guacamole Client from Apache Guacamole Site ####
``` bash
cd ~/
curl -O -J -L "http://apache.org/dyn/closer.cgi?action=download&filename=guacamole/1.2.0/source/guacamole-client-1.2.0.tar.gz" --output guacamole-client-1.2.0.tar.gz
tar -zxvf guacamole-client-1.2.0.tar.gz
apt-get install openjdk-11-jdk/stable -y
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
cd ~/guacamole-client-1.2.0
sed -i 's/\/detectOfflineLinks\>/\/detectOfflineLinks\> \n                    \<source\>8\<\/source/g' ~/guacamole-client-1.2.0/guacamole-common/pom.xml
sed -i 's/\/detectOfflineLinks\>/\/detectOfflineLinks\> \n                    \<source\>8\<\/source/g' ~/guacamole-client-1.2.0/guacamole-ext/pom.xml
mvn package
mkdir /etc/guacamole
cp guacamole/target/guacamole*.war /etc/guacamole/guacamole.war
ln -s /etc/guacamole/guacamole.war /var/lib/tomcat8/webapps
```

##### Restart Guacamole Services
``` bash
systemctl enable guacd
systemctl start guacd
systemctl enable tomcat8
```

###  Installing MySQL to support Guacamole <br>
``` bash
apt-get install default-mysql-server libmysql-java -y
```

####  Adding Guacamole MySQL Backend Extension <br>
``` bash  
cd ~/
curl -O -J -L "http://apache.org/dyn/closer.cgi?action=download&filename=guacamole/1.2.0/binary/guacamole-auth-jdbc-1.2.0.tar.gz" --output guacamole-auth-jdbc-1.2.0.tar.gz
tar -zxvf guacamole-auth-jdbc-1.2.0.tar.gz
mkdir /etc/guacamole/extensions
mkdir /etc/guacamole/lib

In the next cp command make sure you copy the jdbc-mysql.jar as shown:
cp guacamole-auth-jdbc-1.2.0/mysql/guacamole-auth-jdbc-mysql-1.2.0.jar /etc/guacamole/extensions/
```

##### Create the guacamole.properties File <br>
This next set of command are used to create the guacamole.properties file and define the SQL Connection information.  SQL will be used to store data for guacamole. <br>
<br>
``` bash
nano /etc/guacamole/guacamole.properties


mysql-hostname: localhost
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: guacamole_user
mysql-password: P@ssw0rd

(ctrl-x then y) - then press enter to save the file
```
Note:  Set the password for mysql-password to a password of your choice. 

##### Start MySQL and set it to Start on Next Boot<br>
``` bash
systemctl start mysql
systemctl enable mysql
```
Note:  Ignore the "Too many levels of symbolic links message as it is a false positive" <br>
<br>
####  Create MySQL Database and Add User(s)
``` bash
mysql -u root
create database guacamole_db;
create user 'guacamole_user'@'localhost' identified by 'P@ssw0rd';
grant select,insert,update,delete on guacamole_db.* to 'guacamole_user'@'localhost';
flush privileges;
quit
```
<br>
Note:  Change the password after identified by to the password you set in the guacamole.properties file step above.

####  Populate MySQL Tables
``` bash
cd ~/
cat guacamole-auth-jdbc-1.2.0.tar.gz/mysql/schema/*.sql | mysql -u root -D guacamole_db
ln -s /usr/share/java/mysql-connector-java.jar /etc/guacamole/lib/
systemctl restart guacd
systemctl restart tomcat8.service
echo /usr/local/lib >> /etc/ld.so.conf
/usr/sbin/ldconfig
```

### Install Guacamole LDAP Support
``` bash
curl -O -J -L "http://apache.org/dyn/closer.cgi?action=download&filename=guacamole/1.2.0/binary/guacamole-auth-ldap-1.2.0.tar.gz" --output guacamole-auth-ldap-1.2.0.tar.gz
tar -zxvf guacamole-auth-ldap-1.2.0.tar.gz
cp guacamole-auth-ldap-1.2.0/*.jar /etc/guacamole/extensions
ln -s /etc/guacamole /usr/share/tomcat8/.guacamole
systemctl restart guacd
```

#### Configure LDAP Connection Information
Add the following to the /etc/guacamole/guacamole.properties file using nano: <br>
``` bash
nano /etc/guacamole/guacamole.properties

#Paste Everything Below this line at the end of the file

#Auth provider class 
auth-provider=net.sourceforge.guacamole.net.auth.ldap.LDAPAuthenticationProvider
#LDAP properties
ldap-hostname:corpdc01.mydomain.com
ldap-port=636
ldap-encryption-method:ssl
ldap-user-base-dn:OU=Admins,OU=MyHQ,DC=mydomain,DC=com
ldap-username-attribute:sAMAccountName
ldap-search-bind-dn:CN=svcaccount04,OU=Admins,OU=MyHQ,DC=mydomain,DC=com
ldap-search-bind-password:P@ssw0rd
ldap-group-base-dn:OU=Groups,OU=MyHQ,DC=mydomain,DC=com
ldap-config-base-dn:OU=Admins,DC=MyHQ,DC=mydomain,DC=com

(ctrl-x then y) - then press enter to save the file
```

### Connect to the Guacamole Web Service to Confirm Initial Installation<br>
Open a web browser and type http://IPOFGUACAMOLESYSTEM:8080/guacamole <br>

Default Username:  guacadmin <br>
Default Password:  guacadmin <br>
<br>
Note: Make sure you change the default password after login.<br>

###  Install Custom Root CA if using Self-Signed Certificate through NGINX
keytool -importcert -file /etc/certs/cacert.pem <br>
Set password to P@ssword or something <br>
type yes - to trust the keystore when prompted. <br>
