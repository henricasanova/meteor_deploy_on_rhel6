# meteor_deploy_on_rhel6


Deploying a Meteor App on RHEL 6
========
---

Initial Setup
--------

So you god a brand new (virtual) machine running RHEL (Red Hat Enterprise Linux) 6. Let's
assume your machine has a fixed IP and a DNS entry _databet.ics.hawaii.edu_. 

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


####MongoDB

Based on the information [on this
site](http://tecadmin.net/install-mongodb-on-centos-rhel-and-fedora/),
first create the _/etc/yum/repos.d_ directory and a file __mongodb.repo_  in that
directory:
```
	sudo mkdir /etc/yum/repos.d
	sudo vi /etc/yum/repos.d/mongodb.repo
```
In the file, cut and paste this content and save:

```text
	[MongoDB]
	name=MongoDB Repository
	baseurl=http://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.2/x86_64/
	gpgcheck=0
	enabled=1
```
Once that's done, simply type:
```text
	sudo yum install mongodb-org
```
which will install MongoDB.  You can then start mongodb:
```text
	sudo service mongodb start
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


####NodeJS (0.10)

Based on the information [on this
site](https://nodejs.org/en/download/package-manager/), first run:
```text
	curl --silent --location https://rpm.nodesource.com/setup | sudo bash -	
```
and then install with:
```
	sudo yum install nodejs
```		
It will also be a good idea to install build tools to be abe to use npm stuff:
```
	sudo yum install gcc-c++ make
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
	meteor build --directory /home/john
```
This takes about 1 minute, and creates a directory called _/home/john/bundle/ that contains a file
called _main.js_, along with two directories (_programs_ and _server_) and two 
other files (_README_ and _star.json_). 

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

You should see the Meteor app server running in the terminal. (You can connect to it on the default Meteor port if launching a web browser from the VM, or perhaps directly if you asked ITS to open that port, which as we'll see shortly is not required). 

Now, this is very bare-bone, and you want to set some useful environment variables before
running the app: 
- PORT: This will be the port number the app is listening on
- MONGO_URL: _mongodb://localhost:27017_
- ROOT_URL: The curtom url of the app, e.g., _http://databet.ics.hawaii.edu/my_meteor_app_
- METEOR_SETTINGS: Set this to *the content* of the JSON file you are passing to meteor using the --settings flag when running in development mode
- UPLOAD_DIR: The path to the directory in which tomi:meteor-uploads will store files, in case you use that package

Clearly, you want to write a script to do the above.

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
_http://databet.ics.hawaii.edu_ and see some default Apache
Welcome page.

##### Making it so that Apache can connect to your Meteor app

On RHEL, the Security layer (Security-Enhanced Linux) will prohibit 
Apache to initiate an "outbound" connection, even to localhost! 
To disable this:
```text
/usr/sbin/setsebool -P httpd_can_network_connect 1
```
The "-P" above makes this setting permanent.  (Thanks a million to Justin
Ellison for your post on [SysAdmin's
Journey](http://sysadminsjourney.com/content/2010/02/01/apache-modproxy-error-13permission-denied-error-rhel/).


##### Making Apache know about the Meteor app


Let's use the setting in the previous section:
```
PORT=1234
ROOT_URL=http://databet.ics.hawaii.edu/my_meteor_app
```

We want to tell Apache that whenever a Web browser points to the $ROOT_URL above, content
should come from Meteor on port 1234.  This is accomplished by adding the following to the
Apache config file:
```
	ProxyPreserveHost On
	ProxyRequests     Off Order deny,allow Allow from all
	ProxyPass /my_meteory_app http://localhost:4000/my_meteory_app
	ProxyPassReverse /my_meteory_app http://localhost:4000/my_meteory_app
```
As root, add these 4 lines inside a <VirtualHost *:80> </VirtualHost> tag at the end of file
_/etc/httpd/conf/httpd.conf_. 

Start the Meteor app as in the previous section (with the PORT and METEOR_URL 
set). Then, in another shell, restart the apache service:
```
	sudo service httpd restart	
```


At this point, on your own machine, you should be able to start a Web browser, go
to URL http://databet.ics.hawaii.edu/my_meteor_app/  and see the Meteor app running!


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
Then, let's create a meteor group:
```text
	sudo groupadd meteor
```
and finally let's add our user to it:
```text
	usermod -a -G meteor my_meteor_app
```

#### Install npm forever

```text
sudo npm -g install forever
```

#### Create a /etc/init.d script

We need to create a new script in _/etc/init.d_ for our meteor app, so
essentially do what we did in the "Manually Running the Meteor App"
section. Most scripts in _/etc/init.d_ are pretty good examples of how to
do this.  This requires a preliminary steps:
- _sudo mkdir /etc/meteor/_:  In this directory will be the config file
  for our init.d script. Of course it doesn't have to be in _/etc_, but might
  as well. 
- create a config file in _/etc/meteor_. Let's call this
file _/etc/meteor/my_meteor_app_. Here is a config file that would work for
our example in previous sections:

```shell
PORT=1234
MONGO_URL=mongodb://localhost:27017
ROOT_URL=http://databet.ics.hawaii.edu/my_meteor_app
METEOR_SETTINGS=$(cat /etc/meteor/my_meteor_appsettings.real.json)
UPLOAD_DIR=/home/my_meteor_app/uploads
METEOR_MAINJS_PATH=/home/john/bundle/main.js
```

Note the last variable above, which simply says where your main.js is. 


At this point, we're ready for the _/etc/init.d/my_meteor_app_ script. Here is an example that should
require very little modification for your own app:
```text
XXXX
```

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

At this point, you should be able to always connect to your Meteor application.


