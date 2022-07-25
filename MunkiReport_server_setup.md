# MunkiReport on a Digital Ocean Docker droplet

Here's a quick summary of how I set up a MunkiReport server on Digital Ocean using Docker. My setup uses MySQL for the database and Caddy which handles maintaining a valid Let's Encrypt certificate for https.

(I'm out of my depth here so if you see strange choices, obvious mistakes, etc., it's probably because I just didn't know any better.)

*Note: since I wrote this, the official Docker setup instructions have been updated. You may want to review those, as they were definitely written by someone who understands this stuff better than I. https://github.com/munkireport/munkireport-php/wiki/Docker* 

## Create a Droplet 
- create a droplet on Digital Ocean using the "Docker 19.x on Ubuntu 20.x" image. I used the cheapest type of droplet & figure I'll upgrade if I need more RAM or whatever.
- follow the basic ubuntu security guide to make sure your setup is reasonably secure. I created a user to use instead of root, and granted sudo access. I can't figure out why this is safer than using root and just not letting your password/key get stolen but that's probably my problem.

## Set up DNS
- create an A record pointing to your droplet. Once I had the IP of my new droplet, I created a munkireport.example.net DNS record

## Configure stuff
Three files in the right places and one command will get this going.

### docker-compose.yml
- docker-compose.yml contains the basic info defining the three docker containers: MunkiReport, MySQL, and Caddy.
In my users's home directory, I created **~/munkireport/docker-compose.yml** with the following contents

```
version: '3.8'
networks:
    web:
        external: true
services:
    munkireport:
        image: "ghcr.io/munkireport/munkireport-php:v5.7.1"
        container_name: munkireport
        env_file:
            - /var/docker/munkireport.env
        volumes:
            - /var/docker/munkireport-modules:/var/munkireport/custom-modules
            - /var/docker/munkireport-local:/var/munkireport/local
        depends_on:
            - mrdb
        restart: always
        networks:
            - web
    mrdb:
        image: "mysql/mysql-server:5.7"
        container_name: mrdb
        environment:
            MYSQL_DATABASE: 'munkireport_db'
            MYSQL_USER: 'root'
            MYSQL_ROOT_PASSWORD: 'redacted'
        volumes:
            - /var/docker/munkireport:/var/lib/mysql
        ports:
            - "3306:3306"
        restart: always
        networks:
            - web
    caddy:
        image: "abiosoft/caddy"
        container_name: caddy
        environment:
            - ACME_AGREE=TRUE
        volumes:
            - /var/docker/caddy/Caddyfile:/etc/Caddyfile
            - /var/docker/caddy:/root/.caddy
        ports:
            - "80:80"
            - "443:443"
        depends_on:
            - munkireport
        networks:
            - web
        restart: always
```

### munkireport.env 
munkireport.env customizes the MunkiReport instance.
The location of this file is defined in docker-compose.yml. Mine lives at **/var/docker/munkireport.env** and looks like this

```
INDEX_PAGE=index.php?
SITENAME=MunkiReport
WEBHOST=https://munkireport.example.net
TIMEZONE=America/New_York
VNC_LINK=vnc://%s:5900
SSH_LINK=ssh://local@%s
DISPLAYS_INFO_KEEP_PREVIOUS=TRUE
AUTH_METHODS="LOCAL"
AUTH_SECURE=TRUE
ROLES_ADMIN="adminuser"
ENABLE_BUSINESS_UNITS=TRUE
CONNECTION_DRIVER=mysql
CONNECTION_HOST=mrdb
CONNECTION_DATABASE=munkireport_db
CONNECTION_USERNAME=mkrp
CONNECTION_PASSWORD=redacted
MODULES=kernel_panics, ms_office, ssd_service_program, applications, appusage, bluetooth, certificate, disk_report, displays_info, extensions, fan_temps,
 filevault_status, fonts, gpu, ibridge, installhistory, inventory, localadmin, managedinstalls, mdm_status, munkiinfo, munkireport, munkireportinfo, netw
ork, network_shares, power, printer, profile, security, softwareupdate, supported_os, usage_stats, usb, user_sessions, warranty, wifi
APPS_TO_TRACK=Safari, Firefox, Google Chrome, Java, 1Password%, Adobe Reader, Numbers, Keynote, Pages, Microsoft Remote Desktop, Microsoft Excel, Microso
ft Outlook, Microsoft PowerPoint, Microsoft Word
BUNDLEID_IGNORELIST=com.adobe.Acrobat.Uninstaller, com.adobe.ARM, com.adobe.air.Installer, com.adobe.distiller, com.parallels.winapp.*, com.vmware.proxyA
pp.*, com\.apple\.(appstore|airport.*|Automator|keychainaccess|launchpad|iChat|calculator|iCal|Chess|ColorSyncUtility|AddressBook|ActivityMonitor|iTunes|
VoiceOverUtility|backup.*|TextEdit|Terminal|systempreferences|mail|PhotoBooth|gamecenter|Preview|Safari|Stickies|RAIDUtility|QuickTimePlayerX|NetworkUtil
ity|Image_Capture|grapher|Grab|FontBook|DiskUtility|DigitalColorMeter|Dictionary|dashboardlauncher|DVDPlayer|Console|BluetoothFileExchange|audio.AudioMID
ISetup|ScriptEditor2|MigrateAssistant|bootcampassistant|AudioMIDISetup|Photos|iBooksX|exposelauncher|Maps|FaceTime|reminders|Notes|siri.*|SystemProfiler|
launchpad.*)
BUNDLEPATH_IGNORELIST=/System/Library/.*, /Library/(?!Internet).*, /usr/.*, .*/Scripting.localized/.*, .*\.app\/.*\.app, /Applications/Utilities/Adobe.*,
 /Applications/Microsoft Office \d+/Office/.*, /Developer/*
HIDE_INACTIVE_MODULES=TRUE
MODULE_SEARCH_PATHS=/var/munkireport/custom-modules
DEBUG=TRUE

```

### Caddyfile
Caddyfile sets up Caddy to do the https stuff for the MunkiReport container.
The location of this file is also defined in docker-compose.yml. Mine lives at **/var/docker/caddy/Caddyfile** and looks like this

```
munkireport.example.net {
  proxy / http://munkireport {
    transparent
    insecure_skip_verify
  }
  tls me@example.net
}

```
(I can't believe it's this easy.)

## Next Steps

With these three files in place, you can start everything up with `sudo docker-compose up` (run this in the directory containing docker-compose.yml). When everything's working right, you can use `sudo docker-compose up -d` to send the Docker stuff to the background and get back to a shell prompt.

To create a local admin user for MunkiReport (the username is defined in the munkireport.env as "adminuser"), visit 
https://munkireport.example.net/index.php?/auth/generate/ to define the username and password. You'll download an adminuser.yml file, and place this file in /var/docker/munkireport-local/users.
Now you can visit https://munkireport.example.net and log in and the rest is up to you!

## Upgrading

I've done one upgrade to my MunkiReport server, to update MunkiReport from 5.6.5 to 5.7.1 (required for Monterey 12.3+). The process was quite simple!
  1. Edit the 7th line of docker-compose.yml to specify the newer version of MunkiReport (https://github.com/munkireport/munkireport-php/pkgs/container/munkireport-php/15262697?tag=v5.7.1)
  2. Take the enviromnent down with `sudo docker-compose down`
  3. Update MunkiReport & start everything back up with `sudo docker-compose up -d`

Note: You'll also want to download the latest client installer and distribute that. Instructions are here under "Generate an Installer Package (pkg)": https://github.com/munkireport/munkireport-php/wiki/Client-setup

## Questions or Feedback?

I'd love to hear from you! You can find me in the [MacAdmins Slack](https://www.macadmins.org) as @adamrice and on Twitter at [@_AskAdam](https://twitter.com/_AskAdam)
