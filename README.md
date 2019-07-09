# Linux Server Configuration
This project covers how to set up a remote Linux server, and use it to deploy a flask based web app.

## Addresses
### IP Address
* http://3.91.209.134  
* Port: 2200
### Address with Top Level Domain
* http://3.91.209.134.xip.io/

## Amazon Lightsail
This project uses Lightsail to host our Linux instance.  
1. Log in to your Amazon Web Services account, creating one if you lack it.
2. Create a instance, this particular configuration uses Ubuntu 16.04 and Lightsail's deploy OS only option.
3. Choose your payment plan, this one uses the free tier.
4. Name your instance/
5. Wait for startup.
6. You may now SSH into your server.

## Housekeeping on your New Server
Once connected to your server update all currently installed packages
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## Securing the New Server

Configure the Uncomplicated Firewall (UFW) to block the defaul port (2200) and only allow connections on 2200 (SSH), 80 (HTTP), or 123 (NTP)

```
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow 123/udp
sudo ufw deny 22
sudo ufw enable
```

Use the Amazon Lightsail GUI to match the instance firewall with the UFW.

Install fail2ban

```
sudo apt-get install fail2ban
sudo apt-get install sendmail
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

Set the `destemail` variable to your admin's email.

## Add a Grader Account

Create a new user name `grader`.
```
sudo adduser grader
```

Grand superuser permissions to `grader`
```
sudo visudo
```
Add 
```
grader ALL=(ALL:ALL) ALL
```
to this document, below the line
```
grader ALL=(ALL:ALL) ALL
```

Create and SSH keypair for `grader`.
User the ssh-keygen tool to create a `grader` key on the local machine.

```
cd ~/.ssh
ssh-keygen -f ~/.ssh/grader_key.rsa
grader
grader
cat ~/.ssh/grader_key.rsa.pub
```

Copy the code that appears on the terminal.

On your Lightsail instance, while logged in as root
```
mkdir /home/grader/.ssh/
nano /home/grader/.ssh/authorized_keys
```
Paste the key into this file.

On your Lightsail instance, while logged in as `grader`
```
sudo chmod 700 /home/grader/.ssh
sudo chmod 644 /home/grader/.ssh/authorized_keys
sudo chown -R grader:grader /home/grader/.ssh
sudo service ssh restart
```
`grader` can now login with
```
ssh -i ~/.ssh/grader_key.rsa -p 2200 grader@3.91.209.134
```

## Prepare the Server to Show the Catalog

Configure the timezone to UTC
```
sudo dpkg-reconfigure tzdata
```
Choose `None of the above` and `UTC`

Install and configure Apache

As grader run
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
sudo a2enmod wsgi
sudo service apache2 start
```
Install git
As grader run

```
sudo apt-get install git
```
## Deploy the Catalog

As grader run
```
sudo mkdir /var/www/catalog
cd /var/www/catalog
sudo git clone https://github.com/jbelcher86/nwo-app catalog
cd /var/www
sudo chown -R grader:grader catalog/
cd /var/www/catalog/catalog
mv app.py __init__.py
nano __init__.py
```

In `__init__.py ` replace 
```
app.debug = False
app.run(host='0.0.0.0', port=5000)
```
with
```
app.run()
```
`__init__.py `, `populate_db.py`, and `database.py` all contain the line
```
engine = create_engine('sqlite:///nwo.db',connect_args={'check_same_thread': False})
```
Replace this with
```
engine = create_engine('postgresql://nwo:nwo@localhost/nwo')
```

Edit the `catalog.wsgi` file to contain
```
activate_this = '/var/www/catalog/catalog/venv/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/catalog/")
sys.path.insert(1, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

Install the virtual environment and requirements
```
sudo apt-get install pip
sudo pip install virtualenv
cd /var/www/catalog/catalog
virtualenv -p python venv
sudo chown -R grader:grader venv/
. venv/bin/activate
pip install Flask
pip install httplib2
pip install requests
pip install --upgrade oauth2client
pip install sqlalchemy
sudo apt-get install libpq-dev
pip install pyopenssl
pip install pyscopg2
deactivate
```

Add the python path to `wsgi.conf`

Where it says `#WSGIPythonPath` add
```
/var/www/catalog/catalog/venv/lib/python2.7/site-packages
```
Create a `catalog.conf` file in `/etc/apache2/sites-available/catalog.conf` and add
```
<VirtualHost *:80>
    ServerName 3.91.209.34
    ServerAlias ec2-3-91-209-34.us-east-1.compute.amazonaws.com
    ServerAdmin admin@52.34.208.247
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

Enable the apache 2 virtual host
```
sudo a2ensite catalog
sudo service apache2 reload
```

## Use PostgreSQL for the Catalog DB

Run the following as `grader`

```
sudo apt-get install libpq-dev python-dev
sudo apt-get install postgresql postgresql-contrib
sudo su - postgres
psql
CREATE USER nwo WITH PASSWORD 'nwo';
ALTER USER nwo CREATEDB;
CREATE DATABASE catalog WITH OWNER nwo;
\c nwo
REVOKE ALL ON SCHEMA public FROM public;
GRANT ALL ON SCHEMA public TO nwo;
\q
exit
. venv/bin/activate
python /var/www/catalog/catalog/populate_db.py
deactivate
```
Edit `pg_hba.conf` to contain
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

Finally, restart Apache
```
sudo service apache2 restart
```
## Additional Resources
kcalata's [readme](https://github.com/kcalata/Linux-Server-Configuration) was super helpful for completing this project.
Further resources on adding sudo users can be found [here] (https://www.digitalocean.com/community/tutorials/how-to-create-a-sudo-user-on-ubuntu-quickstart)
And more info on configuring SSH keys [here] (https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server)
