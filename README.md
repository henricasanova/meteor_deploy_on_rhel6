
Deploying a Meteor App on RHEL 6
========
---

The Short Story
-------

- Install Apache, Mongo, Apache, Meteor
- Build a Meteor bundle of the application
- Configure Apache to know about the application
- Make the application a service

Software version numbers are based on the time this page has
been last updated. If newer versions are available, update this
page with whatever has changed.

---

Initial Setup
--------

So you got a brand new (virtual) machine running RHEL (Red Hat Enterprise Linux) 6. Let's
assume your machine has a fixed IP and a one or more DNS 
entry (e.g., _server1.ics.hawaii.edu_). 

The first thing to do is to ssh to the machine and change the root password:
```text
	su              (enter the old root passwd)
	passwd          (enter the old and new root password)
```
	

At this point, it's likely a good idea to create a user acount for yourself. Let's
say your username is _john_, then as root do:

```text
	adduser john
	passwd john
```

Still as root, you now want to add john to the list of sudoers so that you can install new softare:

```text
	visudo
```
This will open a terminal text editor (should be vim). In the file being edited, below the line:
```
	root        ALL=(ALL)       ALL
```
add a line:
```
	john        ALL=(ALL)       ALL
```
Save the changes, log out, and log back in using your account. You can now execute commands as the super user by prefixing them with _sudo_.  It's probably a good idea to add a .ssh/authorized_keys file with your public key so that you can connect to the VM passwordlessly.

---

Installing Useful Software
--------

####Apache

Easy-peasy:
```
	sudo yum install httpd
```			
You can then start
apache:
```
	sudo service httpd start
```
and to make sure it's running (listening on port 80):
```
	sudo netstat -tulpn | grep :80
```
(which should produce one line with "LISTEN" in it)


If the above works, then enable autostart:
```
	sudo chkconfig httpd on
```


####MongoDB 3.4

