# Project: Linux Server Configuration

This readme describes the process to deploy a [flask web application](https://github.com/andersp89/catalog-app) on a Ubuntu 16.04.2 LTS server at Amazon Lightsail. Please visit http://18.195.163.63/, to see the functioning app.

IP address: 18.195.163.63
Port: 2200

It is divided into two main parts. Firstly, the server configuration is described, and secondly an overview of all installed packages through installation is given. 

# Summary of server configuration
The below process outlines the steps needed to successfully deploy the Flask App on a Ubuntu server. Moreover, each step is described in the following.

1. Create a Ubuntu instance on Amazon Lightsail
2. Update all currently installed packages.
3. Create a new user ("grader") with sudo permissions
4. Configure SSH
5. Configure firewall (UFW)
6. Configure local Time Zone to UTC
7. Install and configure Apache HTTP server
8. Install Git and clone Flask App
9. Set up Postgresql
10. Configure OAuth for LinkedIn authentication


## 1. Create a Ubuntu instance on Amazon Lightsail
Follow the instructions at [Udacity](https://classroom.udacity.com/nanodegrees/nd004/parts/ab002e9a-b26c-43a4-8460-dc4c4b11c379/modules/357367901175462/lessons/3573679011239847/concepts/c4cbd3f2-9adb-45d4-8eaf-b5fc89cc606e), to create a virtual machine running Ubuntu (OS only) at Amazon Lightsail. Remember, to allow port number 2200 in firewall settings. Also, go to settings and download the key file (ending with .pem). Then, go to your terminal, cd to the folder of the .pem-file from Amazon and login with ssh:

```
YOUR LOCAL MACHINE:~$ ssh -i LightsailDefaultPrivateKey-eu-central-1.pem ubuntu@18.195.163.63
```

## 2. Update all currently installed packages.
Update all available packages: This will provide a list of packages to be upgraded.
```    
ubuntu@:~# sudo apt-get update  
``` 
Upgrade packages to newer versions:   
``` 
ubuntu@:~# sudo apt-get upgrade   
```

## 3. Create a new user ("grader") with sudo permissions
Add a new user called "grader".
```
ubuntu@:~# sudo adduser grader
```
When promted, set a password for the user. After the user has been created, give the user sudo rights, by running visudo:
```
ubuntu@:~# sudo visudo
```
Once your in the visudo file, add the following code directly below the "root" user following the same format:
```
grader  ALL=(ALL:ALL) ALL
```

## 4. Configure SSH
Configure SSh to use a non-default port, i.e. **_not_** port 22, deactivate root log-in and enable SSH key at login.

a) First, access the ssh config file:
```
ubuntu@:~# sudo nano /etc/ssh/sshd_config
```

b) Change ```Port 22``` to ```Port 2200```:
```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200
```

c) Disable root login:
```
PermitRootLogin no
```

d) Enable clear text password. Later we will disable clear text password, when the ssh-keygen is completed for grader:
```
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication yes
```

e) Restart SSH service for changes to take effect: 
```
ubuntu:~# sudo service ssh restart  
```  

f) Now, you can connect to Ubuntu server with new user
```
YOUR LOCAL MACHINE:~$ ssh grader@18.195.163.63 -p 2200
```

g) Now, create a ssh key pair for grader, to securely connect with SSH. First, switch back to your local machine and run:
```
YOUR LOCAL MACHINE:~$ ssh-keygen
```  
Now you are asked to give a filename for the key pair. You can change the **id_rsa** to whatever filename you want. You will be prompted to enter a password to protect the files. This password you will use at every login. When the command has been run, you will see that ssh-keygen has generated two files : file_name (private_key) and file_name.pub (public_key) file. The file file_name.pub will be placed on the server for authorization.

h) Switch to the remote server as `grader` and create a directory called `.ssh` in the root directory of grader and a new file in `.ssh` named `authorized_keys`, to store the public key in:
```
grader:~$ mkdir .ssh
grader:~$ touch .ssh/authorized_keys
```

i)  Switch back to your Local Machine, and copy the contents of your_file.pub:
```
YOUR LOCAL MACHINE:~$ sudo cat ~/.ssh/your_file.pub
```

j)  Switch back to virtual server, edit ```authorized_keys```-file and paste the content of ```your_file.pub``` inside. Save file.
```
grader:~$ sudo nano .ssh/authorized_keys
```

k)  Set specific file permission on SSH and ```authorized_keys``` directories:
```
grader:~$ chmod 700 .ssh
grader:~$ chmod 644 .ssh/authorized_keys
```

l) Disable clear text password, to use ssh key only, by running:
```
grader:~$ sudo nano /etc/ssh/sshd_config
```
In file, change `PasswordAuthentication` to `no`, like so:
```
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no
```
m) Restart SSH service for changes to take effect. 
```
grader:~$ sudo service ssh restart  
```

n) At next login for grader, use the private key, instead of the clear text passwrod, to login, like so:
```
YOUR LOCAL MACHINE:~$ ssh grader@18.195.163.63 -p 2200 -i path/to/your/private/key
```

## 5. Configure firewall (UFW)
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

a) First, check that UFW is inactive by running, note it should be **_inactive_** currently:
```
grader:~$ sudo ufw status
```  

b) Disallow all incoming connections except the ones needed, following the security rule of least privilege, run the following commands:

```
grader:~$ sudo ufw allow 2200/tcp
grader:~$ sudo ufw allow 80/tcp
grader:~$ sudo ufw allow 123/udp
grader:~$ sudo ufw default deny incoming
grader:~$ sudo ufw default allow outgoing
```

c) You can now enable the firewall. Note: make sure that the firewall is open for port 2200 in Lightspeed settings:
'''
grader:~$ sudo ufw enable
'''

