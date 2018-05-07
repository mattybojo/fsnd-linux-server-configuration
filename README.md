# Linux Server Configuration

Using a base Ubuntu image on Amazon Lightsail, I was tasked to prepare it to host a web application.  

## Server Information

- URL of hosted web application: http://ec2-34-217-195-185.us-west-2.compute.amazonaws.com/
- SSH port: 2200
- SSH connection command:

````
ssh -i [private_key_path] grader@34.217.195.185 -p 2200
````
where [private_key_path] is the path to the file containing the supplied private key of the grader user.

## Configuration Steps

### Login

##### Login as standard ubuntu user

````
ssh -i [private_key_path] ubuntu@34.217.195.185
````

### Create a new user named grader and grant the user sudo permissions

##### Create the user

````
sudo adduser grader
````

##### Grant the user sudo permissions

````
sudo nano /etc/sudoers.d/grader
# type in the following: "grader ALL=(ALL) ALL"
# save and exit
````

### Enable key-based login for grader

##### On your local machine, generate a ssh-keygen pair

````
ssh-keygen -f ~/.ssh/grader
````

##### Put the public key in the grader user's .ssh folder on the server

````
sudo mkdir /home/grader/.ssh
sudo nano /home/grader/.ssh/authorized_keys
# copy/paste the public key (grader.pub) into authorized_keys file
# save and exit
````

##### Set the correct owner/group on each of the files

````
sudo chown grader:grader /home/grader/.ssh
sudo chown grader:grader /home/grader/.ssh/authorized_keys
````

##### Log out of ubuntu user and log back in as the grader user

````
ssh -i [private_key_path] grader@34.217.195.185
````

### Update installed packages

````
sudo apt-get update
sudo apt-get upgrade
````

### Change SSH port, disable remote login of root user, and force key-based login

````
sudo nano /etc/ssh/sshd_config
# change 'Port 22' to 'Port 2200'
# set PermitRootLogin to no
# set PasswordAuthentication to no
# save and exit

# restart ssh service
sudo service ssh restart
````

### Configure and enable Uncomplicated Firewall (ufw)

````
# close all incoming ports
sudo ufw default deny incoming

# open all outgoing ports
sudo ufw default allow outgoing

# open ssh port
sudo ufw allow 2200/tcp

# open http port
sudo ufw allow 80/tcp

# open ntp port
sudo ufw allow 123/udp

# turn on firewall
sudo ufw enable
````

### Set server timezone to UTC

````
# run the program
sudo dpkg-reconfigure tzdata

# select 'Europe' then 'London'
````

### Install Apache, WSGI, and PostgreSQL

````
sudo apt-get install apache2 libapache2-mod-wsgi postgresql
````

### Configure postgres

````
# open the config, verify only local connections are specified
sudo nano /etc/postgresql/9.5/main/pg_hba.conf

sudo -u postgres -i
psql
postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
postgres=# CREATE DATABASE catalog;
postgres=# ALTER DATABASE catalog OWNER TO catalog;
postgres=# \q
postgres:~$ exit
````

### Install git and clone the web application project

````
# cd to the proper directory
cd /var/www

# install git, if needed
sudo apt-get install git

# clone the repo
sudo git clone https://github.com/mattybojo/fsnd-item-catalog.git

# ensure git folder is not accessible via web server
# add the following: RedirectMatch 404 /\.git
````

### Install python libraries required by Item Catalog

````
sudo apt-get install python-pip python-dev python-psycopg2
sudo pip install -r /var/www/fsnd-item-catalog/requirements.txt 
````

### Configure web application to use Postgres, not SQLite

````
sudo nano /var/www/fsnd-item-catalog/database_setup.py
# change the database link from 'sqlite:// ...' to 'postgresql://catalog:catalog@localhost/catalog', and save
````

### Run DB setup and seed

````
python /var/www/fsnd-item-catalog/database_setup.py
python /var/www/fsnd-item-catalog/bd_seed.py
````

### Configure apache to serve the WSGI application

````
sudo nano /var/www/fsnd-item-catalog/app.wsgi
Add the following lines to the file, and save the file.

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/fsnd-item-catalog/")
from application import app as application
application.secret_key = 'asecretkey'
````

##### Update the Apache configuration file to serve the web application with WSGI.

````
sudo nano /etc/apache2/sites-enabled/000-default.conf
Add the following line inside the <VirtualHost *:80> element, and save the file.

WSGIScriptAlias / /var/www/fsnd-item-catalog/app.wsgi
# save and exit
````

##### Restart Apache

````
sudo apache2ctl restart
````

### Test the application

Browse to [the server](http://ec2-34-217-195-185.us-west-2.compute.amazonaws.com/) to test if application is displaying data.

### Fix Google and Facebook logins

##### Google

In the Google developers console:
- Update the Authorized Javascript Origins URI to include 'http://ec2-34-217-195-185.us-west-2.compute.amazonaws.com/'
- Update Authorized redirect URI in the same way
- Download a new client_secrets file and put it in the root of the application

##### Facebook

In the Facebook developers console:
- Include the [web application](http://ec2-34-217-195-185.us-west-2.compute.amazonaws.com/) under ValidOAuth redirect URIs

##### Use absolute path to client_secrets

````
sudo nano application.py 
# prepend '/var/www/fsnd-item-catalog/' to 'client_secrets'
# do same with fb_clients_secrets.json
````

##### Restart apache service

````
sudo service apache2 restart
````

Update the Valid OAuth redirect URIs, in the Facebook developers console for the web application, to include the web application URL http://ec2-52-42-38-177.us-west-2.compute.amazonaws.com.

### Third party references

[PosgreSQL Docs](https://www.postgresql.org/docs/9.5/static/index.html)

[Apache Docs](https://httpd.apache.org/docs/2.4/)

[Git Help - Serverfault](https://serverfault.com/questions/128069/how-do-i-prevent-apache-from-serving-the-git-directory?utm_medium=organic&utm_source=google_rich_qa&utm_campaign=google_rich_qa)

[Postgres Help - Codementor](https://www.codementor.io/engineerapart/getting-started-with-postgresql-on-mac-osx-are8jcopb)

[Digital Ocean - Securing PostgreSQL on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)

[Stack Overflow](http://www.stackoverflow.com)
