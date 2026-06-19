## cacti-weathermap

Plugin for [cacti](cacti.net)

This is a port for use with tools, utilities, & libraries I see fit
- This currently includes: Panzoom

# There is no expectation that anyone cares

# No warranty expressed or otherwise to the functionality of the code presented here

Last merge of Cacti/plugin_weathermap develop into cacti-1.2.x_weathermap-develop_rigrace: **2025-12-07**

 - functionality not yet verified 

- [x] Shoe-horn of [Panzoom](https://www.jqueryscript.net/zoom/jQuery-Plugin-For-Panning-Zooming-Any-Elements-panzoom.html#google_vignette) project into map edit & display flows is functional

- [ ] Cleanup of class variables, and thinking about how to OO the code

- [ ] Convert the use of OS files as the method of storage for map, node, & link data/configurations to tables

-------------------------

## my installation steps:


Install & configure Cacti 1.2.30 on ubuntu 24.04
referenced:
- https://docs.cacti.net/Installing-Under-Ubuntu-Debian.md
- https://github.com/Cacti/documentation/blob/develop/Installing-Under-Ubuntu-Debian.md
- https://zacs-tech.com/how-to-install-cacti-network-monitoring-tool-on-ubuntu-24-04-22-04/
- https://idroot.us/install-cacti-ubuntu-24-04/

Set /etc/hostname & /etc/hosts
-------------------
Update os & install base required applications
preperation: updates & upgrades (per your discression)

```bash
#Setup basic utilities
sudo apt install libfuse2t64 lldpad vim gzip parted git tree curl cifs-utils openssh-client openssh-server acl net-tools synaptic gparted synaptic gnome-system-monitor smbclient

#setup secure ssh connections now

#Using sftp copy files/scripts over 
remote

Set /etc/hostname & /etc/hosts
-------------------
#Update os & install base required applications
sudo apt update
sudo apt upgrade

#start installing Cacti support 
sudo apt install -y apache2 rrdtool mariadb-server snmp snmpd php php-mysql php-curl php-intl php-mcrypt php-rrd php-snmp php-xml php-mbstring php-json php-gd php-gmp php-zip php-ldap php-intl libapache2-mod-php php-xdebug php-cli 
```
Add/enable aditional apache modules
```bash
sudo a2enmod rewrite 
```
#Clone cacti base application to local folder & push it into /var/www/html/(cacti)


mkdir -p ~/Documents/cacti
CHANGE TO 1.2.x branch
```bash
mkdir -p ~/Documents/cacti

git clone -b 1.2.x https://github.com/Cacti/cacti.git ~/Documents/cacti
#CHANGE TO 1.2.x branch
```
Start configure cacti:
```bash
cp ~/Documents/cacti/include/config.php.dist ~/Documents/cacti/include/config.php
vim ~/Documents/cacti/include/config.php
  - set mysql connection credentials

sudo cp -R ~/Documents/cacti /var/www/html
```

```bash
#Set file permissions for apache:
sudo chown -R www-data:www-data /var/www/html/cacti
sudo chmod -R 770 /var/www/html/cacti
```
-----------------
Using port 80/443 for /var/www/html/cacti
Use another port such as 81 as SSL for workspace or alternate /var/www/html vhosts
set listening ports - for example:
http w/ port 81
```bash
sudo vim /etc/apache2/ports.conf
Listen 81
```
https w/ 81 (for alternate vhost, I usually save 443 for /var/www/html vhosts)
```VIM
<IfModule ssl_module>
	Listen 81
</IfModule>
```
```bash
sudo vim /etc/apache2/sites-available/cactiV.conf:
```
```vim
<VirtualHost *:80>
   ServerName cacti
   #ServerAlias www.cacti
   ServerAdmin no@email.com
   DocumentRoot /var/www/html
   DirectoryIndex index.php
   <Directory /var/www/html/cacti>
     Options FollowSymLinks
     AllowOverride None
	 Require all granted
	 #Redirect 403 /cacti/index.php
   </Directory>
   
       LogLevel warn
       ErrorLog ${APACHE_LOG_DIR}/cacti_error.log
       CustomLog ${APACHE_LOG_DIR}/cacti_access.log combined
   
   
   
</VirtualHost>
[exit saving changes]
```

remove default vhosts & add cactivhost
```bash
sudo a2dissite 000-default.conf
sudo a2ensite cactiV.conf
```

----------------
```bash
basic mysql setup
sudo mysql -u root (actually using mariadb)
```
once in mariadb
```SQL
CREATE DATABASE cacti DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci ;
CREATE DATABASE test;
[create view to show mysql user status]
create view test.musers_view as SELECT host, User, Select_priv, Insert_priv, Update_priv, Delete_priv, Execute_priv, Show_db_priv, ssl_type, grant_priv, ssl_cipher, x509_issuer, x509_subject FROM mysql.user;
select * from test.musers_view;

[create cactiuser for accessing database from cacti application, phpmyadmin if installed]
create user 'cactiuser'@'localhost' IDENTIFIED BY 'some_password';
alter user 'cactiuser'@'localhost' IDENTIFIED BY 'some_password'; [example of setting password after user is created]
GRANT ALL PRIVILEGES ON cacti.* TO 'cactiuser'@'localhost'; [SUID will not suffice as cacti creates database objects during at least the install]
GRANT SELECT ON mysql.time_zone_name TO 'cactiuser'@'localhost'; [to be filled below]

[create admin for accessing database from cacti application, phpmyadmin if installed]
create user 'admin'@'localhost' identified by 'some_password';
grant all privileges on *.* to 'admin'@'localhost';
grant grant option on *.* to 'admin'@'localhost';

```

run cacti database script to create cacti's database objects
```BASH
mysql -u cactiuser -p cacti < ~/Documents/cacti/cacti.sql
```
```mysql
[set cacti's admin password]
use cacti;
update `user_auth` set password = md5('123456') where username = 'admin';
[cacti will prompt to change password]

FLUSH PRIVILEGES;
quit
```

Load mysql timezones
```BASH
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u allowed_user -p mysql
```
------------------
Install phpmyadmin if not already
```BASH
sudo apt install phpmyadmin

[using apache]
[answer no to db-config prompt]


./ApacheRestart.sh
./MySQLRestart.sh
```
Run cacti initialization  by starting http://localhost/cacti

- hit page localhost/cacti in browser
- follow prompts
  - Some mysql settings can't be set when the service is up (read on reload)
sudo vim /etc/mysql/my.cnf # and add them there
[mysql] #section for mysql proper

[mariadb] section for the one used here

------

```bash
sudo chown -R www-data:www-data /usr/share/cacti/site/resource/snmp_queries/
sudo chown -R www-data:www-data /usr/share/cacti/site/resource/script_server/
sudo chown -R www-data:www-data /usr/share/cacti/site/resource/script_queries/
sudo chown -R www-data:www-data /usr/share/cacti/site/scripts/
```

-----------------
Setup poller actuation
1. quick & dirty cron method
	```bash	crontab -e ```
	#Add 5 minute interval call to poller
	```vim 
	*/5 * * * * www-data php /var/www/html/cacti/poller.php &>/dev/null
	```
2. Slightly more involved service base method
	```bash
	sudo vim /var/www/html/cacti/service/cactid.service
	```
	# Set 'User' & 'Group' to www-data
        
    create cacti environment file
    ```bash
	sudo mkdir -p /etc/sysconfig
	sudo touch /etc/sysconfig/cactid
	```bash
	sudo cp -p /var/www/html/cacti/service/cactid.service /etc/systemd/system
	
	sudo chown root:root /etc/systemd/system/cactid.service
	

	sudo sudo systemctl daemon-reload
	sudo systemctl enable cactid
	sudo systemctl restart cactid
    sudo systemctl status cactid
	```
----------------------------------

ADDING WEATHERMAP
#get code from source from:
- cloned git source
  - change to consumable branch using git
- downloaded zip file
  - extract to folder
- copy weathermap folder into .../vhost_folder/plugins folder
```bash
cp .../path_to_source/weathermap .../var/www/html/cacti/plugins/
``` 
- set vhost folder permissions
  - for var/www/html/cacti I use:
    ```bash
    sudo chown -R www-data:www-data /var/www/html
    sudo chmod -R 0770 /var/www/html
    ```
- enable weathermap plugin 
  http://localhost/cacti
  browse to console - configuration - plugins
  install the plugin
  enable the plugin
Input Validation Whitelist Protection

<h1>This is a claenup note</h1>
Cacti Data Input methods that call a script can be exploited in ways that a non-administrator can perform damage to either files owned by the poller account, and in cases where someone runs the Cacti poller as root, can compromise the operating system allowing attackers to exploit your infrastructure.

Therefore, several versions ago, Cacti was enhanced to provide Whitelist capabilities on the these types of Data Input Methods. Though this does secure Cacti more thoroughly, it does increase the amount of work required by the Cacti administrator to import and manage Templates and Packages.

The way that the Whitelisting works is that when you first import a Data Input Method, or you re-import a Data Input Method, and the script and or arguments change in any way, the Data Input Method, and all the corresponding Data Sources will be immediatly disabled until the administrator validates that the Data Input Method is valid.

To make identifying Data Input Methods in this state, we have provided a validation script in Cacti's CLI directory that can be run with the following options:

    php -q input_whitelist.php --audit - This script option will search for any Data Input Methods that are currently banned and provide details as to why.
    php -q input_whitelist.php --update - This script option un-ban the Data Input Methods that are currently banned.
    php -q input_whitelist.php --push - This script option will re-enable any disabled Data Sources.

It is strongly suggested that you update your config.php to enable this feature by uncommenting the $input_whitelist variable and then running the three CLI script options above after the web based install has completed.

Check the Checkbox below to acknowledge that you have read and understand this security concern



