Nightscout on Ubuntu VPS server with Nginx

Full Instruction on: https://www.michael-schloemp.de/2022/08/28/nightscout-on-ubuntu-vps-server-with-nginx-translated/

This guide was translated using Google Translate and may contain translation errors.
I beg your pardon.

The Nightscout installation with Apache 2, as written in the original guide, has not proven to be as practical.
In general, everything works with Apache 2, but only as long as you are on a wireless network without port blocking.
If a port lock comes into play, you can no longer call up Nightscout.
If you now set up port forwarding via Apache 2 (instructions here), this works, but not with AAPS.
AAPS throws an XHR poll error when using port forwarding with Apache 2 and cannot connect to Nightscout.
For this reason there is now a new guide here.
Install Nightscout on a VPS running Ubuntu 20.04 and Nginx (pronounced Engin X).

Adding to this guide: I am working with nginx for the first time. If there is an error in the instructions, I would be happy to receive constructive criticism.




After successful installation of the server we start with the preparations.
Login with Putty of course required.

Uninstall Apache2

First, Apache 2 is uninstalled, for this we stop the Apache service.

$ sudo service apache2 stop

Perform uninstallation:

$ sudo apt-get purge apache2 apache2-utils

also delete leftovers:

$ sudo rm -rf /etc/apache2
$ sudo rm -rf /usr/sbin/apache2
$ sudo rm -rf /usr/lib/apache2
$ sudo rm -rf /etc/apache2
$ sudo rm -rf /usr/share/apache2
$ sudo rm -rf /usr/share/man/man8/apache2.8.gz

Find files and directories related to Apache

$ whereis apache2

If directories are still displayed, delete them as well.

Update Ubuntu

$ apt update

If updates are available, install them

$ apt upgrade

Install important components

Firewall (Firewall will be set up later):

$ apt install etc

nginx:

$ apt install nginx

Nano:

$ apt-get install nano

Git:

$ apt install git

Python:

$ apt install python

GCC, if not already present:

$ apt install gcc

Install MongoDB

$ apt install mongodb

After successful installation check the Mongo DB status:

$ systemctl status mongodb

MongoDB should be active, now let's check the connection:

$mongo --eval 'db.runCommand({ connectionStatus: 1 })'

Output:

MongoDB shell version v3.6.8
connecting to: mongodb://127.0.0.1:27017
Implicit session: session { "id" : UUID("e3c1f2a1-a426-4366-b5f8-c8b8e7813135") }
MongoDB server version: 3.6.8
{
"authInfo" : {
"authenticatedUsers" : [ ],
"authenticatedUserRoles" : [ ]
},
"okay" : 1
}

Create Mongo database

(“change username” “password” as desired and remember/write it down)

$ mongo
> use Nightscout
> db.createUser({user: "username", pwd: "password", roles:["readWrite"]})
> quit()

Create user

Next, a new Ubuntu user is created (mainuser=username).

Create new user:

$adduser mainuser

Grant root permission to mainuser:

$usermod -aG sudo mainuser

Check authorization:

$su mainuser

grep '^sudo' /etc/group

Stay logged in as main user!

Install nodejs

$ sudo apt install nodejs
$ sudo apt install build-essential checkinstall
$ sudo apt install libssl-dev
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash

– restart console – login with mainuser –

$ source /etc/profile
$nvm ls-remote
$nvm install 14.18.1
$nvm list
$nvm use 14.18.1

Install NPM

$ sudo apt install npm

Git clones

Check which directory you are in:

$pwd
> /home/mainuser

If not in the home/mainuser directory

§ cd /home/mainuser

Git Copy/Clone

$ git clone https://github.com/nightscout/cgm-remote-monitor.git
$ cd cgm-remote-monitor
$ npm install

After installation

$nano start.sh

insert and save the following text. Change/specify "user" "password" of the Mongo database, specify API_Secret as desired, change base url if necessary. Which plugins/modules are required depends on the user individually.
The plugins/modules listed here are optimal for me. There is a list of possible plugins on github
https://github.com/nightscout/cgm-remote-monitor#plugins :

#!/bin/bash

# environment variables
export DISPLAY_UNITS="mg/dl"
export MONGO_CONNECTION="mongodb://benutzer:passwort@localhost:27017/Nightscout"
export BASE_URL="127.0.0.1:1337"
export API_SECRET="12-stellige-API-Secret-Code"
export PUMP_FIELDS="reservoir battery status"
export DEVICESTATUS_ADVANCED=true
export ENABLE="careportal loop iob cob openaps pump bwg rawbg basal cors direction timeago devicestatus ar2 profile boluscalc food sage iage cage alexa basalprofile bgi directions bage upbat googlehome errorcodes reservoir battery openapsbasal"
export TIME_FORMAT=24
export INSECURE_USE_HTTP=true
export LANGUAGE=de
export EDIT_MODE=on
export PUMP_ENABLE_ALERTS=true
export PUMP_FIELDS="reservoir battery clock status"
export PUMP_RETRO_FIELDS="reservoir battery clock"
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
export OPENAPS_FIELDS="status-symbol status-label iob meal-assist rssi freq"
export OPENAPS_RETRO_FIELDS="status-symbol status-label iob meal-assist rssi"
export LOOP_ENABLE_ALERTS=false
export LOOP_WARN=30
export LOOP_URGENT=60
export SHOW_PLUGINS=careportal
export SHOW_FORECAST="ar2 openaps"

