# Udacity - Linux Server Configuration
This repo contains information about how an Ubuntu server was setup as a webserver with a web app deployed to it.

## Steps:
1. Provision an *AWS LightSail* Ubuntu instance. IP address = **52.90.97.136** and pem file **LightsailDefaultPrivateKey-us-east-1.pem**.
2. SSH onto the server:
```bash
ssh -i "/home/air-walk/workspaces/udacity-linux-server-configuration/LightsailDefaultPrivateKey-us-east-1.pem" ubuntu@52.90.97.136 -p 22
```
3. Update all packages:
```bash
sudo apt-get update && sudo apt-get -y upgrade
```
4. Update SSH port from 22 to 2200:
```bash
sudo vim /etc/ssh/sshd_config
# Update 'Port' from 22 to 2200 in this file

sudo service sshd restart
sudo netstat -tulpn | grep "sshd"
# You should now see the daemon listening on port 2200
exit
# Logout of the server
```
5. Finish changing SSH port:
    1. Open TCP ports `2200` and `123` and remove port `22` from the firewall settings in the LightSail console.
    2. Try logging onto the server on the new port:
```bash
ssh -i "/home/air-walk/workspaces/udacity-linux-server-configuration/LightsailDefaultPrivateKey-us-east-1.pem" ubuntu@52.90.97.136 -p 2200
```
6. Create user account `grader`:
```bash
sudo adduser grader
# pwd: <grader-password-here>
# Full name: Grader Ninja
```
7. Grant sudo permissions to `grader`:
```bash
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
sudo vim /etc/sudoers.d/grader
# Replace 'ubuntu' with 'grader' everywhere
```
8. Create SSH keypairs for `grader`:
```bash
# On your host machine, type:
ssh-keygen
# Enter filename as: /home/air-walk/.ssh/id_lightsail_ubuntu_rsa and passphrase: <private-passphrase-here>
```
9. Copy the content of following command to clipboard:
```bash
less /home/air-walk/.ssh/id_lightsail_ubuntu_rsa.pub
```
10. On the LightSail instance:
```bash
sudo su grader
mkdir .ssh
sudo chmod 700 .ssh/
vim .ssh/authorized_keys
# Add the content you had copied on clipboard

sudo chmod 600 authorized_keys
```
11. Now try logging onto the server as `grader` with this private key (+passphrase)
```bash
ssh -i "/home/air-walk/.ssh/id_lightsail_ubuntu_rsa" grader@52.90.97.136 -p 2200
```
12. Configure *UFW*:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow 80/tcp
sudo ufw allow 123/tcp
sudo ufw enable
sudo ufw status
```
13. (Re)-configure local TZ to UTC:
```bash
sudo dpkg-reconfigure tzdata
# Select 'None of the above', and then 'UTC'
```
14. Install *Apache HTTP server* and *mod_wsgi*:
```bash
sudo apt-get install -y apache2

# Check that it's up and running by opening your browser and navigating to:
http://52.90.97.136

# Install *mod_wsgi*:
sudo apt-get install -y libapache2-mod-wsgi python-dev
```
15. Configure a demo WSGI app:
```bash
sudo vim /etc/apache2/sites-enabled/000-default.conf
```
Right before ending `</VirtualHost>` block, add:
`WSGIScriptAlias / /var/www/html/myapp.wsgi`

```bash
sudo service apache2 restart
sudo vim /var/www/html/myapp.wsgi
```
Add this content in that file:
```python
def application(environment, response):
  status = '200 OK'
  output = 'Hello, world!'
  response_headers = [('Content-type', 'text/plain'), ('Content-Length', str(len(output)))]
  response(status, response_headers)
  return [output]
```
Check that *Apache* and *mod_wsgi* are installed correctly by opening your browser, navigating to the following link and seeing *Hello, world!*: http://52.90.97.136

16. Install and configure PostgreSQL:
```bash
sudo apt-get install -y postgresql postgresql-contrib
# Connect to DB and update password:
sudo -u postgres psql postgres

# Change password for this user:
\password postgres

# Configure a new DB user with appropriate permissions:
create user catalog WITH PASSWORD '<password-here>';
alter role catalog WITH LOGIN;
alter user catalog CREATEDB;
create database catalog WITH OWNER catalog;
\c catalog
revoke all on schema public FROM public;
grant all on schema public TO catalog;
\q

sudo service postgresql restart
```
17. Fetch catalog app from GitHub and set it up:
```bash
cd /var/www
sudo mkdir catalog
sudo chown -R grader:grader catalog
cd catalog
git clone https://github.com/air-walk/udacity-item-catalog-app.git catalog

# Prohibit access to .git:
vim /var/www/catalog/.htaccess
```
Place this content in that file:
```
RedirectMatch 404 /\.git
```
```bash
# Install Flask and virtualenv:
sudo apt-get install -y python-pip
pip install virtualenv

cd catalog/vagrant/catalog
virtualenv catalogenv
source catalogenv/bin/activate
pip install Flask
pip install sqlalchemy flask-sqlalchemy
pip install oauth2client
pip install psycopg2
deactivate

# Configure and enable new Vhost:
sudo vim /etc/apache2/sites-available/DemoApp.conf
```
Add this content to that file:
```
<VirtualHost *:80>
  ServerName 52.90.97.136
  ServerAdmin admin@catalog.com
  WSGIScriptAlias / /var/www/catalog/catalog/vagrant/catalog/flaskapp.wsgi
  <Directory /var/www/catalog/catalog/vagrant/catalog/>
    Order allow,deny
    Allow from all
  </Directory>
  Alias /static /var/www/catalog/catalog/vagrant/catalog/static
  <Directory //var/www/catalog/catalog/vagrant/catalog/static/>
    Order allow,deny
    Allow from all
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/error-catalog.log
  CustomLog ${APACHE_LOG_DIR}/access-catalog.log combined
</VirtualHost>
```
Go back to the shell and execute:
```bash
sudo a2ensite DemoApp
sudo service apache2 reload

# Update .wsgi file:
vim /var/www/catalog/catalog/vagrant/catalog/flaskapp.wsgi
```
Add this content to that file:
```python
#!/usr/bin/python
import sys
import logging
import os
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/catalog/vagrant/catalog/")

activate_this = os.path.join("/var/www/catalog/catalog/vagrant/catalog/catalogenv/bin/activate_this.py")
execfile(activate_this, dict(__file__=activate_this))

from application import app as application
application.secret_key = '<secret-key-here>'
```
* Place `client_secrets.json` (contains secret generated from the *Google API console*) in this root folder of the project.
* Edit `application.py` and `models.py` to replace *SQLite* with *PostgreSQL* (see code comments present in those files).
* Edit `application.py`'s `redirect_uri` inside `gconnect()` to `http://www.52.90.97.136.xip.io/gconnect`
* In *Google API Console* for your project, add `http://www.52.90.97.136.xip.io` under **Authorized JavaScript origins** section, and `http://www.52.90.97.136.xip.io/gconnect`under **Authorized redirect URIs**. Click *Save*.
* Go back to your shell and execute:
```bash
sudo a2dissite 000-default.conf
sudo service apache2 reload
sudo service apache2 restart
```
18. You should now be able to use the website at: http://www.52.90.97.136.xip.io
19. Logs can be accessed using:
```bash
sudo less /var/log/apache2/access-catalog.log
sudo less /var/log/apache2/error-catalog.log
```

**References:**
[1] https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
