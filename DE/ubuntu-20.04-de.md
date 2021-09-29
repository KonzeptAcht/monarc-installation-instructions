
# Installation Anleitung Ubuntu 20.04

## Vorraussetzungen

- Ubuntu 20.04 LTS server installation
- Internet Zugang
- Command line Zugriff (Putty/SSH/direct access)


## Installation

:warning: In der nachfolgenden Anleitung werden die Platzhalter „monarc.domain.de“ und „password“ verwendet. Diese sind gegen die bei ihnen genutzten Werte auszutauschen.

### Preparation

Zunächst bereiten wir die installierte Ubuntu Server Instanz vor. Zunächst bringen wir das System auf den aktuellsten Stand bevor wir mit der installation von firewalld als Firewall-Managementtool und vim als Editor beginnen. Diese sind unserer Meinung nach am einfachsten bedienbar. Sie können natürlich andere Tools einsetzen. Bitte beachten sie dann im folgenden, die entsprechenden Zeilen für sich anzupassen.

```bash
sudo apt update && apt upgrade
sudo apt install firewalld vim
```

Nach erfolgreicher Installation richten wir firewalld für den Autostart ein und starten den Dienst.

```bash
sudo systemctl enable firewalld
sudo systemctl start firewalld
```

Im nächsten Schritt richten wir die notwendigen Firewall-Richtlinien ein. In unserem Fall greifen wir auf den Ubuntu Server mittels SSH zu. Daher werden wir auch die hierfür notwendigen Ports freigeben.

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

Abschließend installieren wir einige Tools und Abhängigkeiten, für die spätere MONARC Installation.

```bash
sudo apt install zip unzip git gettext curl gsfonts
```

### Installation MariaDB database server

Wir beginnen mit der Installation des Datenbankservers und der Clientsoftware.

```bash
sudo apt install mariadb-server mariadb-client
```

Anschließend richten wir hier ebenfalls den Autostart ein und starten den Dienste initial.

```bash
sudo systemctl enable mariadb
sudo systemctl start mariadb
```

Nach erfolgreichem Start des Dienstes können wir mit der Konfiguration beginnen.

```bash
sudo mysql_secure_installation
```

Hier sind folgende Angaben zu machen:

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

Nach erfolgreicher Installation bereiten wir die Datenbank auf die Installation von MONARC vor. Hierfür loggen wir uns zunächst mit der Anmeldedaten des „root“ Users am Datenbankdienst an und erstellen anschließend den MONARC Datenbankbenutzer und die Datenbanken.

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

Zuerst installieren wir die erforderlichen Apache- und PHP 7.4-Pakete.

```bash
sudo apt install apache2 php php-{curl,gd,mysql,pear,apcu,xml,mbstring,intl,imagick,zip,bcmath} libapache2-mod-php
```

Um später Zugriff auf die MONARC Umgebung zu erhalten, bei Apache ein sogenannter „Virtual Host“ einzurichten. In der hier zur verfügung gestellten Konfiguration sind einige teile auskommentiert. Diese Fassung ermöglicht es unverschlüsselten Zugriff auf die installierte MONARC Instanz zu erhalten. Es ist jedoch sehr zu empfehlen, die Verbindung mittels SSL/TLS Zertifikat abzusichern. Hierfür ist die Konfigurationsdatei bereits vorbereitet.

Zur Erstellung eines „Virtual Host“ wechseln wir in das Apache Konfigurationsverzeichnis

```bash
cd /etc/apache2/sites-available/
```

Hier legen wir nun eine neue Datei an und benennen sie entsprechend der Domain, unter der die Installation betrieben werden soll.

```bash
vim monarc.domain.de.conf
```

Hier fügen wir die folgende Konfiguration ein und speichern diese ab.

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

Abschließend aktivieren wir die benötigten Module:

```bash
sudo a2dismod status
sudo a2enmod ssl
sudo a2enmod rewrite
sudo a2enmod headers
```

