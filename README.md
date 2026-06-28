# Web Application Firewall Home Lab

## Introduction

In this lab, we will build a complete cybersecurity home lab using virtual machines. The goal is to first set up a vulnerable web application (DVWA) on Ubuntu. Demonstrate how to perform a basic SQL injection attack from Kali Linux then use the web application firewall (WAF) to protect against such attacks. This lab assumes you have basic Linux command line knowledge. 

## Prerequisites

* Host Machine

(at least 8 GB RAM and 50 GB disk space)

* Internet Connection

(for downloading software and updates)

## Lab Environment Setup

### Download and Install VirtualBox

Go to the official VirtualBox download page: https://www.virtualbox.org/wiki/Downloads and select the version for your host OS. Run the downloaded installer and follow the on-screen instructions.

### Create the Kali Linux Virtual Machine

1. Go to the official Kali Linux download site: https://www.kali.org/get-kali and select virtual machines to get sent to the page with pre-built inages. Select VirtualBox.

2. Attach the Kali ISO & Install:

* Go to Machine > Add, select the Kali virtual disk image you downloaded (you may have to decompress the file first).

* Start the VM, it should bring you straight to the login screen. Login is kali/kali by default.

![](/img/create-kali-vm.png)

### Create the Ubuntu Server Virtual Machine

1. Download Ubuntu from the official site: https://ubuntu.com/download/desktop and choose the latest LTS release.

2. Create a New VM in VirtualBox, start the VM and follow the Ubuntu Server installation instructions. Create a username/password (e.g., ubuntu / ubuntu).

* Name: UbuntuServer

* Type: Linux

* Version: Ubuntu (64-bit)

* Memory: At least 2 GB (2048 MB)

* Hard Disk: ~20 GB (dynamically allocated)

### Enable Bridged Networking

1. Open VirtualBox > select your VM (e.g., UbuntuServer) > Settings > Network.

2. Adapter 1: Choose Bridged Adapter from the “Attached to” drop-down.

3. Select your host’s network interface (Ethernet or Wi-Fi) then click OK.

4. Repeat for second VM.

![](/img/enable-bridged-networking.png)

## DVWA Installation

1. You will need to install the packages listed below then restart apache2.

* apache2

* libapache2-mod-php

* mariadb-server

* mariadb-client

* php php-mysqli

* php-gd

`apt install -y apache2 mariadb-server mariadb-client php php-mysqli php-gd libapache2-mod-php`

`sudo systemctl restart apache2`

2. Set up the database. Jump to root and create a new database user.

`sudo su -`

`mysql` (odd but keep going)

You should be in a MariaDB CLI now but not yet connected to database.

`create database dvwa;`

`create user dvwa@localhost identified by 'p@ssw0rd';` The User and Password must match the config.inc.php file from above.

`grant all on dvwa.* to dvwa@localhost;`

`flush privileges;` Tells database to reload authentication privileges.

![](/img/maria-database-creation.png)

You can check if you properly configured things by using `mysql -u root -p` then entering the password. If done properly you should have no trouble logging in from a separate terminal. Alternatively you can do `mysql -u root -ppassword` with no spaces between the tag and the password.

3. Download DVWA. This next section has four actions. Navigate to the web root, remove the default index, clone the dvwa git repository and set permissions.

`cd /var/www/html`

`sudo rm index.html`

`sudo git clone https://github.com/digininja/DVWA.git`

`sudo chmod -R 755 /var/www/html`

`sudo chown -R www-data:www-data /var/www/html`

4. Configure DVWA. DVWA ships with a dummy copy of its config file which you will need to copy into place and then make the appropriate changes. From the DVWA directory `cp config/config.inc.php.dist config/config.inc.php`. Or just `cp /var/www/html/config/config.inc.php.dist /var/www/html/config/config.inc.php`. Edit the file with `nano /var/www/html/config/config.inc.php`.

The contents of the file should look like this: 

```
$_DVWA[ 'db_server'] = '127.0.0.1';

$_DVWA[ 'db_port'] = '3306';

$_DVWA[ 'db_user' ] = 'dvwa';

$_DVWA[ 'db_password' ] = 'p@ssw0rd';

$_DVWA[ 'db_database' ] = 'dvwa';
```

Steps 1 to 4 can be automated by using the following script:

`sudo bash -c "$(curl --fail --show-error --silent --location https://raw.githubusercontent.com/IamCarron/DVWA-Script/main/Install-DVWA.sh)"`

5. DVWA Webpage

Navigate to `http://localhost` in your browser and click `Create/Reset Database`. You should be redirected to the login page. The default credentials are `admin/password`.

![](/img/dvwa-setup-page.png)

![](/img/dwva-database-setup-1.png)

![](/img/dvwa-database-setup-2.png)

![](/img/dvwa-login.png)

Now to give it a test to make sure it works lets try a reflected XSS attack. First change the security level to low in DVWA Security. Navigate to XSS (Reflected) on the DVWA webpage. It will ask you for your name. Place any alphanumeric string in the box as well as `<script> alert() </script>` then click submit. The browser should trigger a popup.

![](/img/dvwa-xss.gif)

5. Set DVWA to Listen on port 8080

By default, Apache listens on port 80. Edit the Apache Configuration and change it to 8080 then restart the apache2 service.

`sudo nano /etc/apache2/ports.conf`

Change `Listen 80` to `Listen 8080`

`sudo systemctl restart apache2`

## DNS Resolution Setup

Edit `/etc/hosts` on Both Ubuntu and Kali. Add a Line (replace <Ubuntu IP> with the actual IP address).

`sudo nano /etc/hosts`

`<Ubuntu IP> dvwa.local`

Once again we will test the DVWA but from the Kali VM. Navigate to SQL Injection and it will ask for User ID. Enter the following to perform an SQL Injection attack: `1' or '1'='1`. The database should return a list of all users.

![](/img/kali-sqli-test.gif)

6. Implement some firewall rules.

`sudo ufw default deny incoming`

`sudo ufw allow from 192.168.1.0/24 to any port 80`  Basically allow from your LAN only. Your LAN IP range may differ.

`sudo ufw enable`

____________________

## Install and Configure Web Application Firewall

1. We will be using Cyberserval's WAF Safeline. Navigate to cyberserval.tech/landing/safeline and click `Install`. We will be using their automatic install script for convenience. 

`bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en`

![](/img/waf-install.png)

2. Follow On-Screen Prompts to complete installation. You will be provided with an admin password and username as well as the URL to access Safeline, typically on port 9443.

![](/img/waf-install-2.png)

3. Access the dashboard and management console at `https://localhost:9443` and log-in.

## Onboarding the DVWA

Use the SafeLine WAF management console or interface to add a new application. In the Safeline dashboard go to "Applications" then "Add Application".

* Domain: `dvwa.local`

* Port: `80` `HTTP`

* Upstream: `<ubuntu-ip>:8080` 

![](/img/waf-add-dvwa.png)

This is just a test of the system so we don't need to worry about HTTPS right now. Delete the 443 port option. The port flow should be: User → Safeline (port 80) → DVWA (port 8080). So you want to use `dvwa.local:80` from kali to go through the WAF. If you use `dvwa.local:8080` you will bypass the WAF.

## SQL Injection

From your Kali VM navigate to the DVWA. Navigate to SQL Injection and try a typical SQL injection string. The web application firewall we just configured should stop it.

`' OR '1'='1`

![](/img/sqli-blocked.png)
