# Deploy to Amazon Lightsail


Download Private Key Contained in Submission
`ssh grader@52.14.163.100 -p 2200`



## Configuration Steps
The following steps were taken to configure and secure the server




#### *Set Up Non-Root User*
Create User grader

`sudo adduser grader`

`su grader`

Configure SSH Access

`cd /home/grader`

`mkdir .ssh`

`nano .ssh/authorized_keys`

Paste public key for grader

`exit`

Add grader To sudoers Group

`sudo -s`

`cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader`

`nano /etc/sudoers.d/grader`

Change the Following in File:

`ubuntu ALL=(ALL) NOPASSWD:ALL`

to:

`grader ALL=(ALL) NOPASSWD:ALL`

Hit CTRL-X and Then y to save

`exit`

Disable root SSH Access and Password Based Authentication

`sudo nano /etc/ssh/sshd_config`

Change the Following:

`Port 2200`

`PermitRootLogin no`




#### *Configure UFW and Amazon Lightsail Firewalls*

Open Port 2200 in Firewall Configuration on Amazon Lightsail Control Panel, Under Networking

`Custom      TCP     2200`

Configure UFW

`sudo ufw default deny incoming`

`sudo ufw default allow outgoing`

`sudo ufw allow ssh`

`sudo ufw allow 2200/tcp`

`sudo ufw allow www`

`sudo ufw allow ntp`

`sudo ufw enable`

`sudo ufw status`

On Checking Status One Should See The Following:

```Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere                          
2200/tcp                   ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
123                        ALLOW       Anywhere                  
22 (v6)                    ALLOW       Anywhere (v6)             
2200/tcp (v6)              ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
123 (v6)                   ALLOW       Anywhere (v6)  
```



#### *Install Software*

`sudo apt-get -qqy install make zip unzip postgresql git apache2 libapache2-mod-wsgi python3 python3-pip python python-pip`

`sudo pip2 install --upgrade pip`

`sudo pip2 install flask packaging oauth2client redis passlib flask-httpauth`

`sudo pip2 install sqlalchemy flask-sqlalchemy psycopg2 requests`




#### *Configure Postgresql*

`sudo nano /etc/postgresql/9.5/main/pg_hba.Configure`

Check to make sure outside connections are prevented

Configure Database

`sudo su postgres`

`psql`

`CREATE DATABASE catalog;`

`CREATE USER grader;`

`ALTER ROLE grader WITH PASSWORD 'grader';`

`GRANT ALL PRIVILEGES ON DATABASE catalog TO grader;`

`\q`

`exit`




#### *Set Up Application*

`cd /var/www`

`sudo mkdir FlaskApp`

`sudo chown grader FlaskApp`

`sudo chgrp grader FlaskApp`

`cd FlaskApp`

`git clone https://github.com/tpierag1/item-catalog.git FlaskApp`




#### *Modify Files*

`cd FlaskApp`

`mv application.py __init__.py`

In db_constructor.py comment out the following lines:
```
print "[\*]Deleting Old Databases...
        session.query(Category).delete()
        session.query(Item).delete()
        session.query(User).delete()"
```

Edit db_constructor.py, models.py and __init__.py:
```
    replace engine = create_engine('sqlite:///catalog.db')
    with engine= create_engine('postgresql://grader:grader@localhost/catalog')

```
Build Databases

`python db_constructor.py`

Change client_secrets.json location in __init__.py

`nano __init__.py`

Change the Following:

`CLIENT_ID = json.loads('client_secrets.json', 'r')`

to

`CLIENT_ID = json.loads('/var/www/FlaskApp/FlaskApp/client_secrets.json', 'r')`




#### *Write WSGI file and Configure apache2*

Create WSGI file

`cd ..`

`nano flaskapp.wsgi`

Add the following:

```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/FlaskApp/")

from FlaskApp import app as application
application.secret_key = 'super-secret-key'
```

Add Apache Configuration

`sudo nano /etc/apache2/sites-available/FlaskApp.conf`

Add the following:
```
<VirtualHost *:80>
    ServerName 52.14.163.100
    ServerAdmin admin@52.14.163.100
    WSGIDaemonProcess FlaskApp python-path=/var/www/FlaskApp
    WSGIProcessGroup FlaskApp
    WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
    <Directory /var/www/FlaskApp/FlaskApp/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/FlaskApp/static
    <Directory /var/www/catalog/FlaskApp/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```
Enable Site

`sudo a2ensite FlaskApp`

Create flaskapp.wsgi

`cd /var/www/FlaskApp`

`sudo nano flaskapp.wsgi`

Add the Following:

```
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/")

from FlaskApp import app as application
application.secret_key = 'super-secret-key'
```

Go to Google Dashboard console.developers.google.com
Download client-secrets.json and Tranfer to Server, Into /var/www/FlaskApp/FlaskApp

Restart Apache

`sudo service apache2 restart`




### *Configuration Details*

User

`username: grader`

`password: grader`

Database

`username: grader`

`password: grader`

URL

`http://52.14.163.100`
