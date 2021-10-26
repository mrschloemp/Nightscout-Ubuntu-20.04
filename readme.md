Nightscout auf Ubuntu 20.04 installieren

Installation auf einem VPS R6 von 1blu.de. 
Der VPS hat 2 CPU Cores, 6 GB Ram und 80GB SSD u.v.m…, ausreichend für Nightsout.
https://www.1blu.de/vps-aktion/

Eine zusätzliche Domain ist nicht zwangsläufig notwendig, kann man aber günstig dazu buchen. Eine Subdomain von 1blu ist beim Server enthalten. Let´s Encrypt funktioniert mit der Subdomain ohne Probleme (Die IP Adresse des Servers kann nicht mit Let´s Encrypt abgesichert werden!).
Da ich schon ein Webhosting Paket bei 1blu besitze, habe ich daraus eine Domain auf den Server per DNS Eintrag geleitet.

Einrichtung des Servers.
Ich setzte Grundkenntnisse Server/SSH/Konsole voraus.


Ich habe mich für Ubuntu 20.04 als Distribution entschieden.
Im Vorfeld habe ich Ubuntu 20.04 über das 1blu Kundeninterface installiert und mir die SSH Zugangsdaten notiert.
Als Konsole/SSH Client verwende ich Putty https://www.putty.org/

Beginnen wir mit dem Einrichten.

Putty öffnen, Serverdaten eingeben und via root User einloggen. Beim ersten Login via SSH muss das Passwort geändert werden.

Server Updates und diverse notwendige Plugins installieren
Neuen Benutzer anlegen:

$ adduser mainuser

mainuser Root Berechtigung erteilen:

$ usermod -aG sudo mainuser

Berechtigung prüfen:

$ su mainuser

$ grep ‚^sudo‘ /etc/group

Ubuntu aktualisieren:

$ sudo apt update

Wenn Aktualisierungen vorliegen:

$ sudo apt ugrade

Firewall UFW installieren:

$ sudo apt install ufw

Um die Konfiguration der Firewall kümmern wir uns zum Schluss.

Nano installieren

$ sudo apt-get install nano

Git installieren:

$ sudo apt install git

Phython sollte vorinstalliert sein, wenn nicht

$ sudo apt install phyton

Wenn Apache2 noch nicht aktiv/installiert

$ sudo apt install apache

Nodejs installieren

$ sudo apt install nodejs
$ sudo apt install build-essential checkinstall
$ sudo apt install libssl-dev
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash

– restart konsole – login mit mainuser –

$ source /etc/profile
$ nvm ls-remote
$ nvm install 14.18.1
$ nvm list
$ nvm use 14.18.1

NPM installieren

$ sudo apt install npm

GCC sollte installiert sein, wenn nicht

$ sudo apt install gcc

Ab hier kann mit Root Benutzer weiter gearbeitet.

Mongo DB installieren
$ sudo apt install mongodb
$ sudo systemctl status mongodb

MongoDB sollte active sein, Verbindfung prüfen:

$ mongo –eval ‚db.runCommand({ connectionStatus: 1 })‘

 

Ausgabe:

MongoDB shell version v3.6.8
connecting to: mongodb://127.0.0.1:27017
Implicit session: session { „id“ : UUID(„e3c1f2a1-a426-4366-b5f8-c8b8e7813135“) }
MongoDB server version: 3.6.8
{
„authInfo“ : {
„authenticatedUsers“ : [ ],
„authenticatedUserRoles“ : [ ]
},
„ok“ : 1
}

 

Mongo Datenbank erstellen („benutzername“ „passwort“ nach Wunsch ändern und merken/aufschreiben)

$ mongo
> use Nightscout
> db.createUser({user: „benutzername„, pwd: „passwort„, roles:[„readWrite“]})
> quit()

 

Prüfen in welchem Verzeichnis man sich befindet:

$ pwd
> /home/mainuser

Wenn nicht im home/mainuser verzeichnis

§ cd /home/mainuser

 

Git Kopieren/Clonen

$ git clone https://github.com/nightscout/cgm-remote-monitor.git
$ cd cgm-remote-monitor
$ npm install

Nach der Installation
$ nano start.sh

