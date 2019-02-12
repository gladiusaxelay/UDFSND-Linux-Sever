# UDFSND-Linux-Server-Hosting

The final project of the UDFSND requires setting up a Linux server with sound security settings, a running web server, and deployment of the Item Catalog [project](https://github.com/gladiusaxelay/UDFSND-Item-Catalog).

Student is expected to take a baseline installation of a Linux distribution on a virtual machine and prepare it to host the web app, install updates, securing it from a number of attack vectors and installing/configuring the web and database server.

## Getting Started

Select a cloud provider whereby you can deploy and configure a Linux web server. For this one, AWS Lightsail was used as instructed in the Nanodegree. Distribution is Ubuntu LTS 16.04 LTS.

### Prerequisites

Knowledge of:

* Python, HTML, CSS, PSQL.
* Git.
* Experience securing and configuring Linux web servers.

## Walkthrough:

### Begin by updating the distribution:

```
sudo apt-get update
sudo apt-get upgrade
```

Setup unattended-upgraades for the server:

```
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

### Create a new user for the Udacity grader and grant it the right permissions:

```
sudo adduser grader
sudo nano /etc/sudoers.d/grader
```

In the above file, place the following to set the permissions (save upon exit):

```
"grader ALL=(ALL:ALL) ALL"
```

In case you get the message "sudo: unable to resolve host", open the hosts file:

```
sudo nano /etc/hosts
```
and add 

```
127.0.0.1 ip-aaa-bbb-ccc-ddd
```

Substitute a b c d for the private ip address of your cloud server.

### Generate the authentication key for the user 'grader':

On your *local* computer, use the ssh-keygen tool:

```
ssh-keygen -f ~/.ssh/grader_keys
```

Copy the contents of the grader_keys.pub file to your cloud server in the following path:

```
touch /home/grader/.ssh/authorized_keys
```

Change the permissions to:

```
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
```

And change the owner of these files to grader:

```
 sudo chown -R grader:grader /home/grader/.ssh
```

Your should be able to log in to the cloud server from your local machine like so:

```
ssh -i ~/.ssh/udacity_key grader@3.8.124.12
```

After login, change the default SSH port from 22 to 2200 (change the reference of 22 to 2200 in this file):

```
sudo nano /etc/ssh/sshd_config
sudo service ssh restart
```

Lastly, disable the login as root (could be disabled by default, but do check);

```
sudo nano /etc/ssh/sshd_config
```

PermitRootLogin should be set to *no*.

```
sudo service ssh restart
```

### Firewall configuration (UFW):

```
sudo ufw status verbose
```
Allow SSH over port 2200:
```
sudo ufw allow 2200/tcp
```
Allow HTTP:
```
sudo ufw allow 80/tcp
```
Allow NTP:
```
sudo ufw allow 123/udp
```
Enable the firewall:
```
ufw enable
```

### Configuring the web server:

Install the web server, git (cloning the repository too) and WSGI:

```
sudo apt-get install apache2
sudo apt-get install python-setuptools libapache2-mod-wsgi
echo "ServerName HOSTNAME" | sudo tee /etc/apache2/conf-available/fqdn.conf
sudo a2enconf fqdn
sudo apt-get install git
git clone https://github.com/gladiusaxelay/UDFSND-Item-Catalog.git
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
```

Let's now create the web app folder structure:

(a good explanation of the following steps can be found [here](http://flask.pocoo.org/docs/1.0/deploying/mod_wsgi/))

```
cd /var/www
sudo mkdir catalog
cd catalog
sudo mkdir catalog
cd catalog
sudo mkdir static templates
```

Head back to:
```
cd /var/www/catalog
```

Install virtualenv:

```
sudo virtualenv venv
sudo chmod -R 777 venv
source venv/bin/activate
```

Install the packages referenced in requirements.txt.

Create an Apache configuration file for your application.

```
sudo nano /etc/apache2/sites-available/catalog.conf
```

```
<VirtualHost *:80>
  ServerName {YOUR-PUBLIC-SERVER-IP}
  ServerAdmin admin@{YOUR-PUBLIC-SERVER-IP}
  ServerAlias HOSTNAME {YOUR-PUBLIC-SERVER-IP}.xip.io
  WSGIScriptAlias / /var/www/catalog/catalog.wsgi
  <Directory /var/www/catalog/catalog/>
      Order allow,deny
      Allow from all
  </Directory>
  Alias /static /var/www/catalog/catalog/static
  <Directory /var/www/catalog/catalog/static/>
      Order allow,deny
      Allow from all
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error.log
  LogLevel warn
  CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

```
sudo a2ensite catalog
```

To run your app we need a .wsgi file. This file contains the code mod_wsgi runs on startup to get the app object. The object called application in that file is then used as application.

```
cd /var/www/catalog
```
```
sudo nano catalog.wsgi
```

And place:

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")
  
from catalog import app as application
application.secret_key = 'YOUR_APP_SECRET_KEY'
```

Rename your application.py file to (cd to where it is located first):

```
mv application.py __init__.py
```

### Configuring web app DB:

Update the engine object in your db_setup.py and app.py (note: renamed to init now) to:

```
https://docs.sqlalchemy.org/en/latest/core/engines.html
```

Create a user named catalog:

```
sudo adduser catalog
```

Swap to it:

```
sudo su - postgres
```

Go to the psql prompt:

```
psql
```

Run:

```
CREATE DATABASE catalog WITH OWNER catalog;
```

Connect to the database catalog 

```
\c catalog
```

Revoke all rights:

```
REVOKE ALL ON SCHEMA public FROM public;
```

Grant only access to the catalog role:

```
GRANT ALL ON SCHEMA public TO catalog;
```

Exit:

```
\q
exit
```

Remember to modify the path of the clients_secrets.json file in your code where needed, i.e.:


```
/var/www/catalog/catalog/client_secrets.json
```

And update in your Google API console the authorized domains, origins and redirects, i.e.:

* Authorized Domain: xip.io
* Authorized js origins: http:/{YOUR-SERVER-PUBLIC-IP}.xip.io
* Authorized redirect URIs: http:/{YOUR-SERVER-PUBLIC-IP}.xip.io/{YOUR-REDIRECT-LANDING-PAGE}

restart the apache server and try to login to the web app:

```
sudo service apache2 restart
http://{YOUR-SERVER-PUBLIC-IP}.xip.io/
```

If you face issues, I found useful to retch the most recent 40 entries of the error.log file:

```
sudo tail -40 /var/log/apache2/error.log
```

## Acknowledgments

Hat tip to the following github community members and their READMEs:
* [Stueken](https://github.com/stueken/FSND-P5_Linux-Server-Configuration)
* [iliketomatoes](https://github.com/iliketomatoes/linux_server_configuration/tree/master)
