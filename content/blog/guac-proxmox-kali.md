---
title: Addressing Kali Crashes - Guacamole for Remote Desktop with Proxmox VMs
date: 2025-05-02
tags:
  - homeprod
  - proxmox
  - kali
  - guacamole
---

{% imagesmall '/img/2025-05-03_10-48.png', '' %}

In this post I describe why I installed Apache Guacamole (non-Docker) to connect to a Kali VM and how I did so. I envision this post as part of a series in which I migrate from using Vagrant with VMWare workstation to Proxmox with OpenTofu and Ansible.

<br>
</br>

Overview in Brief:

- {% linkprimary "What to do About Frequent Kali Crashes?", "https://christopherbauer.xyz/blog/guac-proxmox-kali/#what-to-do-about-frequent-kali-crashes" %}
- {% linkprimary "Solution Design", "https://christopherbauer.xyz/blog/guac-proxmox-kali/#solution-design" %}
- {% linkprimary "No Docker?", "https://christopherbauer.xyz/blog/guac-proxmox-kali/#no-docker" %}
- {% linkprimary "Install Guide", "https://christopherbauer.xyz/blog/guac-proxmox-kali/#guide-to-installing-apache-guacamole" %}

<br>
</br>

## What to do About Frequent Kali Crashes?

When I was working on my OSCP certification, I frequently encountered Kali crashes. For those unfamiliar, Kali crashes seem to be commonly accepted, as though they were a "fact of life" among Kali users. In my case, downloading a bunch of exploit code probably didn't help stability of the OS, but I'd noticed that even fresh installs of Kali were sometimes wonky. Maybe it was my IaC/VM install method, I'm not laying blame here. Bottom line was that I made sure I put in place an automated way to get a new copy of Kali up and running, typically through a VM. This was seared into my brain as table stakes for running Kali.

Up to now my solution to this combines Vagrant with VMWare Workstation on my primary machine. Of course installing is only half the problem. A fresh OS install still leaves you with a pile of work on configs and preference settings. What about my bespoke command notes and CLI history? What about my CLI preferences like vim keybinds, fzf search, or auto complete? What about config files for tools like Feroxbuster or my Neovim config to make code easier to read? There is a lot of provisioning required, as I'm sure readers know. I've started moving my provisioning methods over to Ansible.

To be clear, a discussion on the merits of whether to automate using Bash scripts or Ansible is outside the scope of this already long post. Of course I could use Bash scripts, and indeed that is how I got started provisioning after Vagrant builds when I first began using Vagrant. However, I never knew when something had broken with my scripts, they just marched on mindlessly. Sure there are ways to ensure scripts report errors and exit responsibly and much more besides that I haven't learned yet. However, with the popularity of Ansible as a provisioning solution, it turns out I've already encountered it a few times without really understanding it in other contexts. So, as an idempotent option it seemed like now was a time to start learning.

When I switched to using Ansible with Vagrant, I noticed some issues I hadn't seen before. Vagrant, in combination with VMWare and Ansible was starting to appear overburdened. For one thing, running the Kali VM on my primary machine took up more overhead on than I would have liked. That wasn't the fault of VMWare or Vagrant, I'd just rather run the VM on another machine. For another, it seemed that anytime I wanted to run Vagrant, I had to make sure all the extensions were properly updated, that the box selection was correct, and any manner of other minutiae. While this is common, what was unusual was my bespoke solution made no prompt when I started, to resolve these issues before running. This constellation of factors has had me wondering about running Kali on a Hypervisor I know well: Proxmox. Ultimately, I'd like to learn OpenTofu to provision Kali on Proxmox, but that is a topic for another post.

### Wait, You've Got More Constraints?

To resolve these issues, I wanted to outsource running Kali VM to my Proxmox server. The hitch with this plan was that Proxmox doesn't share VMWare's seamless remote desktop option. Instead, Proxmox's remote desktop options seem like, well lets call them the main cabin class of passengers on the Proxmox ship. When you fire up a VM in Workstation, there isn't even an option to configure a remote desktop, it is simply baked in, with copy and paste, and runs immediately.