folgenden Text einfügen und speichern. „benutzer““passwort“ der MongoDatenbank ändern/angeben, API_Secret wunschgemäß Angeben, Base-Url ggf. ändern. Welche Plugins/Module benötigt werden hängt vom Anwender individuell ab.
Die hier aufgeführten Plugins/Module sind für mich Optimal. Eine Auflistung der möglichen Plugins gibt´s auf github
https://github.com/nightscout/cgm-remote-monitor#plugins  :

#!/bin/bash

# environment variables
export DISPLAY_UNITS=“mg/dl“
export MONGO_CONNECTION=“mongodb://benutzer:passwort@localhost:27017/Nightscout“
export BASE_URL=“127.0.0.1:1337"
export API_SECRET=“1234567890„
export PUMP_FIELDS=“reservoir battery status“
export DEVICESTATUS_ADVANCED=true
export ENABLE=“careportal loop iob cob openaps pump bwg rawbg basal cors direction timeago devicestatus ar2 profile boluscalc food sage iage cage alexa basalprofile bgi directions bage upbat googlehome errorcodes reservoir battery openapsbasal“
export TIME_FORMAT=24
export INSECURE_USE_HTTP=true
export LANGUAGE=de
export EDIT_MODE=on
export PUMP_ENABLE_ALERTS=true
export PUMP_FIELDS=“reservoir battery clock status“
export PUMP_RETRO_FIELDS=“reservoir battery clock“
export PUMP_WARN_CLOCK=30
export PUMP_URGENT_CLOCK=60
export PUMP_WARN_RES=50
export PUMP_URGENT_RES=10
export PUMP_WARN_BATT_P=30
export PUMP_URGENT_BATT_P=20
export PUMP_WARN_BATT_V=1.35
export PUMP_URGENT_BATT_V=1.30
export OPENAPS_ENABLE_ALERTS=false
export OPENAPS_WARN=30
export OPENAPS_URGENT=60
export OPENAPS_FIELDS=“status-symbol status-label iob meal-assist rssi freq“
export OPENAPS_RETRO_FIELDS=“status-symbol status-label iob meal-assist rssi“
export LOOP_ENABLE_ALERTS=false
export LOOP_WARN=30
export LOOP_URGENT=60
export SHOW_PLUGINS=careportal
export SHOW_FORECAST=“ar2 openaps“

#export SSL_KEY=/etc/letsencrypt/live/domain.de/privkey.pem
#export SSL_CERT=/etc/letsencrypt/live/domain.de/fullchain.pem

# start server
/home/mainuser/.nvm/versions/node/v14.18.1/bin/node server.js

Speichern und beenden mit: strg+X – Y – enter

$ chmod +100 +010 start.sh
$ ./start.sh

Nach Erfolgsmeldung strg+c

Nightscout Service einrichten
$ sudo nano /etc/systemd/system/nightscout.service

einfügen und speichern:

[Unit]
Description=Nightscout Service
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/mainuser/cgm-remote-monitor
ExecStart=/home/mainuser/cgm-remote-monitor/start.sh

[Install]
WantedBy=multi-user.target

Speichern und beenden mit: „strg+X“ – „Y“ – „enter“
Reload systemd:

$ sudo systemctl daemon-reload

Nigtscout Service aktivieren und starten:

$ sudo systemctl enable nightscout.service
$ sudo systemctl start nightscout.service

Nightscout Service Status anzeigen lassen:

$ sudo systemctl status nightscout.service

Ausgabe:

? nightscout.service – Nightscout Service
Loaded: loaded (/etc/systemd/system/nightscout.service; enabled; vendor preset: enabled)
Active: active (running)
[..]

Konfiguration VirtualHost
Standard Konfiguration kopieren (Standard zu deiner Domain). Rot Markierte Elemente auf deine Domain/Bedürnisse ändern.

$ sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/Domain.de.conf

Konfiguration abändern auf Deine Domain

$ sudo nano /etc/apache2/sites-available/Domain.de.conf

<VirtualHost *:80>
# The ServerName directive sets the request scheme, hostname and port that
# the server uses to identify itself. This is used when creating
# redirection URLs. In the context of virtual hosts, the ServerName
# specifies what hostname must appear in the request’s Host: header to
# match this virtual host. For the default virtual host (this file) this
# value is not decisive as it is used as a last resort host regardless.
# However, you must set it for any further virtual host explicitly.

ServerAdmin webmaster@domain.de
ServerName Domain.de
ServerAlias www.Domain.de
DocumentRoot /var/www/html

