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
Check that *Apache* and *mod_wsgi* are installed correctly by opening your browser, navigating to the following link and seeing *Hello, world!*:
http://52.90.97.136



21. Fetch catalog app from GitHub and set it up:
```bash
# TODO
```

**References:**
[1] https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