Um unsere Änderungen am Virtual Host zu aktivieren ist nun noch ein symbolischer Link anzulegen.


```bash
cd /etc/apache2/sites-enabled/
ln -s ../sites-available/monarc.domain.de.conf monarc.domain.de.conf
```

Um zu Prüfen, ob unsere Konfiguration fehlerfrei ist, können wir nun noch einen Check durchführen.

```bash
sudo apachectl -S
```

In der Auflistung sollte unsere angelegte Datei aufgeführt sein. Zudem sollten keinerlei Fehler ausgegeben werden.

Damit können wir den Autostart für Apache einrichten und den Dienst starten

```bash
sudo systemctl enable apache2
sudo systemctl start apache2
```

### PHP settings

Da MONARC zum aktuellen Zeitpunkt bei größeren Risikomanagementstrukturen noch Probleme mit den Standard Timeouts von PHP haben kann, sollten folgende Grenzwerte in der php.ini angepasst werden:

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
Um später bei möglichen Uplouds in die MONARC Umgebung keine Schwierigkeiten zu bekommen, stellen wir in der `/etc/php/7.4/apache2/php.ini` das Uploudlimit höher ein

```bash
sudo vim /etc/php/7.4/apache2/php.ini
```
```bash
upload_max_filesize = 200M
```

und änder diese entsprechend ab. In vim wird mit der Taste „i“ in den Edit-Modus gewechselt. Nach erfolgreicher Editierung, kann mit der Taste „ESC“ das Editieren beendet und mit „:wq“ die Änderungen gespeichert und das Dokument verlassen werden.

### Optional: Installation with secured SSL/TLS certificate

Für eine sichere Installation über ein SSL/TLS-Zertifikat verwenden wir "certbot" von LetsEncrypt. Wenn der Server nicht öffentlich über DNS aufgelöst werden kann, sollte dieser Schritt übersprungen werden. In diesem Fall können Sie PKI verwenden oder manuell ein Zertifikat mit Openssl erzeugen und es in der Virtual Host-Datei und im Client speichern.

#### Let's Encrypt

```bash
sudo apt install certbot python-certbot-apache
```

Nach erfolgreicher installation, und Erstellung des „Virtual Host“ aus dem vorherigen Schritt, können wir ein Zertifikat generieren.

```bash
certbot certonly --apache -d monarc.domain.de
```

Ist das Zertifikat erfolgreich generiert worden, können die auskommentierten Teile aus der „Virtual Host“ Konfiguration eingeschaltet werden.

```bash
SSLCertificateFile    /etc/letsencrypt/live/monarc.domain.de/cert.pem
SSLCertificateKeyFile /etc/letsencrypt/live/monarc.domain.de/privkey.pem
SSLCertificateChainFile /etc/letsencrypt/live/monarc.domain.de/chain.pem
```

#### Self signed certificate

Mit dem folgenden Befehl können Sie ein SSL-Zertifikat erzeugen.

```bash
openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
```

Description:
```bash
  -newkey #Ein neues Schlüsselpaar soll generiert werden.
  rsa:2048 # Ein Schlüssel mit dem Algorithmus RSA und einer Schlüssellänge von 2048 Bit
  -nodes #Nicht mit 3DES-CBC verschlüsseln
  -keyout key.pem #Ausgabe Pfad und Name der Schlüsseldatei
  -x509 #X.509-Zertifikatsdatenverwaltung verwenden
  -days 365 #Das Zertifikat ist 365 Tage gültig
  -out certificate.pem #Ausgabe Pfad und Name der Zertifikatsdatei
```

Danach können wir das generierte Zertifikat überprüfen

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

Da MONARC "nodejs 14" benötigt und diese Version derzeit nicht Teil des Ubuntu-Repositorys ist, muss sie nun manuell installiert werden. Nach der Installation sind alle Abhängigkeiten erfüllt, um auch "grunt-cli" zu installieren.

```bash
curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
sudo apt-get install nodejs
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
sudo apt update && sudo apt upgrade
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
