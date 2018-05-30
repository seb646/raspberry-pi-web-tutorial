# Setup a Raspberry Pi Web Server
This guide will show you how to setup a [Raspberry Pi](https://www.raspberrypi.org/) web server on your home network! Here is a [list of materials](http://a.co/c5fqu6n) needed for this tutorial. 


## Step 1: Raspberry Pi Configuration
The first step in this tutorial is to flash Raspberry Pi's version of Linux onto an SD card. This is where all of our data will be kept, and where the hardware will look to for operating commands. In order to do this, you need to download an application called Etcher (Windows and MacOS). Once you've downloaded [Etcher](https://etcher.io/), you'll also need to download [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/).

The next step is to plug your SD card into your computer and open up Etcher. Select the Raspbian .img file your downloaded from the Raspberry Pi website as the image. For the drive, select the SD card you just plugged in. Then click "Flash!" and wait for the process to complete.

Once the image is flashed on the drive, safely eject the SD card and plug it into your Raspberry Pi. You will then need a micro-USB cord and an ethernet cable to power the device and connect it to the internet.

After you plug the ethernet cord into your router, open up Terminal and enter the following commands to connect to your Raspberry Pi: `ssh pi@pi.local 22`. The system should prompt you for a password, and the default password is "password".

The next thing we need to do is configure the operating system itself. In the console, type sudo raspi-config and the following menu should pop up:

![Raspberry Pi Config](http://seb646.com/assets/terminal.png)

This menu allows you to change the password of your Pi (recommended), as well as make other configuration changes. Go into "Advanced Options" and select "Expand Filesystem." Once this is done, the Pi may reboot itself. If it does, simply log back in using the method described earlier.

Now, we need to make sure that all of the software installed on Raspbian is up-to-date by typing the following commands into your console:

```
sudo apt-get update
sudo apt-get upgrade
```

## Step 2: Install Apache & PHP
To install the Apache web server and PHP, type `sudo apt-get install apache2 php5` into your terminal and click enter. It will list out the packages it intends to download and install, and then it will ask you if you'd like to download all of the packages. Type in "y" for yes and click enter.

Next, you'll want to change the ownership of the folders that contain your website's files. To do this, type:
```
sudo chown -R pi /var/www
sudo chgrp -R pi /var/www
```
To check if Apache is working, go to your favorite web browser and type in "[http://pi.local](http://pi.local)" (or the IP address of your pi, if you know it). It should display a basic page with the title "Apache2 Debian Default Page." For future reference, the page you see is located in `/var/www/html/index.htm. Note: this page is only accessible on your local wifi network. We will explain how to make it accessible on the web in steps 3 and 4.

## 3. Router Configuration
The next step is to find the local IP address that your router assigned to your Raspberry Pi. To do this, we're going to use an application called [Angry IP Scanner](http://angryip.org/). Download the application, open it, and click the start button. This application will scan IP addresses in the range 10.0.1.0-255 by default, but if your router uses another format for subnet (172.16.X.X or 192.168.X.X) simply change the range. After scanning, locate the device with the hostname "pi.local" and note it's IP address and MAC address.

Then, open up your router's configuration. For the purpose of this tutorial, we'll be using Apple's Airport Utility. Click on your router and select the edit button. Enter your password and then find the "Network" tab.

The first thing we're going to add is a DHCP reservation. This allows the router to assign a specific IP address to a device every time it connects to the network. Under the DHCP reservations section, click the "+" button and fill in the information noted from the pi.local entry in Angry IP Scanner.

Next, we're going to open up two ports to allow people to access your website from any location, not just your home network! To do this, click "+" under Port Settings. Both the public and private UDP and TCP ports for the first entry should be "80" (the default port for Apache). The private IP address is the Pi's IP (aka the IP you just made a DHCP reservation for).

To check if port forwarding is enabled, enter your home IP address in a device outside of your home network and see if the default Apache start page loads.

## Step 4: Setup Dynamic DNS (with Google Domains)
Once your website is accessible by the outside world, you're ready to setup a custom domain name! For this tutorial, we're using [Google Domains](https://domains.google.com/) because of their awesome Dynamic DNS feature. Most people are assigned an IP address from their internet service provider, and that IP can change from time to time. A Dynamic DNS service allows users to link a domain to a non-static IP address.

The .com TLD on Google Domains is around $12/year, and Dynamic DNS is included for free. After purchasing a domain, go to the "Configure DNS" tab of that domain's management panel. Scroll down to "Synthetic records" and in the dropdown change "Subdomain forward" to "Dynamic DNS." For the base url (http://yoururl.com), put "@" in the input area preceding your domain name and add the record. Then, expand information on the record by clicking the arrow beside where it says "Dynamic DNS." Then click "View credentials" and note the username and password in the grey text box.

Next, we're going to setup a command line file on the server to tell Google Domains what the current IP of your home network is. To do this, create a file in the root directory of your Raspberry Pi by typing the following command: `sudo nano ~/dns_update.sh`. In the file, copy/paste the following information:

```
wget https://USERNAME:PASSWORD@domains.google.com/nic/update?hostname=YOURDOMAIN.com -qO dns_update_results.txt
wget https://USERNAME:PASSWORD@domains.google.com/nic/update?hostname=www.YOURDOMAIN.com -qO- >> dns_update_results.txt
echo " Last run: `date`" >> dns_update_results.txt
```

Then, make sure the file is executable by typing in the following command: `chmod +x ~/dns_update.sh`.

Next, we need to setup a cronjob for the file. A cronjob is a way to schedule when to run an executable file. We're going to tell the Raspberry Pi to run the file every hour. First, type in crontab -e and add the following entry: `0 * * * * ~/dns_update.sh`. To save the file, type `Ctrl + X` and follow the prompt.

## Step 5: Setup a Virtual Host with Apache
Now that you have the domain pointing to your Raspberry Pi, it's time to tell Apache what to do with the custom domain. We're going to create what's known as a Virtual Host, which is basically a domain name used in place of an IP address.

First, we're going to create a configuration file for the domain. In your console, type: `sudo nano /etc/apache2/sites-available/YOURDOMAIN.conf`. In the file, add the following lines:

```
<VirtualHost *:80>
    ServerName www.YOURDOMAIN.com
    ServerAlias YOURDOMAIN.com *.YOURDOMAIN.com
    DocumentRoot /var/www/YOURDOMAIN
</VirtualHost>
```
Save the file by typing Ctrl + X and following the prompt. Then, create the folder for the website's root: `sudo mkdir /var/www/YOURDOMAIN`.

And, again, we'll need to change the permissions for these folders so you can edit files inside them:

```
sudo chown -R pi /var/www/YOURDOMAIN 
sudo chgrp -R pi /var/www/YOURDOMAIN
```

For the last step, we need to activate the virtual host entry and then restart Apache's services:

```
sudo a2ensite YOURDOMAIN 
sudo service apache2 reload
```

## Finished!
Congratulations! You now have a custom domain and working website! To upload files to the server, download a File-Transfer Protocol (FTP) client like [Cyberduck](https://cyberduck.io/). Enter `YOURDOMAIN` as the server, `pi` as the username, `password` (or whatever you changed the password to) as the password, and use SFTP port 22. 
