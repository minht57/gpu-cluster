# Moodle installation
## Install SLAM stack
```bash
sudo apt update && sudo apt upgrade
sudo apt install apache2
sudo apt install mysql-server mysql-client
sudo apt install mariadb-server php-mysql
#sudo apt install php libapache2-mod-php php-mysql
#sudo apt install php-curl php-json php-cgi

sudo apt install graphviz aspell ghostscript clamav php8.1-fpm php8.1-cli php8.1-pspell php8.1-curl php8.1-gd php8.1-intl php8.1-mysql php8.1-xml php8.1-xmlrpc php8.1-ldap php8.1-zip php8.1-soap php8.1-mbstring
```

## Moodle
```bash
sudo apt update && sudo apt upgrade
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update

sudo apt install php8.1 libapache2-mod-php8.1
sudo a2enmod php8.1

cd /opt/
sudo git clone https://github.com/moodle/moodle.git

sudo git branch --track MOODLE_402_STABLE origin/MOODLE_402_STABLE
sudo git checkout MOODLE_402_STABLE
sudo cp -R /opt/moodle /var/www/html/
sudo chmod -R 0777 /var/www/html/moodle
sudo mkdir /var/moodledata
sudo chown -R www-data /var/moodledata
sudo chmod -R 0777 /var/moodledata
```

## Setup database for moodle
```bash
sudo mysql -u root -p
```
- Create database
```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'moodle-user'@'localhost' IDENTIFIED BY 'Cluster_p@ssw0rd';

GRANT SELECT,INSERT,UPDATE,DELETE,CREATE,CREATE TEMPORARY TABLES,DROP,INDEX,ALTER ON moodle.* TO 'moodle-user'@'localhost';

exit
```

## Config apache2
- Edit file `sudo vi /etc/apache2/sites-available/000-default.conf`
```conf
Listen 8888
<VirtualHost *:8888>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html/

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
```

- Restart `apache2`
```bash
sudo systemctl status apache2
```

- Check IP and port config in moodle config
```bash
cd /var/www/html/moodle
sudo vi config.php
```

- Please ensure that `$CFG->wwwroot ` has the `IP:port`
```php
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'localhost';
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodle-user';
$CFG->dbpass    = 'Cluster_p@ssw0rd';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_unicode_ci',
);

$CFG->wwwroot   = 'http://192.168.33.5/moodle';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 0777;

require_once(__DIR__ . '/lib/setup.php');

// There is no php closing tag in this file,
// it is intentional because it prevents trailing whitespace problems!
```

- Change config in `sudo vi /etc/php/8.1/apache2/php.ini`
```ini
max_input_vars=5000
```

- Restart `apache2`
```bash
sudo systemctl restart apache2
```

- Open website and config: [http://192.168.33.5:8888/moodle](http://192.168.33.5:8888/moodle)
  - Follow [this tutorial](https://www.linode.com/docs/guides/how-to-install-moodle-on-ubuntu-22-04/#configuring-moodle-using-the-web-interface).


# Install OAuth Plugin
- Reference [this tutorial](https://github.com/jupyterhub/oauthenticator/blob/master/docs/source/getting-started.rst#moodle-setup). 
- Install the Moodle OAuth plugin:
- Install by uploading the zip file at *Site administration* > *Plugins* > *Install plugins*
- Head over to *Site administration* > *Server* > *OAuth provider settings* > *Add new client*
  - `client_id`: JupyterHub
  - callback URL to http://192.168.33.5/hub/oauth_callback

- Copy `client_token`:
  - Paste to config.yaml file