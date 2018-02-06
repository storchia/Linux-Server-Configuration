# Linux Server Configuration
Configuring a Linux server to host a web app securely using flask application on to AWS Lightsail. Installation of a Linux distribution on a virtual machine and prepare it to host web application(Item Catalog).

### Access Information
- IP Address *[35.170.117.90](http://35.170.117.90/)*
- Port 2200
- URL http://ec2-35-170-117-90.compute-1.amazonaws.com

### Server Update
```
sudo apt-get update 
sudo apt-get upgrade
```

### Add GRADER user and give him ROOT permissions
`sudo adduser grader`
`sudo touch /etc/sudoers.d/grader`
`sudo nano /etc/sudoers.d/grader` and add  the following line `grader ALL=(ALL:ALL) ALL`

### Edit file `/etc/ssh/sshd_config` 
```
Port 2200
PermitRootLogin no
PasswordAuthentication yes
service sshd restart
```
Make sure you add Firewall rule for port 2200 on Networking tab on AWS page.

### Configure key-based authentication for grader user
```
mkdir /home/grader/.ssh
chown grader:grader /home/grader/.ssh
chmod 700 /home/grader/.ssh
cp /root/.ssh/authorized_keys /home/grader/.ssh/
chown grader:grader /home/grader/.ssh/authorized_keys
chmod 644 /home/grader/.ssh/authorized_keys
```
On your Computer:
```
ssh-keygen -f ssh_keys/grader
cat ssh_keys/grader.pub | ssh grader@35.170.117.90 -p 2200 "cat >>  ~/.ssh/authorized_keys"
```

### Disable login with password in file /etc/ssh/sshd_config as requested on Project
`PasswordAuthentication no`
`sudo service ssh restart`
Try to ssh from your computer and no password will be requested.

### Configure timezone to UTC
`sudo dpkg-reconfigure tzdata`

### Configure Linux Firewall
```
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/udp
sudo ufw enable
```

### Install Apache
`sudo apt-get install apache2`

### Install mod_wsgi
```
sudo apt-get install libapache2-mod-wsgi python-dev
sudo a2enmod wsgi
sudo service apache2 start
```

### Get your Catalog app from Github
```
sudo apt-get install git
cd /var/www
sudo mkdir catalog
sudo chown -R grader:grader catalog
cd /catalog
git clone https://github.com/storchia/Catalog catalog
```
Create `catalog.wsgi` and put following code inside:
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey'
```

### Rename `catalog.py`
`mv catalog.py __init__.py`

### Install virtual environment
```
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo chmod -R 777 venv
```

### Install Packages
`sudo apt-get install python-pip`
`pip install Flask`
`sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils`

### Change path of client_secrets.json file
`sudo nano __init__.py`
Update `client_secrets.json` path to `/var/www/catalog/catalog/client_secrets.json`

### Configure and enable a virtual host
`sudo nano /etc/apache2/sites-available/catalog.conf`
Add following code to file:
```
<VirtualHost *:80>
    ServerName 35.170.117.90
    ServerAlias http://ec2-35-170-117-90.compute-1.amazonaws.com
    ServerAdmin admin@35.170.117.90
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
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
`sudo a2ensite catalog` to enable virtual host.

### Configure PostgreSQL
```
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
CREATE USER catalog WITH PASSWORD 'password';
ALTER USER catalog CREATEDB;
CREATE DATABASE catalog WITH OWNER catalog;
\c catalog
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO catalog;
\q
exit
```
Change create engine line in your `__init__.py` , `database_setup.py` and `lotsofitems.py` to: `engine = create_engine('postgresql://catalog:password@localhost/catalog')`

### Restart Apache
`sudo service apache2 restart`
