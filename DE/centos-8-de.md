
# Installation Anleitung Centos 8

## Vorraussetzungen

- Centos 8 server installation
- Internet Zugang
- Command line Zugriff (Putty/SSH/direct access)


## Installation

:warning: In der nachfolgenden Anleitung werden die Platzhalter „monarc.domain.de“ und „password“ verwendet. Diese sind gegen die bei ihnen genutzten Werte auszutauschen.

### Preparation

Zunächst bereiten wir die installierte Centos Server-Instanz vor. Zuerst bringen wir das System auf den neuesten Stand, bevor wir mit der Installation und Konfiguration von firewalld als Firewall-Management-Tool und vim als Editor beginnen. Diese sind unserer Meinung nach am einfachsten zu benutzen. Sie können natürlich auch andere Tools verwenden. Bitte beachten Sie dann im Folgenden, die entsprechenden Zeilen für sich anzupassen.

```bash
sudo dnf update
sudo dnf install firewalld vim epel-release
```

Nach erfolgreicher Installation richten wir firewalld für den Autostart ein und starten den Dienst.

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

Im nächsten Schritt richten wir die erforderlichen Firewall-Richtlinien ein. In unserem Fall greifen wir über SSH auf den Centos-Server zu. Daher werden wir auch die dafür notwendigen Ports freigeben.

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Schließlich installieren wir noch einige Tools und Abhängigkeiten für die spätere MONARC-Installation.

```bash
sudo dnf install zip unzip git gettext curl tar gzip wget
```

### Installation MariaDB Datenbank server

Wir beginnen mit der Installation des Datenbankservers und der Client-Software.

```bash
sudo dnf install mariadb-server mariadb
```

Anschließend richten wir auch hier den Autostart ein und starten zunächst die Dienste.

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

Nach erfolgreichem Start des Dienstes können wir mit der Konfiguration beginnen.

```bash
sudo mysql_secure_installation
```

Die folgenden Informationen sollten hier angegeben werden:

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

Damit ist die Installation des Datenbankdienstes abgeschlossen und wir können mit dem Apache-Webserver fortfahren.

### Installation Apache Webserver und PHP 7.4

Zunächst installieren wir den erforderlichen Apache Server mit den PHP 7.4-Paketen. Da PHP 7.4 in Centos nicht standardmäßig auf der aktuellsten Version enthalten ist, beziehen wir es aus dem REMI-Repository

```bash
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-8.rpm
sudo dnf module enable php:remi-7.4
sudo dnf install httpd php php-{curl,gd,mysql,pear,apcu,xml,mbstring,intl,imagick,zip,bcmath}
```

Um später auf die MONARC-Umgebung zugreifen zu können, muss bei Apache ein "Virtual Host" eingerichtet werden. In der hier bereitgestellten Konfiguration sind einige Teile auskommentiert. Diese Version ermöglicht einen unverschlüsselten Zugriff auf die installierte MONARC-Instanz. Es wird jedoch dringend empfohlen, die Verbindung mit einem SSL/TLS-Zertifikat abzusichern. Die Konfigurationsdatei ist für diesen Zweck bereits vorbereitet.

Um einen "Virtual Host" zu erstellen, wechseln wir in das Apache-Konfigurationsverzeichnis

```bash
sudo mkdir /etc/httpd/sites-available
sudo mkdir /etc/httpd/sites-enabled
cd /etc/httpd/sites-available/
```

Hier legen wir nun eine neue Datei an und benennen sie nach der Domäne, unter der die Installation laufen soll.

```bash
vim monarc.domain.de.conf
```

Hier fügen wir die folgende Konfiguration ein und speichern sie.

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
#SSLProtocol -all +TLSv1.3 +TLSv1.2 +TLSv1.1
#SSLHonorCipherOrder on

# Encoded slashes need to be allowed
AllowEncodedSlashes             NoDecode

#SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
#SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
#SSLCertificateChainFile /etc/letsencrypt/live/monarc.domain.de/chain.pem
</VirtualHost>
```

Schließlich fügen wir unseren Konfigurationsordner zu den Apache-Einstellungen hinzu:

```bash
sudo vim /etc/httpd/conf/httpd.conf
```

In dieser Datei ergänzen wir am Ende die folgende Zeile

```bash
IncludeOptional sites-enabled/*.conf
```

Um unsere Änderungen am virtuellen Host zu aktivieren, müssen wir nun einen symbolischen Link erstellen.

```bash
cd /etc/httpd/sites-enabled/
ln -s ../sites-available/monarc.domain.de.conf monarc.domain.de.conf
```

Um zu überprüfen, ob die Konfiguration richtig funktioniert, können Sie sie vorher auf Syntaxfehler überprüfen

```bash
sudo apachectl -S
```

Die von uns erstellte Datei sollte in der Ausgabe aufgeführt sein. Außerdem sollten keine Fehler gemeldet werden.

Damit können wir den Autostart für Apache einrichten und den Dienst starten.

```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```

### PHP settings

Da MONARC bei größeren Risikomanagement-Strukturen derzeit noch Probleme mit den Standard-Timeouts von PHP haben kann, empfehlen wir, den folgenden Parameter in der php.ini anzupassen:

```bash
sudo vim /etc/php.ini
```
```bash
...
max_execution_time = 0 #Achtung, diese Einstellung deaktiviert den Timeout für PHP, wenn andere Anwendungen auf dem Server verwendet werden, sollte geprüft werden, ob diese Einstellung verwendet werden kann.
...
max_input_time = 0 #Achtung, diese Einstellung deaktiviert den Timeout für PHP, wenn andere Anwendungen auf dem Server verwendet werden, sollte geprüft werden, ob diese Einstellung verwendet werden kann.
...
memory_limit = 2048M
...
```
Um spätere Probleme mit möglichen Uploads in die MONARC-Umgebung zu vermeiden, setzen wir das Upload-Limit in `/etc/php.ini` höher

```bash
sudo vim /etc/php.ini
```
```bash
upload_max_filesize = 200M
```

### Optional: Installation with secured SSL/TLS certificate

Für eine sichere Installation über ein SSL/TLS-Zertifikat verwenden wir "certbot" von LetsEncrypt. Wenn der Server nicht öffentlich über DNS aufgelöst werden kann, sollte dieser Schritt übersprungen werden. In diesem Fall können Sie eine PKI verwenden oder manuell ein Zertifikat mit Openssl erzeugen und es in der Virtual Host-Datei und im Client speichern.

#### Let's Encrypt

```bash
sudo dnf install certbot python-certbot-apache
```

Nach erfolgreicher Installation und der Erstellung des "virtuellen Hosts" aus dem vorherigen Schritt können wir ein Zertifikat erstellen.

```bash
certbot certonly --apache -d monarc.domain.de
```

Wenn das Zertifikat erfolgreich generiert wurde, können die kommentierten Teile der "Virtual Host"-Konfiguration eingeschaltet werden. Denken Sie daran, die folgenden Pfade in der Datei anzupassen.

```bash
SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
SSLCertificateChainFile /etc/letsencrypt/live/monarc.domain.de/chain.pem
```

#### Selbst signierte Zertifikate

Mit dem folgenden Befehl können Sie ein SSL-Zertifikat erzeugen.

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
```

Beschreibung:
```bash
  -newkey #Ein neues Schlüsselpaar soll generiert werden.
  rsa:2048 # Ein Schlüssel mit dem Algorithmus RSA und einer Schlüssellänge von 2048 Bit
  -nodes #Nicht mit 3DES-CBC verschlüsseln
  -keyout key.pem #Ausgabe Pfad und Name der Schlüsseldatei
  -x509 #X.509-Zertifikatsdatenverwaltung verwenden
  -days 365 #Das Zertifikat ist 365 Tage gültig
  -out certificate.pem #Ausgabe Pfad und Name der Zertifikatsdatei
```

after this we can check the generated certificate

```bash
openssl x509 -text -noout -in certificate.pem
```

Anschließend können Sie die erzeugten Dateien in der virtuellen Hostdatei speichern. Da es keine Vertrauensstruktur gibt, müssen nur die beiden folgenden Zeilen bearbeitet werden. Darüber hinaus können die anderen kommentierten Konfigurationen aktiviert werden. Um die Kommunikation abzusichern, sollte die Zertifikatsdatei ebenfalls auf den Client Geräten eingespielt werden.

```bash
SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
```

### Installation Composer

Eine weitere Abhängigkeit von diesem Produkt ist der Composer. Er ist für die Einrichtung und spätere Aktualisierung des Produkts unerlässlich. Da die Installation nicht über die offiziellen Repositories erfolgen kann, da eine Version >= 2.1.0 benötigt wird, wird sie hier nochmals erläutert.

Mit dem folgenden Befehl können Sie das Installationsskript herunterladen.

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
```

Anschließend sollte die Signatur überprüft werden, um sicherzustellen, dass das Skript nicht kompromitiert ist.

```bash
HASH="$(wget -q -O - https://composer.github.io/installer.sig)"
php -r "if (hash_file('SHA384', 'composer-setup.php') === '$HASH') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
```

Wenn die Hash-Werte übereinstimmen, sollte die folgende Meldung ausgegeben werden.
`Installer verified`

Damit ist die Installation vorbereitet und wir können das Skript wie folgt starten:
```bash
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```

Der folgende Befehl kann ausgeführt werden, um den Versionsstand zu überprüfen. Es sollte mindestens die Version >=2.1.0 installiert sein.

```bash
composer -V
```

### MONARC Installation

Wir beginnen damit, das Installationsverzeichnis für MONARC zu erstellen, die Installationsdateien von Git herunterzuladen und einige Ordnereinstellungen vorzunehmen.

```bash
mkdir -p /var/lib/monarc/fo
git clone https://github.com/monarc-project/MonarcAppFO.git /var/lib/monarc/fo
cd /var/lib/monarc/fo
mkdir -p data/cache
mkdir -p data/LazyServices/Proxy
chmod -R g+w data
```

Mit dem folgenden Befehl installieren wir alle offenen Abhängigkeiten.

```bash
composer install -o
```

Anschließend müssen einige Anpassungen und Verknüpfungen für das von MONARC verwendete Framework erstellt werden.

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

Nachdem wir nun alle Dateien von MONARC haben, können wir die Datenbank mit dem erforderlichen Schema und den entsprechenden Informationen initialisieren.

```bash
mysql -u monarc -p monarc_common < db-bootstrap/monarc_structure.sql
mysql -u monarc -p monarc_common < db-bootstrap/monarc_data.sql
```

Damit die MONARC-Instanz Zugriff auf die erstellte Datenbank hat, muss eine Konfigurationsdatei erstellt werden. Hierfür wird uns eine Vorlage zur Verfügung gestellt, die wir kopieren und entsprechend anpassen.

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

Dadurch wird die Datenbankverbindung hergestellt.

:bulb: Da mit der Version 2.9.17 ein Erweiterungsmodul zur Generierung von Statistiken aus erstellten Risikoanalysen veröffentlicht wurde, sind seither weitere Anpassungen an der local.php vorzunehmen, wenn diese Funktion genutzt werden soll. Weitere Informationen finden Sie in diesem Fall unter folgendem Link: https://www.monarc.lu/documentation/stats-service/master/installation.html

Da MONARC "nodejs 14" benötigt. Nach der Installation sind alle Abhängigkeiten erfüllt, um auch "grunt-cli" zu installieren.

```bash
sudo dnf module list nodejs
sudo dnf module enable nodejs:14
sudo dnf install nodejs
sudo npm install -g grunt-cli
```

Schließlich führen wir das Update-Skript aus, das die vorbereitete MONARC-Instanz betriebsbereit macht.

```bash
./scripts/update-all.sh -c
```

Um Probleme mit den Berechtigungen zu vermeiden, setzen wir die entsprechenden Berechtigungen für den Apache-Benutzer noch einmal.

```bash
sudo chown -R www-data:www-data /var/lib/monarc/fo/public
```

MONARC ist nun betriebsbereit und kann über die konfigurierte Domain angesprochen werden. Um sich an der Weboberfläche anmelden zu können, muss lediglich ein Administrator angelegt werden.

```bash
php ./vendor/robmorgan/phinx/bin/phinx seed:run -c ./module/Monarc/FrontOffice/migrations/phinx.php
```

Die erstellten standard Anmeldedaten lauten wie folgt:
```bash
User: admin@admin.localhost
Password: admin
```

## MONARC Update

Wenn Sie monarc aktualisieren möchten, sollten Sie zunächst sicherstellen, dass Ihr System auf dem neuesten Stand ist.

```bash
cd /var/lib/monarc/fo/
sudo dnf update
composer update
npm install -g npm
```

Dann können wir das System mit dem vorbereiteten Skript aktualisieren

```bash
./scripts/update-all.sh -c
```

Schließlich löschen wir die im Cache vorhandenen Daten.

```bash
rm -rf data/cache/*
rm -rf data/DoctrineORMModule/Proxy/*
rm -rf data/LazyServices/Proxy/*
```

## Problems?

Wenn Sie während der Installation Probleme oder Fragen haben, können Sie sich an den offiziellen MONARC [Diskussionsbereich](https://github.com/monarc-project/MonarcAppFO/discussions) wenden.
