# meteor_deploy_on_rhel6


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

So you god a brand new (virtual) machine running RHEL (Red Hat Enterprise Linux) 6. Let's
assume your machine has a fixed IP and a one or more DNS 
entry (e.g., _databet.ics.hawaii.edu_). 

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

You should see the Meteor app server running in the terminal. (You can connect to it on the default Meteor port if launching a web browser from the VM, or perhaps directly if you asked ITS to open that port, which as we'll see shortly is not required).  

Now, this is very bare-bone, and you want to set some useful environment variables before
running the app. In fact, your app may fail for this reason.  For instance, for DataBET, here are some of the useful variables:

- PORT: This will be the port number the app is listening on
- MONGO_URL: _mongodb://localhost:27017_
- ROOT_URL: The custom url of the app, e.g., _http://databet.ics.hawaii.edu/my_meteor_app_
- METEOR_SETTINGS: Set this to *the content* of the JSON file you are passing to meteor using the --settings flag when running in development mode
- UPLOAD_DIR: The path to the directory in which uploaded files will be stored

Clearly, you want to write a script to do the above. And in fact, we'll see below that you'll put all that junk
in pre-defined scripts so as to enable your app as an official service.

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
If SELinux is not enabled on your system, you can skip this. Othersilde,
To enable Apache to connect to a local application fo:
```text
sudo /usr/sbin/setsebool -P httpd_can_network_connect 1
```
The "-P" above makes this setting permanent.  (Thanks to Justin
Ellison for his post on [SysAdmin's
Journey](http://sysadminsjourney.com/content/2010/02/01/apache-modproxy-error-13permission-denied-error-rhel/).


##### Making Apache know about the Meteor app


Let's use the setting in the previous section:
```
PORT=1234
ROOT_URL=http://databet.ics.hawaii.edu/my_meteor_app
```

We want to tell Apache that whenever a Web browser points to the $ROOT_URL above, content
should come from Meteor on port 1234.  This is accomplished by adding the following to the
end of the Apache config file _/etc/httpd/conf/httpd.conf_:
```
<VirtualHost *:80>
	ProxyPreserveHost On
	ProxyRequests     Off Order deny,allow Allow from all
	ProxyPass /my_meteory_app http://localhost:1234/my_meteory_app
	ProxyPassReverse /my_meteory_app http://localhost:1234/my_meteory_app
</VirtualHost>
```

Start the Meteor app as in the previous section (with the PORT and METEOR_URL 
set). Then, in another shell, restart the apache service:
```
	sudo service httpd restart	
```


At this point, on your own machine, you should be able to start a Web browser, go
to URL _http://databet.ics.hawaii.edu/my_meteor_app/_  and see the Meteor app running!


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
	usermod -a -G my_meteor_app my_meteor_app
```

#### Install forever via nmp

Forever is a useful program to run Node applications as daemons:
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
file _/etc/meteor/my_meteor_app.conf_. Here is a config file that would work for
our example in previous sections, with additional paths useful for the init.d
script:

```shell
###################################################
## For the init.d script
###################################################
export METEOR_USER=my_meteor_app
export METEOR_GROUP=my_meteor_app
export METEOR_ROOT=/home/john/

###################################################
## For the Meteor APP
###################################################
export PORT=1234
export MONGO_URL=mongodb://localhost:27017
export ROOT_URL=http://databet.ics.hawaii.edu/divelog_manager
export METEOR_SETTINGS=$(cat $METEOR_ROOT/settings.real.json)
export UPLOAD_DIR=$METEOR_ROOT/file_uploads

```

The last 3 variables above provide path to the JSON settings file, the directory for
storing files uploaded via tomi:meteor-uploads, and the path to the main.js file in the 
bundle. The my_meteor_app user must have the appropriate read/write permissions to those
locations of course. 

At this point, we're ready for creating the _/etc/init.d/my_meteor_app_
script. Here is an example that should be adapted to your environment.  In 
particular, note that PIDFILE, LOCKFILE, and LOGFILE variables. These have to
point to locations that can be read/written by user my_meteor_app.

```text
!/bin/bash
#
# Startup script for a Meteor app
#
# chkconfig: 35 99 99
#      Run level: 35  (just like MongoDB)
#      Start order: 99 (late)
#      Stop order: 10 (early)
# description: My Meteor app

# Load the helpful functions
#
. /etc/rc.d/init.d/functions

# Load the options from the config file
#
CONFIGFILE="/etc/meteor/my_meteor_app.conf"
if [ -f $CONFIGFILE ]; then
         . $CONFIGFILE
else
   echo -n "Cannot find config file $CONFIGFILE" && failure && echo
   exit 1
fi

# Set the path to the useful files given the METEOR_ROOT
METEOR_MAINJS=$METEOR_ROOT/bundle/main.js
PIDDIR=$METEOR_ROOT/var/run
PIDFILE=$PIDDIR/my_meteor_app.pid
LOCKFILE=$METEOR_ROOT/var/lock/my_meteor_app
LOGDIR=$METEOR_ROOT/var/log
LOGFILE=$LOGDIR/my_meteor_app.log


#
# Function to check that user is root
##################################################
check() {
  [ "`id -u`" = 0 ] || exit 4
}

#
# Function to start the service
##################################################
start()
{

  check     # check that the user is root

  echo -n $"Starting Meteor app...    "

  # Check that no instance is currently running
  ####
  LIST=`su - $METEOR_USER -c "forever list" | grep $METEOR_MAINJS | wc -l`
  [ ! $LIST -eq 0 ] && echo -n "Already running!" && failure && echo && return 1

  # Create a PID dir if not there already
  ####
  [ ! -d $PIDDIR ] && install -d -m 0755 -o $METEOR_USER -g $METEOR_GROUP $PIDDIR

  # Create the LOG dir if not there already
  ####
  [ ! -d $LOGDIR ] && install -d -m 0755 -o $METEOR_USER -g $METEOR_GROUP $LOGDIR

  # Start the daemons using forever
  ###
  BASE_COMMAND="forever start -l $LOGFILE --pidFile $PIDFILE -a $METEOR_MAINJS"
  COMMAND="source $CONFIGFILE; $BASE_COMMAND"
  su - $METEOR_USER -c "$COMMAND" 1> /dev/null 2> /dev/null
  RETVAL=$?

  # Print status and return
  ###
  [ $RETVAL -eq 0 ] && touch $LOCKFILE && echo -n "Started!" && success
  [ ! $RETVAL -eq 0 ] && failure
  echo
  return $RETVAL
}

#
# Function to stop the service
######################################################
stop()
{

  check     # check that the user is root

  echo -n $"Stopping Meteor app...    "

  # Stopping the service
  ###
  su - $METEOR_USER -c "forever stop $METEOR_MAINJS"  1> /dev/null 2> /dev/null
  RETVAL=$?

  # Print status and return
  ###
  [ $RETVAL -eq 0 ] && rm -f $LOCKFILE && echo -n "Stopped!" && success
  [ ! $RETVAL -eq 0 ] && echo -n "Was not running!" && failure
  echo
  return $RETVAL
}

#
# Function to restart the service
#
restart () {
  stop
  start
}


RETVAL=0

case "$1" in
  start)
    start
    ;;
  stop)
    stop
    ;;
  restart|reload|force-reload)
    restart
    ;;
  condrestart)
    [ -f $LOCKFILE ] && restart || :
    ;;
  status)
    status meteor_divelog_manager
    RETVAL=$?
    ;;
  *)
    echo "Usage: $0 {start|stop|status|restart|reload|force-reload|condrestart}"
    RETVAL=1
esac

exit $RETVAL
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

---





































