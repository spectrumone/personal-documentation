#Django Deployment on Ubuntu 16.04

###Initial Server instance
follow this: [https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)

###Install Prerequisites
```bash
sudo apt-get update
sudo apt-get install python-pip python-dev libpq-dev postgresql postgresql-contrib nginx
```

###Create PostgreSQL Database and User
To start off, we need to change the PostgreSQL postgres user password; we will not be able to access the server otherwise. As the “postgres” Linux user, we will execute the psql command.

1. On the terminal, type ```sudo -u postgres psql postgres```
2. Set a password for the "postgres" database role using the command: ```\password postgres```
3. To create your db with "postgres" as the user: ```sudo -u postgres createdb name_of_db```

The default when calling ```psql``` is to user the current system user as the db user.

1. create a superuser that has the same name as your current system user: ```sudo -u postgres createuser --superuser $USER```
2. access psql: ```sudo -u postgres psql```
3. inside psql, to change password of current user: ```\password <the name of the the user>```
4. to quit psql ```\q```
5. outside psql, to create the db: ``` sudo -u $USER createdb $USER```

Connecting to your own database should be as easy as ```psql``` now.
* creating other db will now be like: ``` createdb sampledb;```

For faster queries recommended by [Django](https://docs.djangoproject.com/en/1.9/ref/databases/#optimizing-postgresql-s-configuration):
```bash
psql
ALTER ROLE myusername SET client_encoding TO 'utf8';
ALTER ROLE myusername SET default_transaction_isolation TO 'read committed';
ALTER ROLE myusername SET timezone TO 'UTC';
```

###Setup bash profile
copy everything that is inside `etc/skel` to `~`

###Create Virtualenv and Virtualenvwrapper
```bash
sudo pip install virtualenv
sudo pip install virtualenvwrapper
```
add this to `.bashrc`:
```bash
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/Devel
source /usr/local/bin/virtualenvwrapper.sh
```
###Add Django project
```bash
sudo apt-get install git
```
Add `id_rsa.pub` to the either git or bitbucket account where project is hosted.
```bash
git clone <ssh url of project>
```
* run collectstatic. This will be used by nginx later.
* add IP address of the remote server to the `ALLOWED_HOSTS` of Django.
* connect created to django settings.

###Create a Gunicorn systemd Service File
Create and open a systemd service file for Gunicorn with sudo privileges in your text editor:
`sudo nano /etc/systemd/system/gunicorn.service`
```bash
[Unit]
Description=gunicorn daemon
After=network.target #tell init system to only start this after the networking target has been reached

[Service]
# our regular user account ownership of the process since it owns all of the relevant files.
User=username
# www-data group so that Nginx can communicate easily with Gunicorn.
Group=www-data 
WorkingDirectory=/home/username/myproject
ExecStart=/home/username/myproject/myprojectenv/bin/gunicorn --workers 3 --bind \
    unix:/home/username/myproject/myproject.sock myproject.wsgi:application

[Install]
# This will tell systemd what to link this service to if we enable it to start at boot.
# We want this service to start when the regular multi-user system is up and running
WantedBy=multi-user.target
```
start the Gunicorn service we created and enable it so that it starts at boot:
```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```
###Setup NGINX
on `etc/nginx/sites-available/`: `sudo nano <config name preferrably url of site>`
```bash
server {
  listen 80;
  server_name example.com  www.example.com  *.example.com;
  
  # my preference is to add all logs (celery, gunicorn, etc..) in one folder.
  access_log /var/log/example/nginx-access.log;
  error_log /var/log/example/nginx-error.log;

  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
      alias   /home/username/projectname/staticfiles/; # folder of collectstatic 
  }

  location /media/ {
      alias   /home/username/projectname/media/;
  }

  location / {
      include proxy_params;
      proxy_pass http://unix:/home/username/projectname/projectname.sock;
  }
}
```
create symbolic link to `/etc/nginx/sites-enabled/`: `sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled`

check config, restart, and allow nginx on the firewall:
```bash
sudo nginx -t
sudo systemctl restart nginx
sudo ufw allow 'Nginx Full'
```

###Sources
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-16-04)
* [https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
* [https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
