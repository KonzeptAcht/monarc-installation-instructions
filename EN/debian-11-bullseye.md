
# Installation Instructions Debian 11 (Codename: Bullseye)

## Requirements

- Debian 11 server installation
- internet access
- Command line access (Putty/SSH/direct access)


## Installation

:warning: In the following instructions, the placeholders "monarc.domain.de" and "password" are used. These have to be changed for the values selected by you.

### Preparation

First we prepare the installed Ubuntu Server instance. First, we bring the system up to date before we start installing firewalld as the firewall management tool and vim as the editor. These are the easiest to use in our opinion, but you can of course use other tools like ufw or nano. Please note in the following, the appropriate lines for you to adapt.

```bash
sudo apt update && apt upgrade
sudo apt install firewalld vim
```

After successful installation we set up firewalld for autostart and start the service.

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

In the next step, we set up the necessary firewall policies. In our case, we access the Ubuntu server via SSH. Therefore, we will also open the ports necessary for ssh.

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Finally, we install some tools and dependencies, for the later MONARC installation.

```bash
sudo apt install zip unzip git gettext curl gsfonts
```

### Installation MariaDB database server

We start with the installation of the database server and the client software.

```bash
sudo apt install mariadb-server mariadb-client
```

Anschließend richten wir hier ebenfalls den Autostart ein und starten den Dienste initial.

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

After successful start of the service we can start configuration.

```bash
sudo mysql_secure_installation
```

The following information should be provided here:

```bash
Enter current password for root (enter for none): [ENTER]
...
Set root password? [Y/n] Y
New password:
Re-enter new password:
...
Remove anonymous users? [Y/n] Y
...
Disallow root login remotely? [Y/n] Y
...
Remove test database and access to it? [Y/n] Y
```
After successful installation, we prepare the database for MONARC installation. To do this, we first log in to the database service with the "root" user's credentials and then create the MONARC database user and databases.

```bash
mysql -u root -p
CREATE DATABASE monarc_cli DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE DATABASE monarc_common DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
CREATE USER 'monarc'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON monarc_cli.* to 'monarc'@'localhost';
GRANT ALL PRIVILEGES ON monarc_common.* to 'monarc'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

This completes the installation of the database service and we can proceed with the Apache web server.

### Installation Apache Webserver und PHP 7.4

First we install the required Apache and PHP 7.4 packages.

```bash
sudo apt install php7.4 php7.4-{curl,gd,mysql,pear,apcu,xml,mbstring,intl,imagick,zip,bcmath} libapache2-mod-php
```

In order to get access to the MONARC environment later, a "Virtual Host" must be set up at Apache. In the configuration provided here some parts are commented out. This version allows to get unencrypted access to the installed MONARC instance. However, it is highly recommended to secure the connection using SSL/TLS certificate. The configuration file is already prepared for this purpose.

To create a "Virtual Host" we change to the Apache configuration directory

```bash
cd /etc/apache2/sites-available/
```

Here we now create a new file and name it according to the domain under which the installation will be operating.

```bash
vim monarc.domain.de.conf
```

Here we insert the following configuration and save it.

```bash
<VirtualHost *:80>
 ServerName monarc.domain.de
# Redirect / https://monarc.domain.de/
#</VirtualHost>

#<VirtualHost *:443>
#ServerName monarc.domain.de
DocumentRoot "/var/lib/monarc/fo/public"

    <Directory /var/lib/monarc/fo/public>
        DirectoryIndex index.php
        AllowOverride All
        Require all granted
		Options -Indexes
    </Directory>

    <IfModule mod_headers.c>
       Header always set X-Content-Type-Options nosniff
       Header always set X-XSS-Protection "1; mode=block"
       Header always set X-Robots-Tag none
       Header always set X-Frame-Options SAMEORIGIN
    </IfModule>

# SSL configuration, you may want to take the easy route instead and use Lets Encrypt!
#SSLEngine on
#Include /etc/letsencrypt/options-ssl-apache.conf
#SSLProtocol -all + TLSv1.3 +TLSv1.2 +TLSv1.1
#SSLHonorCipherOrder on

# Encoded slashes need to be allowed
AllowEncodedSlashes             NoDecode

#SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
#SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
#SSLCertificateChainFile /etc/letsencrypt/live/monarc.domain.de/chain.pem
</VirtualHost>
```

Finally, we activate the required modules and install the missing PHP dependencies:

```bash
sudo apt-get install apache2
sudo a2dismod status
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2enmod headers
```

To activate our changes to the virtual host, we now need to create a symbolic link.

```bash
cd /etc/apache2/sites-enabled/
ln -s ../sites-available/monarc.domain.de.conf monarc.domain.de.conf
```

To check whether our configuration is error-free, we can now perform this check.

```bash
sudo apachectl -S
```

Our created file should be listed in the output. In addition, no errors should be reported.

With this we can set up the autostart for Apache and start the service.

```bash
sudo systemctl enable apache2
```

### PHP settings

Since MONARC can still have problems with the standard timeouts of PHP at the current time for larger risk management structures, we recommend to adjusted the following parameter in php.ini:

```bash
vim /etc/php/7.4/apache2/php.ini
```
```bash
...
max_execution_time = 0
...
max_input_time = 0
...
memory_limit = 2048M
...
```
In order to avoid problems with possible uplouds into the MONARC environment later on, we set the uploud limit higher in /etc/php/7.4/apache2/php.ini

```bash
sudo vim /etc/php/7.4/apache2/php.ini
```
```bash
upload_max_filesize = 200M
```

### Optional: Installation with secured SSL/TLS certificate

For a secured installation via SSL/TLS certificate we use "certbot" provided by LetsEncrypt. If the server cannot be resolved publicly via DNS, this step should be skipped. In this case you can use PKI or manually generate a certificate with Openssl and store it in the Virtual Host file and the client.

#### Let's Encrypt

```bash
sudo apt install certbot python-certbot-apache
```

After successful installation, and creation of the "Virtual Host" from the previous step, we can generate a certificate.

```bash
certbot certonly --apache -d monarc.domain.de
```

If the certificate has been generated successfully, the commented parts from the "Virtual Host" configuration can be switched on. Remember to adjust the following paths in the file.

```bash
SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
SSLCertificateChainFile /etc/letsencrypt/live/monarc.domain.de/chain.pem
```

#### Self signed certificate

The following command can be used to generate a SSL certificate.

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
```