Based on the information [on this
site](https://docs.mongodb.com/manual/tutorial/install-mongodb-on-red-hat/),
first create a _/etc/yum.repos.d/mongodb-org-3.4.repo_  file:
```
	sudo vi /etc/yum.repos.d/mongodb-org-3.4.repo
```
In the file, cut and paste this content and save:

```text
[mongodb-org-3.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.4/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.4.asc
```
Once that's done, simply type:
```text
	sudo yum install -y mongodb-org
```
which will install MongoDB.  You can then start mongo:
```text
	sudo service mongod start
```
and to make sure it's running (listening on port 27017):
```text
	sudo netstat -tulpn | grep :27017
```
(which should produce one line with "LISTEN" in it)

If the above works, then set up autostart on boot:
```text
	sudo chkconfig mongod on
```

**Warning:** If you plan to run multiple Meteor applications
on your server, you'll have to run multiple Mongo instances
as well to avoid DB collisions!

####NodeJS (LTS v6.9.4)

Based on the information [on this
site](https://nodejs.org/en/download/package-manager/#enterprise-linux-and-fedora), first run:
```text
	curl --silent --location https://rpm.nodesource.com/setup_6.x | sudo bash -
```
and then install with:
```
	sudo yum -y install nodejs
```		
It will also be a good idea to install build tools to be abe to use npm stuff:
```
	sudo yum install gcc-c++ make
```

You can now check that it all works by printing out the node version:
```
	node --version
```

####Meteor

Easiest of all:
```text
	curl https://install.meteor.com/ | sudo sh
```

---

Building the Meteor Bundle
-----------

Let us assume the Meteor app is in a directory _/home/john/meteor_app_ (i.e., this directory has the directories named client, server, public, etc.). Typically, this directory would be checked out from some github. Building the bundle in /home/john/ is easy:
```
	cd /home/john/meteor_app/
	cd app/
	meteor build --directory /home/john
```
This takes about 1 minute, and creates a directory called _/home/john/bundle/ that contains a file
called _main.js_, along with two directories (_programs_ and _server_) and two 
other files (_README_ and _star.json_). 

__Warning:__ Errors could be generated above regarding Npm packages that
need to be installed manually. The error messages basically tell you
what command to type to fix this (meteor npm install -save ...).


The next step is to an an NMP install of the bundle, i.e., making NodeJS know about the Meteor app:

```
	cd /home/john/bundle/programs/server
	npm install
```

At this point, the Meteor app is ready to run as a NodeJS application. It's probably
a good idea to write a script to do the above.


---

Manually Running the Meteor App
------------

To test that the Meteor app is running, you can now do:
```
	cd /home/john/bundle
	node main.js
```

You should see the Meteor app server running in the terminal, provided your app finds what it expects in terms of config files and environment variables (PORT, MONGO_URL, ROOT_URL, METEOR_SETTINGS). You can connect to it on the default Meteor port if launching a web browser from the VM, or perhaps directly if you asked ITS to open that port, which as we'll see shortly is not required.  

We'll see later how to make all this into a persistent service, but first...

---

Setting up Apache for the Meteor App
---------

Making apache serve your Meteor app has several advantages, especially when
using the same machine to serve Web content as well as the Meteor app, or
to serve multiple Meteor apps. The setup is pretty straightforward.

##### Opening port 80

First you need to open port 80 via software firewall rules. You can do this
by hand easily. As root, edit file _/etc/sysconfig/iptables_ and add
a line
```
-A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
```
right before the line
```
-A INPUT -j REJECT --reject-with icmp-host-prohibited
```

Once that's done, restart the iptables service:
```
sudo service iptables restart
```

To check that this works, restart Apache:
```
sudo service httpd restart
```

Using a Web browser anywhere, you should now be able to go to
_http://server1.ics.hawaii.edu_ and see some default Apache
Welcome page.

##### Making it so that Apache can connect to your Meteor app

On RHEL, the Security layer (Security-Enhanced Linux) will prohibit 
Apache to initiate an "outbound" connection, even to localhost! 
If SELinux is not enabled on your system, you can skip this. Othersilde,
To enable Apache to connect to a local application fo:
```text
sudo /usr/sbin/setsebool -P httpd_can_network_connect 1
```
The "-P" above makes this setting permanent.  (Thanks to Justin
Ellison for his post on [SysAdmin's
Journey](http://sysadminsjourney.com/content/2010/02/01/apache-modproxy-error-13permission-denied-error-rhel/)).


##### Making Apache know about the Meteor app

First, let's **uncomment** the following line in _/etc/httpd/conf/httpd.conf_:

```
	NameVirtualHost *:80
```



Let's use the setting in the previous section os that the URL will look to
the user as:
```
http://server1.ics.hawaii.edu/my_meteor_app
```
and the application is in fact running at:
```
http://localhost:1234/my_meteor_app
```

This is accomplished by adding the following to (the
end of) the Apache config file _/etc/httpd/conf/httpd.conf_:
```
<VirtualHost *:80>
        ServerName server1.ics.hawaii.edu
        ProxyPreserveHost On
        ProxyRequests     Off Order deny,allow Allow from all

        <Location />
                ProxyPass http://localhost:1234/my_meteor_app
                ProxyPassReverse http://localhost:1234/my_meteor_app
        </Location>
</VirtualHost>
```

##### Serving multiple apps
If you're using the same host to serve multiple Meteor apps using different domains (in case
you have multiple DNS entries) and/or URL paths, then
you simply create multiple VirtualHost sections in http.conf and voila! Here is
an example that does:

- Serve http://localhost:1111 as server1.ics.hawaii.edu/
- Serve http://localhost:2222 as server1.ics.hawaii.edu/some_app
- Serve http://localhost:3333 as server2.ics.hawaii.edu/


```
<VirtualHost *:80>
        ServerName server1.ics.hawaii.edu
        ProxyPreserveHost On
        ProxyRequests     Off Order deny,allow Allow from all

        <Location /some_app>
                ProxyPass http://localhost:2222/divelog_manager
                ProxyPassReverse http://localhost:2222/divelog_manager
        </Location>

        <Location />
                ProxyPass http://localhost:1111/
                ProxyPassReverse http://localhost:1111/
        </Location>
</VirtualHost>

<VirtualHost *:80>
        ServerName server2.ics.hawaii.edu
        ProxyPreserveHost On
        ProxyRequests     Off Order deny,allow Allow from all

        <Location />
                ProxyPass http://localhost:3333/
                ProxyPassReverse http://localhost:3333/
        </Location>
</VirtualHost>
```

Switching the order of the 2 Location clauses in the first VirtualHost
clause breaks things!


---

Making the Meteor App a Service
--------------

The way we ran the Meteor app above is not great because it requires the Shell
to keep running (this can't be dealt with with commands like _screen_). More importantly,
if the system reboot, we want our Meteor app to autostart.  In other words, we need
to make our Meteor app a service, just like Apache and Mongo.

<pre>
Annoyingly, ITS is still on RHEL6 even though RHEL7 has been out since June 2014. As a result,
one cannot use the latest and so-much-better systemd way of dealing with services.  This 
section of this file will one day replace with RHEL7 stuff, but for now, we're stuck with RHL6.
</pre>

#### Create a user for your service

We'll run the service as a particular user (not root), to avoid problems. So let's create a new user:
```text
	sudo adduser my_meteor_app
```	
Then, let's create a my_meteor_app group:
```text
	sudo groupadd my_meteor_app
```
and finally let's add our user to it:
```text
	sudo usermod -a -G my_meteor_app my_meteor_app
```

#### Install forever via nmp

Forever is a useful program to run Node applications as daemons:
```text
sudo npm -g install forever
```

#### Create a /etc/init.d script

We need to create a new script in /etc/init.d/my_meteor_app for our meteor app, to
essentially do what we did in the "Manually Running the Meteor App"
section. Most scripts in _/etc/init.d_ are pretty good examples of how to
do this.  See the deploy scripts for the DataBET application for 
a good template to use.


We can now check that the script works:
```
	sudo service my_meteor_app start
```
and to make sure it's running (listening on port 1234, as specified in our config file above):
```
	sudo netstat -tulpn | grep :1234
```
(which should produce one line with "LISTEN" in it)


If the above works, then enable autostart:
```
	sudo chkconfig my_meteor_app on
```

At this point, you should be able to always connect to your Meteor application, even after a reboot of the VM.

---





































