# Project: Linux Server Configuration

To deploy a flask web application to AWS Lightsail, at time of installation, the server is running Ubuntu 16.04.2 LTS. Please visit http://18.195.163.63/, to see the functioning app.

IP address: 18.195.163.63
Port: 2200

user can ssh into the system using this command :  
```shell
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

## 1. Start a new Ubuntu Linux server instance on Amazon Lightsail
Follow the instructions at [Udacity](https://classroom.udacity.com/nanodegrees/nd004/parts/ab002e9a-b26c-43a4-8460-dc4c4b11c379/modules/357367901175462/lessons/3573679011239847/concepts/c4cbd3f2-9adb-45d4-8eaf-b5fc89cc606e), to create a virtual machine running Ubuntu (OS only) at Amazon Lightsail. Remember, to allow port number 2200 in firewall settings. Now, go to your terminal, cd to the folder of the .pem-file from Lightspeed and log-in with ssh:

```
YOUR LOCAL MACHINE:~$ ssh -i LightsailDefaultPrivateKey-eu-central-1.pem ubuntu@18.195.163.63
```

## 2. Update all currently installed packages.
Update all available packages: This will provide a list of packages to be upgraded.
    
    ubuntu@:~# sudo apt-get update  
    
Upgrade packages to newer versions:   
    
    ubuntu@:~# sudo apt-get upgrade   

## 3. Create a new user ("grader") with sudo permissions
Add a new user called "grader".

```
    ubuntu@:~# sudo adduser grader
```
When promted, set a password for the user.

After the user has been created, give the user sudo rights:
```
    ubuntu@:~# sudo visudo
```
Once your in the visudo file, add the following code directly below the "root" user following the same format:
```
grader  ALL=(ALL:ALL) ALL
```

## 4. Configure SSH
Configure SSh to use a non-default port, i.e. _not_ port 22, deactivate root log-in and enable SSH key at login.

(a) First, access the ssh config file by:
```
    ubuntu@:~# sudo nano /etc/ssh/sshd_config
```

(b) Change 'Port 22' to Port 2200
```
# What ports, IPs and protocols we listen for
# Port 22
Port 2200
```

(c) Disable root login
```
PermitRootLogin no
```

(d) Enable clear text password. Later we will disable clear text password, when the ssh-keygen is completed for grader
```
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication yes
```

(e) Restart SSH service for changes to take effect.  
```
	ubuntu:~# sudo service ssh restart  
```  

(f) Now, you can connect to Ubuntu server with new user
```
	YOUR LOCAL MACHINE:~$ ssh grader@18.195.163.63 -p 2200
```

(g) Now, create a ssh key pair for grader, to securely connect with SSH. First, switch back to your local machine and run:
```
	YOUR LOCAL MACHINE:~$ ssh-keygen
```  
Now you are asked to give a filename for the key pair. You can change the **id_rsa** to whatever filename you want. You will be prompted to enter a password to protect the files. This password you will use at every login. When the command has been run, you will see that ssh-keygen has generated two files : file_name (private_key) and file_name.pub (public_key) file. The file file_name.pub will be placed on the server for authorization.

(h) Switch to the remote server as `grader` and create a directory called `.ssh` in the root directory of grader and a new file in `.ssh` named `authorized_keys`, to store the public key in:
```
    grader:~$ mkdir .ssh
    grader:~$ touch .ssh/authorized_keys
```

(i)  Switch back to your Local Machine, and copy the contents of your_file.pub:
```
    YOUR LOCAL MACHINE:~$ sudo cat ~/.ssh/your_file.pub
```

(j)  Switch back to virtual server, edit ```authorized_keys```-file and paste the content of ```your_file.pub``` inside. Save file.
```
    grader:~$ sudo nano .ssh/authorized_keys
```

(k)  Set specific file permission on SSH and ```authorized_keys``` directories:
```
    grader:~$ chmod 700 .ssh
    grader:~$ chmod 644 .ssh/authorized_keys
```

(l) Disable clear text password, to use ssh key only, by running:
```
    grader:~$ sudo nano /etc/ssh/sshd_config
```
In file, change `PasswordAuthentication` to `no`, like so:
```
# Change to no to disable tunnelled clear text passwords
PasswordAuthentication no
```
(m) Restart SSH service for changes to take effect. 
```
	grader:~$ sudo service ssh restart  
