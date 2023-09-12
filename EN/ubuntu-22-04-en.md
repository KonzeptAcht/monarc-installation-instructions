
# Installation Instructions Ubuntu 20.04

## Requirements

- Ubuntu 22.04 LTS server installation
- internet access
- Command line access (Putty/SSH/direct access)


## Installation

:warning: In the following instructions, the placeholders "monarc.domain.de" and "password" are used. These have to be changed for the values selected by you.

### Preparation

First we prepare the installed Ubuntu Server instance. We bring the system up to date before we start installing vim as the editor and configure ufw as firewall. These are the easiest to use in our opinion, but you can of course use other tools like ufw or nano. Please note in the following, the appropriate lines for you to adapt.

```bash
sudo apt update && apt upgrade
sudo apt install ufw vim
```

After successful installation we set up firewalld for autostart and start the service.

```bash
sudo systemctl enable ufw
sudo systemctl start ufw
```

In the next step, we set up the necessary firewall policies. In our case, we access the Ubuntu server via SSH. Therefore, we will also open the ports necessary for ssh.

```bash
sudo ufw allow http
sudo ufw allow https
sudo ufw allow ssh
sudo ufw enable
sudo ufw reload
```

Finally, we install some tools and dependencies, for the later MONARC installation.

```bash
sudo apt install zip unzip git gettext curl jq
```

### Installation MariaDB database server

We start with the installation of the database server and the client software.

```bash
sudo apt install mariadb-server mariadb-client
```

Afterwards, we also set up the autostart here and start the services initially.

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
OK, successfully used password, moving on...

Switch to unix_socket authentication [Y/n] Y
Enabled successfully!
Reloading privilege tables..
 ... Success!

Change the root password? [Y/n] Y
New password: 
Re-enter new password: 
Password updated successfully!
Reloading privilege tables..
 ... Success!

Remove anonymous users? [Y/n] Y
 ... Success!

Disallow root login remotely? [Y/n] Y
 ... Success!

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

Thanks for using MariaDB!
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
sudo apt install apache2 php php-{curl,gd,mysql,pear,apcu,xml,mbstring,intl,imagick,zip,bcmath} libapache2-mod-php
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

Since MONARC can still have problems with the standard timeouts of PHP at the current time for larger risk management structures, we recommend to adjusted the following parameter in php.ini. Adjust the "x" in the path name according to the PHP version you are using.

:warning: Attention
use these adjustments only in productive environments, if this is necessary.

```bash
vim /etc/php/8.x/apache2/php.ini
```
```bash
...
max_execution_time = 100
...
max_input_time = 223
...
memory_limit = 512M
...
upload_max_filesize = 200M
...
post_max_size=50M
```

### Optional: Installation with secured SSL/TLS certificate

For a secured installation via SSL/TLS certificate we use "certbot" provided by LetsEncrypt. If the server cannot be resolved publicly via DNS, this step should be skipped. In this case you can use PKI or manually generate a certificate with Openssl and store it in the Virtual Host file and the client.

#### Let's Encrypt

```bash
sudo apt install certbot python3-certbot-apache
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

### MONARC Installation

We start by creating the installation variables.

```bash
PATH_TO_MONARC='/var/lib/monarc/fo'
PATH_TO_MONARC_DATA='/var/lib/monarc/fo-data'
MONARC_VERSION=$(curl --silent -H 'Content-Type: application/json' https://api.github.com/repos/monarc-project/MonarcAppFO/releases/latest | jq  -r '.tag_name')
MONARCFO_RELEASE_URL="https://github.com/monarc-project/MonarcAppFO/releases/download/$MONARC_VERSION/MonarcAppFO-$MONARC_VERSION.tar.gz"
```

Afterwards, we create our folder structure and download the newest monarc version.

```bash
mkdir -p /var/lib/monarc/releases/
# Download release
curl -sL $MONARCFO_RELEASE_URL -o /var/lib/monarc/releases/`basename $MONARCFO_RELEASE_URL`
# Create release directory
mkdir /var/lib/monarc/releases/`basename $MONARCFO_RELEASE_URL | sed 's/.tar.gz//'`
# Unarchive release
tar -xzf /var/lib/monarc/releases/`basename $MONARCFO_RELEASE_URL` -C /var/lib/monarc/releases/`basename $MONARCFO_RELEASE_URL | sed 's/.tar.gz//'`
# Create release symlink
ln -s /var/lib/monarc/releases/`basename $MONARCFO_RELEASE_URL | sed 's/.tar.gz//'` $PATH_TO_MONARC
# Create data and caches directories
mkdir -p $PATH_TO_MONARC_DATA/cache $PATH_TO_MONARC_DATA/DoctrineORMModule/Proxy $PATH_TO_MONARC_DATA/LazyServices/Proxy $PATH_TO_MONARC_DATA/import/files
# Create data directory symlink
ln -s $PATH_TO_MONARC_DATA $PATH_TO_MONARC/data
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

To prevent permission problems, we set the appropriate permissions for the Apache user once again.

```bash
sudo chown -R www-data:www-data /var/lib/monarc/fo/public/
sudo chown -R www-data:www-data /var/lib/monarc/fo-data/
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

## Problems?

If you have any problems during the installation or if you have any questions, please feel free to use the official MONARC [discussion area](https://github.com/monarc-project/MonarcAppFO/discussions)
