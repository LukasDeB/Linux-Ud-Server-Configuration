# Linux Server Configuration Project

The goal is to set up a ubuntu vps instance and make the correct configuration changes to secure the server and
host a web application.

## Server Info
- IP: 35.182.247.34
- SSH port: 2200
- Web Application: http://35.182.247.34.xip.io/

## Updating the packages

After logging with
````
ssh -i ~\.ssh\lightsail_key.pem ubuntu@35.182.247.34
````
Run the following commands
```
sudo apt-get update
sudo apt-get upgrade
```

## Creating the grader user

To create an user named grader we run:
```
sudo adduser grader
```
(The password is ```linuxcourse``` , for later sudo usage)

We give it sudo access
```
sudo usermod -aG sudo grader
```
## Creating the SSH key pair for grader

On your local machine run
```
ssh-keygen
```
Save the key to any folder you prefer

Log in to the VPS as grader and create a folder named .ssh on the home directory
```
cd /home/grader/
mkdir .ssh
```
Then create the file authorized_keys within it
```
touch .ssh/authorized_keys
```
Copy the contents of the .pub file from your local machine created by the ```ssh-keygen``` command and paste it into the authorized_keys file in the remote server with nano
```
nano .ssh/authorized_keys
```
Now to finish up with setting the approriate file permissions to ensure better security
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```

## Changing the SSH port and denying remote root logins

Open the sshd_config file
```
sudo nano /etc/ssh/sshd_config
```
Where it says Port 20 change it to Port 2200 and to disable remote root login change ``` PermitRootLogin without-password``` to ```PermitRootLogin no```

Then restart the service
```
sudo service ssh restart
```

## Setting up the Uncomplicated Firewall

First we check the status of the ufw with
```
sudo ufw status
```
(It's status should be Inactive)

Then we deny all incoming connections and allow all outgoing connections with:
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
(Careful not to enable the firewall before allowing at least the ssh connection so you don't get locked out of your server)

Afterwards we allow all the wanted connections. In this case SSH(port 2200), HTTP(port 80) and NTP(port 123)
```
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
```
To wrap up enable the firewall.
```
sudo ufw enable
```
To make sure everything is as we wanted check it again with
```
sudo ufw status
```
## Changing the local timezone to UTC

If we want to set up the local timezone to UTC first run:
```
sudo dpkg-reconfigure tzdata
```
Select ```None of the above``` followed by ```UTC```

## Install and configure Apache + mod_wsgi 

Install apache by executing the following command
```
sudo apt-get install apache2
```
And then install mod_wsgi with
```
sudo apt-get install libapache2-mod-wsgi
```
Open the default config of apache responsible for handling requests
```
sudo nano /etc/apache2/sites-available/000-default.conf
```
Paste the following under ```ServerAdmin webmaster@localhost```

```
WSGIDaemonProcess u-catalog-project user=ubuntu group=ubuntu threads=5
WSGIScriptAlias / /var/www/project/u-catalog-project/project.wsgi

<Directory /var/www/project/u-catalog-project>
        WSGIProcessGroup u-catalog-project
        WSGIApplicationGroup %{GLOBAL}
        Require all granted
</Directory>
```
Save the file under the name catalog.conf then we select the new configuration with the command:
```
sudo a2ensite catalog.conf
```
Restart afterwards to apply all the changes
```
sudo service apache2 restart
```

(Apache will throw an error at this point because we haven't set up all the folders and the .wsgi file, that comes next)

## Deploy the Item Catalog Project

Navigate to /var/www
```
cd /var/www/
```
Create a folder named project and clone the item catalog project into it
```
sudo mkdir project
cd project
git clone https://github.com/LukasDeB/u-catalog-project
```
Create a file named project.wsgi in the u-catalog-project folder and paste the code below
```
#!/usr/bin/python

import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/project/u-catalog-project/")

from catalog_project import app as application
application.secret_key = 'super_extra__extra__secret_key'

```
Make the necessary changes to the catalog_project.py file in order to make it run (Changing the file paths)

```
import os

PROJECT_ROOT = os.path.realpath(os.path.dirname(__file__))
json_url = os.path.join(PROJECT_ROOT, 'client_secrets.json')
db_url = os.path.join(PROJECT_ROOT, 'catalogdb.db')
```
(Change the hardcoded urls and replace them with the variables)

Run the corresponding scripts inside the u-catalog-project to set up the Database and fill it with testing data if needed
```
cd /var/www/project/u-catalog-project
python db_setup.py
python dummydata.py
```
Install pip
```
sudo apt-get install python-pip
```
Install the dependencies needed for the projec to run with pip
```
pip install -r requirements.txt
```
## OAuth Changes

In the Google console set up xip.io as trusted domain and enter the web application url as authorized origins.

### Resources

- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
- http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/
- https://www.ssh.com/ssh/keygen/
- https://www.a2hosting.com/kb/getting-started-guide/accessing-your-account/disabling-ssh-logins-for-root
- https://askubuntu.com/questions/138423/how-do-i-change-my-timezone-to-utc-gmt
