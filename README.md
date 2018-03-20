# Project: Linux Server Configuration

To deploy a flask web application to AWS Lightsail, at time of installation, the server is running Ubuntu 16.04.2 LTS. Please visit http://18.195.163.63/, to see the functioning app.

IP address: 18.195.163.63
Port: 2200

user can ssh into the system using this command :  
```
ssh grader@18.195.163.63 -p 2200  
```

Optional: review flask app at: [github](https://github.com/harushimo/fullstack-nanodegree-vm.git)


# Summary of server configuration
1. Setup Virtual Machine and SSH into the server.
2. A new system user grader was created with permission to sudo.
3. All cuurently installed packages were updated and upgraded.
4. CRON tasks added to update and upgrade installed packages.
5. Changed SSH Port from 22 to 2200 and configure SSH access.
6. Configured UFWto only allow incoming connections for SSH(Port:2200), HTTP(Port:80) and NTP(Port:123).
7. Configured local Time Zone to UTC.
8. Installed and configure Apache to serve a Python mod_wsgi application.
9. Installed Git and Setup Environment for delopying Flask Application.
10. Install and configure PostgreSQL with default settings to not allow remote connection.
11. Created a new user catalog, added user to PostgreSQL databse with limited permissions to catalog application database.
12. Get OAUTH-LOGIN by LinkedIn working.
13. Installed and Configured Fail2ban intrusion protection that bans suspicious IPs.
14. Installed Glances to view full system status.

!! Husk at ændre her, så det passer!

_Each step is described in the following._

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

c) Move the application logic inside the newly cloned `catalog-app` to `app`, and delete then the empty folder:
```
grader:/var/www/catalog$ mv catalog-app/* /var/catalog/app
grader:/var/www/catalog$ sudo rm -rf catalog-app
```

## 9. Set up Postgresql
Now we will install and configure **Postgresql**, to serve our database to store users, categories and items.

a) First, install `postgresql`:
```
sudo apt-get install postgresql
```

b) 

a) Change the default user to postgres by typing : ```sudo su postgres``` and then type in ```psql```

```
postgres=# CREATE USER catalog WITH PASSWORD 'catalog';
postgres=# ALTER USER catalog CREATEDB;
postgres=# CREATE DATABASE catalog WITH OWNER catalog;
```

Revoke all rights on the database schema, and grant access to catalog only.
```
catalog=# REVOKE ALL ON SCHEMA public FROM public;
catalog=# GRANT ALL ON SCHEMA public TO catalog;
```

Exit Postgresql and postgres user:
```
postgres=# \q
postgres@ip-10-20-11-110~$ exit
```

Now we should run the `database_setup.py` to create the database and `lotsofmenus.py` to populate the database initially. Note that inside these files your create engine should point to the new databse now :   
```
engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
```

The basic syntax of this statement is:   

```
postgresql://username:password@host:port/database
```

 We will also create a user - **catalog** with previleges to create database only which will be password protected. Follow the steps:

# J. Configure Oauth credentials for 3rd party authentication 

MANGLER!

Restart Apache: `sudo apache2ctl restart`


**Finsihed - enjoy using the deployed Flask web app at your Ubuntu server!**

sudo apt-get install apache2
Visit apache server http://18.195.163.63:80 
sudo apt-get install libapache2-mod-wsgi
editing the /etc/apache2/sites-enabled/000-default.conf file. add the following line at the end of the <VirtualHost *:80> block, right before the closing </VirtualHost> line: WSGIScriptAlias / /var/www/html/myapp.wsgi
Restart the apache server! sudo apache2ctl restart 

You just defined the name of the file you need to write within your Apache configuration by using the WSGIScriptAlias directive. Despite having the extension .wsgi, these are just Python applications. Create the /var/www/html/myapp.wsgi file using the command sudo nano /var/www/html/myapp.wsgi
def application(environ, start_response):
    status = '200 OK'
    output = 'Hello Udacity!'

    response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
    start_response(status, response_headers)

    return [output]
This application will simply print return Hello Udacity! along with the required HTTP response headers. After saving this file you can reload http://localhost:8080 to see your application run in all its glory!

sudo nano /etc/apache2/sites-available/catalog.conf
 sudo apt-get install python-requests
sudo a2ensite catalog

Reload the apache2 server at every change!
Sudo service apache2 reload
Lav INGEN ændringer på ubuntu! Kun local. Og derefter push!

•	If you built your project with Python 3, you will need to install the Python 3 mod_wsgi package on your server: sudo apt-get install libapache2-mod-wsgi-py3.
11. Install and configure PostgreSQL:
sudo apt-get install postgresql
Since you are installing your web server and database server on the same machine, you do not need to modify your firewall settings. Your web server will communicate with the database via an internal mechanism that does not cross the boundaries of the firewall. If you were installing your database on a separate machine, you would need to modify the firewall settings on both the web server and the database server to permit these requests.

sudo apt-get install postgresql
password for db: 
sudo pip install psycopg2

Create new repo for edition of app, that works on ubuntu.
•	Do not allow remote connections
•	Create a new database user named catalog that has limited permissions to your catalog application database.
12. Install git.
sudo apt-get install git
Deploy the Item Catalog project.
13. Clone and setup your Item Catalog project from the Github repository you created earlier in this Nanodegree program.

git config --global user.name "Your Name"
git config --global user.email "youremail@domain.com"
https://www.digitalocean.com/community/tutorials/how-to-use-git-effectively#existing
Create work space environment:
mkdir -p /var/www/git/testing ; cd /var/www/git/testing

WSGIScriptAlias / /var/www/catalog-app/app.py
sudo apt-get install python-pip
sudo apt-get install python-virtualenv
Once you have virtualenv installed, just fire up a shell and create your own environment. I usually create a project folder and a venv folder within:
$ mkdir myproject
$ cd myproject
$ virtualenv venv
Now, whenever you want to work on a project, you only have to activate the corresponding environment. On OS X and Linux, do the following:
$ . venv/bin/activate
And if you want to go back to the real world, use the following command:
$ deactivate
Install flask in virtual env
pip install Flask
http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/ 

sudo apt-get install python-sqlalchemy

Jeg har lavet: sudo nano /etc/apache2/sites-ailable/catalog.conf
sqlalchemy fundet!

14. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser. Make sure that your .git directory is not publicly accessible via a browser!

change oauth at linkedin. Address.

just change the link in app.py from localhost => ip, also in linkedin oAouth settings.

# Summary of software installed
| Package Name    |Descripton     | 
| ----------------|:-------------:| 
| **apache2**     | HTTP Server |
| **libapache2-mod-wsgi** |	hosts Python applications on Apache2 server|
|**postgresql**|	Postgresql Database server|
|**git**|	Version control system tools|
|**python-requests**|	Postgresql Database server|
|**sqlalchemy**|	ORM and SQL tools for Python|
|**flask**|	Microframework for web applications|
|**pip**|	Postgresql Database server|
|**python-psycopg2**|	PostgreSQL adapter for Python|


## Bibliography
* https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps