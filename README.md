# Tutorial: How to Setup a Raspberry Pi Web Server
This guide will show you how to set up a [Raspberry Pi](https://www.raspberrypi.org/) web server on your home network! Here is a [list of materials](http://a.co/c5fqu6n) needed for this tutorial. 


## Step 1: Raspbian Installation & Configuration
The first step in this tutorial is to flash Raspberry Pi's version of Linux onto an SD card. You'll use this SD card to store your PI's operating system. The first thing you need is an application called [Etcher](https://etcher.io/) (Windows and MacOS). After downloading Etcher, you also need to download [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/).

The next step is to plug your SD card into your computer and open up Etcher. Select the .img file downloaded from the Raspberry Pi website as the image you want to flash. For the drive, select the SD card you just plugged in. Then click "Flash!" and wait for the process to complete.

Once you finish flashing the OS onto the SD card, safely eject the SD card and plug it into your Raspberry Pi. You will then need a micro-USB cord and an ethernet cable to power the device and connect it to the internet.

After you plug the ethernet cord into your router, open up your computer's Terminal application. Then enter the following commands to connect to your Raspberry Pi: `ssh pi@pi.local 22`. The system should prompt you for a password, which by default is "password."

The next thing you need to do is configure the operating system itself. In the console, type `sudo raspi-config` and the following menu should pop up:

![Raspberry Pi Config](http://seb646.com/assets/terminal.png)

This menu allows you to change some of your Pi's configurations. We recommend both changing your password and expanding the Filesystem (go into `"Advanced Options" ->  "Expand Filesystem"`). Once complete, reboot your Pi by typing `sudo reboot`.

Next, make sure that all of the software installed on Raspbian is up-to-date. Type the following commands into your console:

```
sudo apt-get update
sudo apt-get upgrade
```

## Step 2: Installation of Apache & PHP
The next step is to install both PHP and the Apache web server onto your Pi. In your terminal, type: `sudo apt-get install apache2 php5`. This command will list out the packages it intends to download, and then asks you to confirm. Type in "y" for yes and click enter.

Next, change the ownership of the folders that contain your website's files by typing: 
```
sudo chown -R pi /var/www
sudo chgrp -R pi /var/www
```

To check if Apache is working, go to your favorite web browser and type in "[http://pi.local](http://pi.local)" (or the IP address of your pi, found in Step 3). It should display a basic page with the title "Apache2 Debian Default Page." For future reference, the location of this page on your Pi is `/var/www/html/index.htm`. (Note: at this stage, this page is only accessible on your local wifi network. We will explain how to make it available to the public in Steps 3 and 4).

## Step 3: Router Configuration
The next step is to find your Pi's local IP address. To do this, download an application called [Angry IP Scanner](http://angryip.org/). This application will scan your router's LAN IP addresses in the range `10.0.1.0`  to ` 0.0.1.255` (if your router uses `172.16.X.X` or `192.168.X.X` for it's subnet you might need to adjust the range). After scanning, locate the entry with the hostname "pi.local" and note both it's IP address and MAC address.

Then, open up your router's configuration. This tutorial uses [Apple's Airport Utility](https://support.apple.com/kb/dl1547?locale=en_US) on an Apple router. Click on your router and select the edit button. 

Navigate to the "Network" tab and find the DHCP reservations. DHCP reservations essentially reserve a specific IP address in your router to a specific MAC address. This way whenever your Pi connects to the router, it will use the same local IP address. Under the DHCP reservations section, click the "+" button and fill in the information noted from the "pi.local" entry of your Angry IP Scanner result.

Then you need to open a port to allow people to access your website from outside your local network. Under "Port Settings," click the "+" button. Both the public and private UDP and TCP ports should be "80" (the default port for Apache). The private IP address you assigned the Pi in the DHCP reservation.

To check if port forwarding is enabled, enter your home IP address in a device outside of your local network. It should load the default Apache page. (This is how the internet _really_ works, domain names hide everything!) 

## Step 4: Setup a custom domain using Google Domains' Dynamic DNS
This step uses [Google Domains](https://domains.google.com) to purchase a custom domain name and use Google's Dynamic DNS service. A .com domain with Google Domains is around $12/month (USD).  

If you're like me and your IP address is at the mercy of your ISP, you need to use Google Domain's DynamicDNS service. We're going to create a file that your Pi will run hourly, telling Google Domains what the current IP address of your home network is. 

After purchasing a domain, go to the "Configure DNS" tab of the domain's management panel. Find the "Synthetic records" section and, in the drop-down menu button, change "Subdomain forward" to "Dynamic DNS." For the base URL (ex: http://YOURDOMAIN.com), put "@" in the input area preceding the base URL. Once you add the record, view more information about it by clicking the arrow beside the bold "Dynamic DNS" title. Then click "View credentials" and note the username and password in the grey text box.

Next, create an executable .sh file on the Pi to tell Google Domains the current IP address of your network. To create the file, type: `sudo nano ~/dns_update.sh`. In the file, copy/paste the following information, replacing `USERNAME` and `PASSWORD` with the credentials noted in the Google Domains Dynamic DNS record entry:

```
wget https://USERNAME:PASSWORD@domains.google.com/nic/update?hostname=YOURDOMAIN.com -qO dns_update_results.txt
wget https://USERNAME:PASSWORD@domains.google.com/nic/update?hostname=www.YOURDOMAIN.com -qO- >> dns_update_results.txt
echo " Last run: `date`" >> dns_update_results.txt
```

Then, make sure the file is executable by typing: `chmod +x ~/dns_update.sh`. To execute the file, type `./dns_update.sh`. Check the Dynamic DNS record entry in your Google Domains management panel. You should see the IP address of your home network. If it's still `0.0.0.0`, stop here and restart this step.

The next thing you need to do is setup a cronjob to execute the `dns_update.sh` file every hour.  Type `crontab -e` and add the following entry: `0 * * * * ~/dns_update.sh`. To save the file, type `Ctrl + X` and follow the prompt.

## Step 5: Setup a Virtual Host with Apache
Now that you have the domain pointing to your Raspberry Pi, you should create a virtual host record for your domain in Apache's configuration. 

To create and edit the configuration file, type: `sudo nano /etc/apache2/sites-available/YOURDOMAIN.conf`. 

In the file, add the following lines:

```
<VirtualHost *:80>
    ServerName www.YOURDOMAIN.com
    ServerAlias YOURDOMAIN.com *.YOURDOMAIN.com
    DocumentRoot /var/www/YOURDOMAIN
</VirtualHost>
```
Save the file by typing `Ctrl + X` and following the prompt. Then, create the folder for the website's root: `sudo mkdir /var/www/YOURDOMAIN`.

And, again, we'll need to change the permissions for these folders so your user can edit the files inside:

```
sudo chown -R pi /var/www/YOURDOMAIN 
sudo chgrp -R pi /var/www/YOURDOMAIN
```

Lastly, you need to register the virtual host entry with Apache and then restart Apache's services:

```
sudo a2ensite YOURDOMAIN 
sudo service apache2 reload
```

## Finished! 
Congratulations! You're now ready to upload your website on the internet! To upload files onto your Pi, download a File-Transfer Protocol (FTP) client (like [Cyberduck](https://cyberduck.io/)).

__Login Information:__

* Server Name: `YOURDOMAIN`
* Username: `pi`
* Password: `password` (or whatever you changed it to)
* Port: 22 (SFTP)


__Other Tutorials for the Pi:__

* [Raspberry Pi SSL Certificates using Let’s Encrypt](http://pimylifeup.com/raspberry-pi-ssl-lets-encrypt/)
* [OpenVPN Server Raspberry Pi with PiVPN](https://www.youtube.com/watch?v=WA7QTM9hovQ) (video)
* [Raspberry Pi Owncloud WD Red Diet Pi](https://www.youtube.com/watch?v=RNehg6AKCiM) (video)