# Available loglevels: trace8, …, trace1, debug, info, notice, warn,
# error, crit, alert, emerg.
# It is also possible to configure the loglevel for particular
# modules, e.g.
#LogLevel info ssl:warn

ErrorLog ${APACHE_LOG_DIR}/Domain.de.error.log
CustomLog ${APACHE_LOG_DIR}/Domain.de.access.log combined

# For most configuration files from conf-available/, which are
# enabled or disabled at a global level, it is possible to
# include a line for only one particular virtual host. For example the
# following line enables the CGI configuration for this host only
# after it has been globally disabled with „a2disconf“.
#Include conf-available/serve-cgi-bin.conf
</VirtualHost>

Spichern mit „strg+x“ –> „Y“–>“enter“

Standard Konfiguration deaktivieren

$ sudo a2dissite 000-default.conf

Deine Konfiguration aktivieren

$ sudo a2ensite Domain.de.conf

Apache neustart

$ sudo systemctl restart apache2

Domain dem Host zuweisen

$ sudo nano /etc/hosts

[…]
IP.deines.Servers Domain.de
[…]

Speichern mit „strg+X“ – „Y“ – „enter“

Let´s Encrypt Zertifikat installieren
und mit Certbot automatisch aktualisieren

$ sudo apt install certbot python3-certbot-apache

Installation mit Y bestätigen,

Servername/Alias überprüfen

$ sudo nano /etc/apache2/sites-available/Domain.de.conf

Wenn ServerName und ServerAlias fehlt, nachtragen.

Konfiguration testen

$ sudo apache2ctl configtest

Wenn Syntax OK, Apache neu starten

$ sudo systemctl reload apache2

Zertifikat anfordern

$ sudo certbot –apache
–> Email Adresse eingeben „Enter“
–> Nutzungsbedingungen bestätigen „A“+“Enter“
–> Newsletter/Neuigkeiten „N“+“Enter“
–> Domain auswählen
–> Nach einem Moment auswählen ob alle Anfragen auf https umgeleitet werden sollen.
1 = Nein 2=Ja mit „Enter bestätigen

Erfolgsmeldung erschein (Congratulations! You have successfully enabled[…])

Certbot status überprüfen

$ sudo systemctl status certbot.timer

Es sollte die Meldung

Active: active (waiting)
erscheinen.

Let´s Encrypt Zertifikat wurde erstellt und wird mit Certbot automatisch aktualisiert.

Firewall Port freigaben und Firewall aktivieren:
$ sudo ufw allow 1337
$ sudo ufw allow 27017
$ ufw allow OpenSSH
$ sudo ufw allow ‚Apache Full‘
$ sudo ufw enable
$ sudo ufw status (Status und freigegebene Ports einsehen)

Kontrolle ob alles korrekt freigegeben wurde

$ sudo ufw status

Output
Status: active

To Action From
— —— —-
27017 ALLOW Anywhere
OpenSSH ALLOW Anywhere
Apache Full ALLOW Anywhere
1337 ALLOW Anywhere
27017 (v6) ALLOW Anywhere (v6)
OpenSSH (v6) ALLOW Anywhere (v6)
Apache Full (v6) ALLOW Anywhere (v6)
1337 (v6) ALLOW Anywhere (v6)

 

Server neu starten (ein paar Minuten warten…)

SSL für Nightscout aktivieren
Via SSH einloggen und ins Nightscout Basisverzeichnis wechseln.

$ cd /home/mainuser/cgm-remote-monitor
$ nano start.sh

bei #export SSL_KEY [..] das „#“ entfernen und deinen Pfad zum SSL Key eintragen

export SSL_KEY=/etc/letsencrypt/live/Domain.de/privkey.pem
export SSL_CERT=/etc/letsencrypt/live/Domain.de/fullchain.pem

Schließen und Speichern mit  „strg+X“ –>“Y“–>“enter“

Nightscout Service neu starten

$ sudo systemctl restart nightscout.service

Nightscout status Prüfen:

$ sudo systemctl status nightscout.service

Nun kann Nightscout über deine domain.de:1337 (ssl verschlüsselt) aufgerufen werden.
Es muss gegebenenfalls die Cache deines Browsers geleert werden wenn zuvor die Domain oder Nightscout aufgerufen wurde.