```

(n) At next login for grader, use the private key, instead of the clear text passwrod, to login, like so:
```
	YOUR LOCAL MACHINE:~$ ssh grader@18.195.163.63 -p 2200 -i path/to/your/private/key
```

## Configure the firewall (UFW)
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

(a) First, check the 




Warning: When changing the SSH port, make sure that the firewall is open for port 2200 first, so that you don't lock yourself out of the server. Review this video for details! When you change the SSH port, the Lightsail instance will no longer be accessible through the web app 'Connect using SSH' button. The button assumes the default port is being used. There are instructions on the same page for connecting from your terminal to the instance. Connect using those instructions and then follow the rest of the steps.
Sudo ufw allow 2200/tcp
Sudo ufw allow 80/tcp
Sudo ufw allow 123/udp
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable

sudo /etc/init.d/ssh restart
	

Connect via ssh locally:
1.	Go to the "SSH Keys" tab under your Lightsail Account page
2.	Select the Default option under your region and download the key pair file
•	Will be a .pem file, ex. LightsailDefaultPrivateKey-us-west-2.pem
3.	Open up your terminal and navigate to the directory where the above file is stored
4.	Run chmod 600 [fileName] at the command line to restrict file permission so only you can read it
5.	Run ssh -i [fileName] [username]@[Public IP] to establish the connection to Lightsail
•	Username and IP are available under the "Connect" tab on the Lightsail web dashboard for your resource
sudo ssh -i LightSailDefaultPrivateKey-eu-central-1.pem ubuntu@18.195.163.63 -p 2200

Give grader access.
In order for your project to be reviewed, the grader needs to be able to log in to your server.
6. Create a new user account named grader.
Password: 
Sudo adduser <new_user_name>
Cat etc/passwd


8. Create an SSH key pair for grader using the ssh-keygen tool.
ssh-keygen
/Users/anderspedersen/.ssh/<project_name>
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
# Password to access file: 
create authorized_keys file in <new_user>/.ssh/authorized_keys and copy contents of .pub.
PasswordAuthentication no
755
644
Log-in as new user:
Sudo ssh -i /Users/anderspedersen/.ssh/project5Udacity grader@18.195.163.63 -p 2200

Ændret to ting: psswordauthroentication no -> yes. Og pubkeyauthornetication yes -> no


Prepare to deploy your project.
9. Configure the local timezone to UTC.
Sudo dpkg-reconfigure tzdata
10. Install and configure Apache to serve a Python mod_wsgi application.
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

sqlalchemy fundet!

error log, current error:
problem is, that sqlalchemy cannot be found!
[Mon Mar 19 09:41:49.751320 2018] [wsgi:error] [pid 11072:tid 140687701903104] [client 37.4.255.144:64386]   File "/var/www/catalog/catalog.wsgi", line 7, in <module>
[Mon Mar 19 09:41:49.751345 2018] [wsgi:error] [pid 11072:tid 140687701903104] [client 37.4.255.144:64386]     from app import app as application
[Mon Mar 19 09:41:49.751351 2018] [wsgi:error] [pid 11072:tid 140687701903104] [client 37.4.255.144:64386]   File "/var/www/catalog/app/app.py", line 6, in <module>
[Mon Mar 19 09:41:49.751362 2018] [wsgi:error] [pid 11072:tid 140687701903104] [client 37.4.255.144:64386]     from sqlalchemy import create_engine, asc
[Mon Mar 19 09:41:49.751377 2018] [wsgi:error] [pid 11072:tid 140687701903104] [client 37.4.255.144:64386] ImportError: No module named sqlalchemy

To-dos
•	Comment out all code to test
•	Jeg har lavet: sudo nano /etc/apache2/sites-ailable/catalog.conf
•	https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps 
•	Follow another student’s path: 	
•	Short and concise: https://github.com/harushimo/linux-server-configuration
•	https://github.com/ghoshabhi/P5-Linux-Config 
OperationalError: (sqlite3.OperationalError) unable to open database file
19 10:50:40.601091

change oauth at linkedin. Address.

14. Set it up in your server so that it functions correctly when visiting your server’s IP address in a browser. Make sure that your .git directory is not publicly accessible via a browser!


Review this sectioN, to check for all requirements: https://review.udacity.com/#!/rubrics/7/view 

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