There was another hitch with this plan though. Proxmox after all, is not without options here. The SPICE protocol that I've traditionally used for remote desktop to my Proxmox VMs hasn't been working on my nodes despite troubleshooting. VNC was, of course, an option as Proxmox's default that is automatically installed into new VMs. However, I found working with the noVNC solution janky. It's fine for initial install or to set up SSH. It hasn't been a true replacement for a remote desktop. I've always found something off about it, as though I'm working with a lag. Most importantly, it hasn't, to my knowledge, offered copy and paste. So whatever solution I devised had to implement something outside of Proxmox's own remote desktop options.

## Solution Design

I settled on setting up a standalone {% linkprimary "Apache Guacamole", "https://guacamole.apache.org/" %} server. For those unfamiliar, a Guacamole server (hereafter Guac) is unique as a remote desktop solution in that it utilizes HTML5 over a browser for a "clientless" connection architecture. You set up the Guac server as well as a VNC server on the target, and simply connect by opening a browser.

For my solution, I envisioned a simple design. I would create an independent Guac server on a Proxmox VM (early tests showed an LXC wouldn't work with a build method) that would serve as the connection relay. That would connect my local machine through any old browser to a standalone Kali VM, also provided by Proxmox. To resolve the frequent Kali crashes, I would investigate OpenTofu at a later date, as my Ansible scripts for provisioning are ready today.

Yes, this design meant three computers/VMs rather than if I'd hosted Guac on Kali itself. However, as I said, Kali crashes often and Guac isn't a simple install, so having a third machine to act as an independent Guac server seemed prudent to avoid re-provisioning Guac itself. One added benefit to this solution that I could see was that it'd allow me to later enroll family members' remote machines with a VPN for troubleshooting through the Guac server.

To build Guac, you'll install two primary parts: the server and what Apache calls the "client." I find this naming confusing given the traditional notion of a server and a remote client, versus a "web client" as they define here. I'll refer to them as the backend and web app frontend respectively. I built Guac-Server/backend from source. For the web app frontend I simply installed Tomcat9 and inserted the Apache-supplied .war file.

The only part of this build that was complex was dealing with x11 on Kali to create automatic start up of TigerVNC. But I'll get to that soon enough. Installing the primary components went along pretty smoothly, though it was a bit complex.

### No Docker?

I also dismissed the option of a Docker install method this time. I chose to avoid docker for a variety of reasons, some of which {% linkprimary "I've talked about before", "https://christopherbauer.xyz/blog/docker-trouble/" %}. In brief, when I create single-purpose servers like this, I usually reject Docker as I tend to think that I can tweak the server's settings to match the single piece of software I want to use to more tightly integrate the distro and the software. Moreover, some enterprise-level software tends to run better directly on a machine (I'm thinking of Graylog as an example that suggests not using it's containerized version for production). And last, I just find it easier to troubleshoot on a headless server if the software isn't also behind a container's isolation where there may or may not be the tools you need, and you may or may not be able to run standard Linux commands.

<br>
</br>

## Guide to Installing Apache Guacamole

All told, I installed or configured software on three separate computers/vms using the command line that included:

- An Ubuntu 24.04 VM headless server for Guac-server backend, the Guac web frontend, and a database
- A Kali 2025.1c VM running the VNC server {% linkprimary "TigerVNC", "https://tigervnc.org/" %}
- My local machine with a browser

For brevity in this post, such as it is, I won't cover creating VMs in proxmox. I used the instructions {% linkprimary "provided by Kali", "https://www.kali.org/docs/virtualization/install-proxmox-guest-vm/" %} to import a Kali pre-built VM. There were also a number of sources (some outdated) on the overall process that helped, including {% linkprimary "this Akamai post", "https://www.linode.com/docs/guides/installing-apache-guacamole-on-ubuntu-and-debian/" %} and the {% linkprimary "original documentation", "https://guacamole.apache.org/doc/gug/installing-guacamole.html#installing-guacamole-natively" %}.

<br>
</br>

### Installing the Server/Backend

Shout out to credit where it's due: I followed [the official instructions](https://guacamole.apache.org/doc/gug/installing-guacamole.html) with help from [this blog](https://medium.com/@anshumaansingh10jan/unlocking-remote-access-a-comprehensive-guide-to-installing-and-configuring-apache-guacamole-on-30a4fd227fcd).

First make sure your system is up to date.

```shell
sudo apt update && sudo apt upgrade -y
```

Guac can connect using VNC, RDP or offer an SSH connection to a terminal shown through the browser. I opted to install all the associated dependencies for these options, prior to installing, just to cover all eventualities.

```shell
sudo apt install build-essential libcairo2-dev libjpeg-turbo8-dev \
    libpng-dev libtool-bin libossp-uuid-dev libvncserver-dev \
    freerdp2-dev libssh2-1-dev libtelnet-dev libwebsockets-dev \
    libpulse-dev libvorbis-dev libwebp-dev libssl-dev \
    libpango1.0-dev libswscale-dev libavcodec-dev libavutil-dev \
    libavformat-dev
```

The official website is [here](https://guacamole.apache.org/releases/) with the latest releases. Download the release.

```shell
wget https://apache.org/dyn/closer.lua/guacamole/1.5.5/source/guacamole-server-1.5.5.tar.gz?action=download -O guacamole-server-1.5.5.tar.gz
```

Decompress the release.

```shell
sudo tar -xvf guacamole-server-1.5.5.tar.gz
```

Next you'll begin the build process.

```shell
cd guacamole-server-1.5.5/
```

```shell
sudo ./configure --with-init-dir=/etc/init.d --enable-allow-freerdp-snapshots
```

```shell
sudo make
```

```shell
sudo make install
```

Now that you've installed, stay in that directory for a moment longer. Create links and cache to the shared libraries of the Guac folder.

```shell
sudo ldconfig
```

Then ensure the Service File has been loaded, enabled, and started.

```shell
sudo systemctl daemon-reload
```

```shell
sudo systemctl enable guacd
```

```shell
sudo systemctl start guacd
```

```shell
sudo systemctl status guacd
```

In future steps you're going to need some directories, so make them now.

```shell
sudo mkdir -p /etc/guacamole/{extensions,lib}
```

<br>
</br>

### Installing the Webapp/Frontend

At the time of this writing, Tomcat10, the default available in Ubuntu 24.04, wouldn't work with Guacamole. To get Tomcat9 on Ubuntu 24.04, add the repo.

```shell
sudo add-apt-repository -y -s "deb http://archive.ubuntu.com/ubuntu/ jammy main universe"
```

Then install Tomcat9.

```shell
sudo apt update && sudo apt install tomcat9 tomcat9-admin tomcat9-common tomcat9-user -y
```

Now download the .war file from the Apache website.

```shell
wget https://apache.org/dyn/closer.lua/guacamole/1.5.5/binary/guacamole-1.5.5.war?action=download -O guacamole-1.5.5.war
```

Move the war file to the directory we created.

```shell
sudo mv guacamole-1.5.5.war /var/lib/tomcat9/webapps/guacamole.war
```

Restart Tomcat9.

```shell
sudo systemctl restart tomcat9 guacd
```

<br>
</br>

### Database for Authentication

While the Guac docs say the web UI is functional at this point, I wasn't able to login with the default credentials without a database connected up to Guac. I decided to use MariaDB because it's a drop in for MySQL and easily provisioned on Debian derivatives.

Install MariaDB.

```shell
sudo apt install mariadb-server -y
```

Secure the install. Make sure to create a root password.

```shell
sudo mysql_secure_installation
```

Before you move on to database creation, you'll need some additions for MariaDB. You'll need a connector provided by MySQL and a Apache-supplied driver to connect MariaDB to the Guacamole web app.

First download the connector from [this page](https://dev.mysql.com/downloads/connector/j/) (select platform _independent_ to find the tar file).

```shell
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-j-9.3.0.tar.gz -O mysql-connector-j-9.3.0.tar.gz
```

Then extract the tar file and copy it to the Guac lib config.

```shell
tar -xvf mysql-connector-j-9.3.0.tar.gz
```

```shell
sudo cp mysql-connector-java-9.3.0/mysql-connector-j-9.3.0.jar /etc/guacamole/lib/
```

You can get the JDBC auth driver from the Guac [releases page](https://guacamole.apache.org/releases/).

```shell
wget https://apache.org/dyn/closer.lua/guacamole/1.5.5/binary/guacamole-auth-jdbc-1.5.5.tar.gz?action=download -O guacamole-auth-jdbc-1.5.5.tar.gz
```

Then extract the tar file and copy it to the Guac extensions directory you created.

```shell
tar -xvf guacamole-auth-jdbc-1.5.5.tar.gz
```

```shell
sudo cp guacamole-auth-jdbc-1.5.5/mysql/guacamole-auth-jdbc-mysql-1.5.5.jar /etc/guacamole/extensions/
```

#### Create the Guac Database in MariaDB

Now we'll create a database for Guac with SQL syntax commands.

```shell
mysql -u root -p
```

Create a database.

```shell
CREATE DATABASE guacamole_db;
```

Create a new user with their password.

```shell
CREATE USER '<SOME_USER>'@'localhost' IDENTIFIED BY '<SOME_PASSWORD>';
```

Grant our new user privileges to the new database.

```shell
GRANT SELECT,INSERT,UPDATE,DELETE ON guacamole_db.* TO '<SOME_USER>'@'localhost';
```

Refresh to update app with the new privileges.

```shell
FLUSH PRIVILEGES;
```

Exit.

```shell
quit
```

#### Import Schema Files

Next add a schema to the database you created. Adding this schema file will fill out the tables in your new database with a format that Guac can use.

Change directories to the schema from the JDBC driver you downloaded earlier.

```shell
cd guacamole-auth-jdbc-1.5.5/mysql/schema
```

Ensure you use the root user in the `mysql` portion of the following command to import the schema into the Guac database.

```shell
cat *.sql | mysql -u root -p guacamole_db
```

Create the properties file for connections between MariaDB and Guac.

```shell
sudo vi /etc/guacamole/guacamole.properties
```

Copy the following into the file.

```shell
# MySQL properties
mysql-hostname: 127.0.0.1
mysql-port: 3306
mysql-database: guacamole_db
mysql-username: <SOME_USER>
mysql-password: <SOME PASSWORD>
```

Reload everything.

```shell
sudo systemctl restart tomcat9 guacd mysql
```

The web UI should already be up but now it can accept your credentials. Check the website at `http://<HOST_IP>:8080/guacamole` and login with `guacadmin:guacadmin`.

{% sidenotesimage "Once you sign in, you'll face a screen like the following.", "Guac main screen", '/img/2025-05-01_13-57.png' %}

{% sidenotesimage "Head to the upper right corner where there is a drop down for guacadmin and select settings. To change the guacadmin password, create a user, or delete users select the user tab.", "Users tab", '/img/2025-05-01_13-57_1.png' %}

{% sidenotesimage "Make a new user with all the permissions boxes ticked, sign out and then back in as the new user, and finally delete the guacadmin user (or at minimum change the gaucadmin password).", "User creation", '/img/2025-05-01_13-58.png' %}

<br>
</br>

### Installing TigerVNC on Kali

Next, you'll install a VNC server called TigerVNC on Kali which will connect to the Guac server and subsequently relay to your browser. I used instructions inspired by "We are going to use TigerVNC:" at [this page](https://www.kali.org/docs/general-use/guacamole-kali-in-browser/) to create this section and {% linkprimary "this GitHub gist.", "https://gist.github.com/mixalbl4-127/4f5ab6744be6f98555f3e29f1cfc7050" %}

On the Kali machine update Kali and get yourself a hot drink, it'll be a minute.

```shell
sudo apt update && sudo apt upgrade -y
```

Install the server.

```
sudo apt install -y tigervnc-standalone-server
```

Make a directory in your home called .vnc.

```shell
mkdir -p ~/.vnc/ && cd ~/.vnc
```

Place a file in it called `xstartup` with the following content. This will configure the vnc server.

```
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
startxfce4
```

It used to be that the last line of the file above ended in `&` to background the startx command. This no longer is necessary since Ubuntu 22.04.

Make sure that file is executable.

```shell
chmod +x ~/.vnc/xstartup
```

Run the following and pay attention to the output. Initiating TigerVNC manually in this way will ask you for a password. Keep that password handy, you'll need it later for the Guac web UI.

```shell
vncserver
```

It'll say something like `New Xtigervnc server '<SOME_HOSTNAME>:2 (<SOME_USER>)' on port 5902 for display :2.` Take note of the `:2` portion and port.

Note, if you have trouble with the command above, try setting the XDG_CONFIG_HOME environment variable to `/home/<YOUR_USERNAME>/.vnc` in your `.bashrc` (or whatever shell you run) file and run `source ~/.bashrc`.

At this point you have two options. If you just want to get TigerVNC up and running to make sure everything works, then run it manually. First kill the server from above with `vncserver -kill :*`. Then enter the following with the parameter to expose the port externally.

```
vncserver :2 -localhost no
```

Then you can skip ahead to the Guac connections section.

If you'd rather ensure that TigerVNC runs each time Kali boots, read on.

#### Ensuring TigerVNC Starts at Boot

Next, we'll set up a Service File to start TigerVNC at startup. Of course you can skip this step if you only plan to occasionally use Guac to connect to Kali, as you can manually start the TigerVNC server over SSH first and then connect through Guac.

For this step, I relied on [this Digital Ocean guide](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-vnc-on-ubuntu-16-04), specifically step 4.

Create a Service File at the following location and with the following name `/etc/systemd/system/vncserver@.service`. Populate that file with the following, and be sure to change "someuser" with your username. Also on the line `ExecStart` modify the geometry variable for your screen resolution.

```
[Unit]
Description=Start TigerVNC server at startup
After=syslog.target network.target

[Service]
Type=simple
User=someuser
Group=someuser
WorkingDirectory=/home/someuser

PIDFile=/home/someuser/.vnc/%H:590%i.pid
ExecStartPre=-/bin/sh -c "/usr/bin/vncserver -kill :%i > /dev/null 2>&1"
ExecStart=/usr/bin/vncserver -fg -depth 24 -geometry 1920x1080 -localhost no :%i
ExecStop=/usr/bin/vncserver -kill :%i

[Install]
WantedBy=multi-user.target
```

Reload the Service Files.

```shell
sudo systemctl daemon-reload
```

Now you'll start the service. You have to add the number you noted from running the `vncserver` command to the following standard systemctl commands in order to tell the Service File what display number the service should appear over. Start it using the number from the `vncserver` command above, e.g.:

```shell
sudo systemctl enable vncserver@2.service
```

Make sure any other VNC servers aren't operating.

```shell
vncserver -kill :*
```

Then start it.

```shell
sudo systemctl start vncserver@2
```

If you need a refresher, you can run the following to identify and ensure the port is external facing and open.

```
ss -tulpn | grep -E -i 'vnc|590'
```

Take note of the port, as you'll need it for the Guac web UI.

Finally, verify the service is running properly.

```shell
sudo systemctl status vncserver@2
```

#### Troubleshooting Service File Creation

I struggled a bit with this section, in part because I ran the `vncserver` command with sudo at one point prior to setting up the Service File. That caused a great deal of confusion as the server repeatedly attempted to find the TigerVNC config in the root directory.

To remedy, I moved the ~/.vnc, ~/.Xauthority, ~/.Xresources and/or ~/.config/tigervnc files and directories to a backup location (or give them a `.bak` extension). Then I uninstalled `tigervnc-standalone-server` and reinstalled. Finally, I started over from the {% linkprimary "section above.", "https://christopherbauer.xyz/blog/guac-proxmox-kali/#installing-tigervnc-on-kali" %}

<br>
</br>

### Creating a Connection in Guac

For this section credit is due to {% linkprimary "the official Kali docs on Guac.", "https://www.kali.org/docs/general-use/guacamole-kali-in-browser/" %}

{% sidenotesimage "To create a connection to Kali, sign into Guac at the address you used above.   Head to the settings in the drop-down at the upper right, and look for the connections tab.  Clicking on that tab will open a new pane with fields for the connection details.", "Connections tab", '/img/2025-05-03_10-52.png' %}

{% sidenotesimage "The most important section is 'PARAMETERS.'  Look for the Network sub-section.  Enter your Kali IP or hostname and the port you noted from above in that section.", "Connections editing pane", '/img/2025-05-03_10-55.png' %}

{% sidenotesimage "Then for the Authentication sub-section, enter the user in Kali that started the VNC server and the password you created.  Finally, under the Display sub-section go to the line for Color depth and select True color (32-bit).", "Connections editing pane", '/img/2025-05-03_10-44.png' %}

With that, you should be able to connect. Head back up to the hamburger in the top right and select "Home" from the drop down. Now you should see a new clickable entry with your title for Kali. Simply click and you should begin the connection process. With any luck that will successfully connect and you're in business!

## Conclusion

Installing Guacamole is a bit of an investment in your time, especially when you consider that Proxmox's SPICE option might work well for you. For my circumstances however, it was worth it to build and write up this guide if only to build some familiarity with Guacamole.

If you'd like to connect, feel free to reach out on {% linkprimary "Mastodon", "https://infosec.exchange/@anthro_packets" %}.