# start server
/home/mainuser/.nvm/versions/node/v14.18.1/bin/node server.js

Save and exit with: Ctrl+X - Y - enter

$ chmod 775 start.sh
$ ./start.sh

After success message Ctrl+c

Set up nightscout service

When setting up the Nightscout Service, there are different statements about the "Type". I use the type "simple" from the beginning.
The "forked" type is recommended in various forums. The type "simple" should normally suffice. If Nightscout doesn't start, you can change the type to "forked" afterwards. After each change to the Nightscout Service or the start.sh file, the Nightscout Service must be restarted (sudo systemctl restart nightscout.service)

$ sudo nano /etc/systemd/system/nightscout.service

insert and save:

[Unit]
Description=Nightscout Service
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/mainuser/cgm-remote-monitor
ExecStart=/home/mainuser/cgm-remote-monitor/start.sh

[Install]
WantedBy=multi-user.target

Save and exit with: "Ctrl+X" - "Y" - "enter"
Reload systemd:

$ sudo systemctl daemon-reload

Activate and start Nigtscout service:

$ sudo systemctl enable nightscout.service
$ sudo systemctl start nightscout.service

Display Nightscout service status:

$ sudo systemctl status nightscout.service

Output:

● nightscout.service – Nightscout Service
Loaded: loaded (/etc/systemd/system/nightscout.service; enabled; vendor preset: enabled)
Active: active (running)
[..]

Assign domain to host

(Main domain and a subdomain for ns).
So that Nightscout is not immediately found by strangers via the main domain, I use a subdomain.

$ sudo nano /etc/hosts

[...]
IP.of.your.server.Domain.de

IP.of.your.server.sub.domain.de

[...]

Save with "ctrl+X" - "Y" - "enter"

Create a configuration for main domain and subdomain

For the main domain we configure the default nginx config. Here we enter the server_name.

$ sudo nano /etc/nginx/sites-available/default

server_name domain.de;
server_name www.domain.de;

Next, set up the subdomain (unencrypted port forwarding)

$ sudo nano /etc/nginx/sites-available/Sub.domain.de.conf

Copy this text, change server_name to your subdomain before:

server {
listen 80;

server_name Sub.domain.de;
server_name www.sub.doamin.de;

location / {
proxy_pass http://127.0.0.1:1337;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_set_header X-Forwarded-Proto "https";
}
}

Next, enable the Nginx configuration for the subdoamin:

$ sudo ln -s /etc/nginx/sites-available/Sub.doamin.de.conf /etc/nginx/sites-enabled/

Restart Nginx for the configuration to take effect.

$ sudo service nginx restart

Install Certbot for Nginx

If not already installed, install certbot/python3:

$ sudo apt install certbot python3-certbot-nginx

Request certificate:

Certificate for the main domain:

$ sudo certbot --nginx -d domain.de -d www.doamin.de

The terms of use must be accepted when the certificate is created for the first time.
Your email address will also be requested, this must be entered.
Advertising from partners can be answered with no.

Confirm redirect with 2! This means that every call to your site is automatically routed to SSL encryption.

–> Enter email address “Enter”
–> Confirm terms of use “A”+”Enter”
-> Newsletter/News "N"+"Enter"
-> After a moment select whether all requests should be redirected to https.
1 = No 2 = Yes confirm with "Enter".

Then request a certificate for the subdomain:

$ sudo certbot --nginx -d Sub.domain.de -d www.Sub.domain.de

Confirm redirect with 2.

Check Certbot timer:

$ sudo systemctl status certbot.timer

Test renewal process

$ sudo certbot renew --dry-run

Renew certificate or redirect

I can not understand this step, but creates what is desired. Request or repair the certificate again for the subdomain. I had to do this step with every test installation so that the subdomain was properly encrypted and forwarded to the Nightscout port.

To do this, simply enter the certificate request command again:

$ sudo certbot --nginx -d Sub.domain.de -d www.Sub.domain.de

It is reported that a certificate already exists. Here with "1" restore the certificate.

1: Attempt to reinstall this existing certificate

Now the encrypted port forwarding should work properly.

Configure/enable firewall

To configure the firewall, log in with the root account!
Release port 27017 (allows external access to Mongo DB e.g. with Robot3T)
Allow Nginx and OpenSSH.
Release port 1337 (Opinions differ here as to whether it makes sense or not.)

$ufw allow 1337
$ufw allow 'Nginx Full'
$ufw allow 27017
$ufw allow OpenSSH

Next, we enable the firewall:

$ ufw enable

To see whether the firewall is activated and the above settings are allowed, check the status:

$ ufw status

The result should look like this:

State: active

To Action From
------ ----
1337 ALLOW Anywhere
27017 ALLOW Anywhere
OpenSSH ALLOW Anywhere
Apache Full ALLOW Anywhere
Nginx Full ALLOW Anywhere
1337 (v6) ALLOW Anywhere (v6)
27017 (v6) ALLOW Anywhere (v6)
OpenSSH (v6) ALLOW Anywhere (v6)
Apache Full (v6) ALLOW Anywhere (v6)
Nginx Full (v6) ALLOW Anywhere (v6)

Now the nginx test page should appear when you call up the main domain (automatically encrypted) and the Nightscout installation should appear when you call up the subdomain (without specifying port 1337, since a forwarding has already been set up above)

Optionally, you can restart the server once,
The boot process until Nightscout is fully accessible again can take up to 5 minutes.