Description:
```bash
  -newkey #A new key will be generated
  rsa:2048 # A Key with cryptcypher RSA an keylenght of 2048 Bit
  -nodes #Do not encrypt with 3DES-CBC
  -keyout key.pem #Output Path and name of Keyfile
  -x509 #Use X.509 Certificate Data Management
  -days 365 #Certificate is 365 days valid
  -out certificate.pem #Outpput Path and name of Certificate file
```

after this we can check the generated certificate

```bash
openssl x509 -text -noout -in certificate.pem
```

Then you can store the generated files in the virtual host file. Since there is no trust root, only the following two lines must be edited. In addition, the other commented configurations can be activated.

```bash
SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
```

### Installation Composer

Another dependency of this product is the composer. It is essential for the setup and later update of the product. Since the installation cannot be done via the official repositories, as version >= 2.1.0 is required, it is explained again here.

With the following command you can download the installation script.

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
```

Afterwards, the signature should be verified to make sure that the script is not corrupted.

```bash
HASH="$(wget -q -O - https://composer.github.io/installer.sig)"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```

If the hash values match, the following message should be output.
`Installer verified`

This prepares the installation and we can start the script as follows:
```bash
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

The following command can be executed to verify the version status. At least version >=2.1.0 should be installed.

```bash
composer -V
```

### MONARC Installation

We start by creating the installation directory for MONARC, downloading the installation files from Git, and making some folder adjustments.

```bash
mkdir -p /var/lib/monarc/fo
git clone https://github.com/monarc-project/MonarcAppFO.git /var/lib/monarc/fo
cd /var/lib/monarc/fo
mkdir -p data/cache
mkdir -p data/LazyServices/Proxy
chmod -R g+w data
```

With the following command we install all open dependencies.

```bash
composer install -o
```

Afterwards, some customizations and shortcuts have to be created for the framework used by MONARC.

```bash
mkdir -p module/Monarc
cd module/Monarc
ln -s ./../../vendor/monarc/core Core
ln -s ./../../vendor/monarc/frontoffice FrontOffice
mkdir node_modules
cd node_modules
git clone https://github.com/monarc-project/ng-client.git ng_client
git clone https://github.com/monarc-project/ng-anr.git ng_anr
cd ..
```

Now that we have all the files from MONARC, we can initialize the database with the required schema and corresponding information.

```bash
mysql -u monarc -p monarc_common < db-bootstrap/monarc_structure.sql
mysql -u monarc -p monarc_common < db-bootstrap/monarc_data.sql
```

In order for the MONARC instance to have access to the created database, a configuration file has to be created. For this we are provided with a template, which we copy and adapt accordingly.

```bash
sudo cp ./config/autoload/local.php.dist ./config/autoload/local.php
sudo vim config/autoload/local.php
```

```bash
return array(
  'doctrine' => array(
      'connection' => array(
          'orm_default' => array(
              'params' => array(
                  'host' => 'localhost',
                  'user' => 'monarc',
                  'password' => 'password',
                  'dbname' => 'monarc_common',
              ),
          ),
          'orm_cli' => array(
              'params' => array(
                  'host' => 'localhost',
                  'user' => 'monarc',
                  'password' => 'password',
                  'dbname' => 'monarc_cli',
              ),
          ),
      ),
  ),
```

This establishes the database connection.

:bulb: Since with the version 2.9.17 an extension module for the generation of statistics from created risk analyses was published, since then further adjustments are to be made to the local.php, if this function will be used. In this case, further information can be found at the following link: https://www.monarc.lu/documentation/stats-service/master/installation.html

Since MONARC requires "nodejs 14" and this version is currently not part of the Ubuntu repository, it must now be installed manually. After the installation all dependencies are fulfilled to install "grunt-cli" as well.

```bash
curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
sudo apt-get install nodejs
sudo npm install -g grunt-cli
```

Finally, we run the update script that makes the prepared MONARC instance operational.

```bash
./scripts/update-all.sh -c
```

To prevent permission problems, we set the appropriate permissions for the Apache user once again.

```bash
sudo chown -R www-data:www-data /var/lib/monarc/fo/public
```

MONARC is now ready for operation and can be accessed via the configured domain. To be able to log in to the web interface, only an administrator has to be created.

```bash
php ./vendor/robmorgan/phinx/bin/phinx seed:run -c ./module/Monarc/FrontOffice/migrations/phinx.php
```

The login data created are as follows:
```bash
User: admin@admin.localhost
Password: admin
```

## MONARC Update

If you want to update monarc, you should first make sure that your system is up to date.

```bash
cd /var/lib/monarc/fo/
sudo apt update && sudo apt upgrade
composer update
npm install -g npm
```

Then we can update the system with the prepared script

```bash
./scripts/update-all.sh -c
```

Finally, we delete the data present in the cache.

```bash
rm -rf data/cache/*
rm -rf data/DoctrineORMModule/Proxy/*
rm -rf data/LazyServices/Proxy/*
```

## Problems?

If you have any problems during the installation or if you have any questions, please feel free to use the official MONARC [discussion area](https://github.com/monarc-project/MonarcAppFO/discussions)