Remark, the Lightsail instance will no longer be acccessible thorugh the web app 'Connect using SSH'-button, as firewall has restricted ssh to ports as specified above.

## 6. Configure local Time Zone to UTC
a) Open Timezone selection dialog, afterwhich a selection menu will open up. Choose **None of the above**,and then choose **UTC**.:  
```
grader:~$ sudo dpkg-reconfigure tzdata
```  

## 7. Install and configure Apache HTTP server
Install Apache as HTTP server and mod_wsgi to connect to python.

a) Install Apache web Server:   
```
grader:~$ sudo apt-get install apache2
```
If Apache was installed correctly, the default page will show when visiting the ip address: http://18.195.163.63/.

b) Install mod_wsgi to serve the Flask app with Apache:  
```
grader:~$ sudo apt-get install libapache2-mod-wsgi
```

c) Configure Apache to serve Flask App. First navigate to the ```wwww``` directory:  
```
grader:~$ cd /var/www
```  

d) Then, create a directory called `catalog` and within that make another directory called `app`. The `/var/www/catalog` will house our wsgi application which points to the server file, in my case it's name is: _app.py_. Follow these steps: 

```
grader:/var/www$ sudo mkdir catalog
grader:/var/www$ cd catalog
grader:/var/www$ sudo mkdir app
```

e) Now install dependancies for running my Flask app:
```
grader:~$ sudo apt-get install python-pip
grader:~$ sudo pip install Flask
grader:~$ sudo pip install Flask-SQLAlchemy
grader:~$ sudo pip install requests
grader:~$ sudo pip install psycopg2
```

d) Then, configure the virtual host file **_catalog.conf_**, to point mod_wsgi to the ```.wsgi```-file, that we will create next. First, create the `.conf` file:   
```
grader:/var/www/catalog/app$ sudo nano /etc/apache2/sites-available/catalog.conf
```  

Type in and save the following code in the file:

```
<VirtualHost *:80>
      ServerName 18.195.163.63
      ServerAdmin anders@ptrading.dk
      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/app/>
          Order allow,deny
          Allow from all
      </Directory>
      Alias /static /var/www/catalog/app/static
      <Directory /var/www/catalog/app/static/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
```

e) Now, create the `catalog.wsgi`-file, to point mod_wsgi to the right python file that holds the application logic. The file will be created in `/var/www/catalog` directory. Follow the steps: 
```
grader:/var/www/catalog/catalog$ cd /var/www/catalog
grader:/var/www/catalog$ sudo nano catalog.wsgi
```

Paste in the following code and save:
```
## /var/www/catalog/
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/app/")

from app import app as application
application.secret_key = 'Add your secret key'
```
_Please note that here I have mentioned `app` because that is the file which houses my application logic._

d) Now, enable the virtual host, for mod_wsgi to use the `catalog.wsgi`:  
```
grader:/var/www/catalog$ sudo a2ensite catalog.wsgi
```

## 8. Install Git and clone Flask App

Now, the server is ready to serve app.py, however, the application is still missing. 

a) First, we need to install git, to be able to clone the repository. Follow this step:
```
grader:~$ sudo apt-get install git
```

b) Clone the git repository:
```
grader:/var/www/catalog$ sudo git clone https://github.com/andersp89/catalog-app.git
```  

c) Move the application logic inside the newly cloned `catalog-app` to the `app` folder, and delete then the empty folder:
```
grader:/var/www/catalog$ mv catalog-app/* /var/catalog/app
grader:/var/www/catalog$ sudo rm -rf catalog-app
```

## 9. Set up Postgresql
Now we will install and configure **Postgresql**, to serve our database to store users, categories and items.

a) First, install `postgresql`:
```
grader:~$ sudo apt-get install postgresql
```

b) Create a new user in catalog and create a new database. First, get super user rights to postgres by typing `sudo su postgres` and then type `psql`, to get the postgresql command line. Then, follow these steps:
```
postgres=# CREATE USER catalog WITH PASSWORD 'password_here';
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog WITH OWNER catalog;
```

c) Revoke all rights on the database schema, and grant access to catalog only.
```
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
```

e) Exit Postgresql and postgres user:
```
postgres=# \q
postgres@ip-10-20-11-110~$ exit
```

d) Now, run the `database_setup.py` to create the database:
```
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```
_The basic syntax of this statement is:_
```
postgresql://username:password@host:port/database
```
## 10. Configure OAuth for LinkedIn authentication

a) Go to LinkedIn Developers at https://developer.linkedin.com/, press "My Apps", select your App, and add `http://18.195.163.63:80/licode` to the "Authorized Redirect URLs", to allow redirects from the URL.

That's it! Everything should now be set-up properly. You just need to restart the Apache server, to be able to run the application:
```
grader:~$ sudo apache2ctl restart
```
**Finsihed - enjoy using the newly deployed Flask app at your Ubuntu server or go to http://18.195.163.63!**

# Summary of software installed
Please find below the packages used during the deployment of https://github.com/andersp89/catalog-app to Amazon Lightspeed:

| Package Name    |Descripton     | 
| ----------------|:-------------:| 
| **apache2**     | HTTP Server |
| **libapache2-mod-wsgi** |	hosts Python applications on Apache2 server|
|**postgresql**|	Postgresql Database server|
|**git**|	Version control system tools|
|**python-requests**|	HTTP module|
|**sqlalchemy**|	ORM and SQL tools for Python|
|**flask**|	Microframework for web applications|
|**pip**|	Python package manager|
|**python-psycopg2**|	PostgreSQL adapter for Python|


# Bibliography
Please find below important sources in the development of this readme:
* https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps