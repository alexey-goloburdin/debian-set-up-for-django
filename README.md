# Debian Server Set Up for Django Instruction

In this guide we will set up clean Debian 9 server for Python and Django projects. We will configure secure SSH connection, install from Debian repositories and from sources all needed packages and ware it together for working Debian Django server.

[Youtube video guide (in Russian)](https://www.youtube.com/watch?v=FLiKTJqyyvs)

## Create user, setup SSH

Connect through SSH to remote Debian server and update repositories and install some initial needed packages:

```
sudo apt-get update ; \
sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make
```

Configure SSH:

```
sudo vim /etc/ssh/sshd_config
    AllowUsers www
    PermitRootLogin no
    PasswordAuthentication no
```

Restart SSH server, change `www` user password:

```
sudo service ssh restart
sudo passwd www
```

## Setup russian locale

```
sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
export LANGUAGE=ru_RU.UTF-8 ; \
export LANG=ru_RU.UTF-8 ; \
export LC_ALL=ru_RU.UTF-8 ; \
sudo locale-gen ru_RU.UTF-8 ; \
sudo dpkg-reconfigure locales
```

In window "Configuring locales":
* OK


- disable en_US.UTF-8 UTF-8
+ enable ru_RU.UTF-8 UTF-8
* OK, OK


+ select ru_RU.UTF-8
* OK

Restart ssh session.

## Install must-have packages

```
sudo apt-get install -y zsh tree redis-server nginx  libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-dev python-imaging python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor
```

Create password for root:

```
sudo passwd
```

## Install ZSH (optional)

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
* Do you want to change your default shell to zsh? [Y/n] **Y**
* enter root password

Make zsh as default shell:

```
sudo usermod $USER -s $(which zsh)
```

Configure some needed aliases:

```
vim ~/.zshrc
    alias cls="clear"
```

## Install Python 3.7

Build from source python 3.7, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tgz ; \
tar xvf Python-3.7.* ; \
cd Python-3.7.3 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/www/.python ; \
make -j8 ; \
sudo make altinstall
```

Now python3.7 in `/home/www/.python/bin/python3.7`. Update pip:

```
sudo /home/www/.python/bin/python3.7 -m pip install -U pip
```

Add python bin to PATH:

```
vim ~/.zshrc
    export PATH=$PATH:/home/www/.python/bin
```

Delete unnecessary files:

```
sudo rm -rf Python-3.7.3.tgz Python-3.7.3
```

Ok, now we can create and activate Python virtual environment:

```
mkdir ~/code ; \
cd code ; \
mkdir project1 ; \
cd project1 ; \
python3.7 -m venv env ; \
. ./env/bin/activate ; \
pip install -U pip
```

## Install and configure Django

Install Django via pip:

```
pip install django
```

Create requirements.txt:

```
pip freeze > requirements.txt
```

Create Django project:

```
django-admin startproject project1
cd project1
```

Test manage.py:

```
./manage.py shell
Ctrl+D
```

Install ipython for comfort (optional):

```
pip install ipython
```

Create Django application:

```
./manage.py startapp first
vim project1/settings.py
    In INSTALLED_APPS section add:
        'first'
```

Test Django server:

```
./manage.py runserver 0.0.0.0:8000
```

In browser at address http://ip_of_our_server:8000 you should see page:

```
DisallowedHost at /
...etc...
```

Add ip of our server to allowed hosts:

```
vim project1/settings.py
    ALLOWED_HOSTS = [] change to ALLOWED_HOSTS = ['ip_of_our_server']
```

Test Django server again:

```
./manage.py runserver 0.0.0.0:8000
```

In browser at address http://ip_of_our_server:8000 you should see:

```
...
The install worked successfully! Congratulations!
...
```

## Install and configure Gunicorn

Install Gunicorn:

```
. ~/code/project1/env/bin/activate
pip install gunicorn
pip freeze > ../requirements.txt
```

Create config. Number of workers = num_of_cores * 2 + 1

```
cd ~/code/project1/project1
vim gunicorn_config.py
    command = '/home/www/code/project1/env/bin/gunicorn'
    pythonpath = '/home/www/code/project1/project1'
    bind = '127.0.0.1:8001'
    workers = 5
    user = 'www'
    limit_request_fields = 32000
    limit_request_field_size = 0
    raw_env = 'DJANGO_SETTINGS_MODULE=project1.settings'
```

```
cd /home/www/code/project1
mkdir bin
vim bin/start_gunicorn.sh
    #!/bin/bash
    source /home/www/code/project1/env/bin/activate
    # source /home/www/code/project1/env/bin/postactivate
    exec gunicorn -c "/home/www/code/project1/project1/gunicorn_config.py" project1.wsgi
chmod +x bin/start_gunicorn.sh
. ./bin/start_gunicorn.sh
```

## Configure Nginx

Change config of Nginx:

```
sudo vim /etc/nginx/sites-enabled/default

server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                proxy_pass http://127.0.0.1:8001;
                proxy_set_header X-Forwarded-Host $server_name;
                proxy_set_header X-Real-IP $remote_addr;
                add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
                add_header Access-Control-Allow-Origin *;
        }
}

sudo service nginx restart
```

In browser at address http://ip_of_our_server you should see:

```
502 Bad Gateway
nginx/1.10.3
```

Add localhost for Nginx proxy to Django allowed hosts and start Gunicorn:

```
vim vim /home/www/code/project1/project1/project1/settings.py
    ALLOWED_HOSTS = ['ip_of_our_server'] change to ALLOWED_HOSTS = ['ip_of_our_server', '127.0.0.1']

/home/www/code/project1/bin/start_gunicorn.sh
```

In browser at address http://ip_of_our_server you should see:

```
...
The install worked successfully! Congratulations!
...
```

## Install and configure Supervisor

Now recommended way is using Systemd instead of supervisor. If you need supervisor — welcome:

```
sudo apt install supervisor
vim project/supervisor.salesbeat.conf
	[program:www_gunicorn]
	command=/home/www/code/project/bin/start_gunicorn.sh
	user=www
	process_name=%(program_name)s
	numprocs=1
	autostart=true
	autorestart=true
	redirect_stderr=true
```

## Install and configure PostgreSQL

Install PostgreSQL 11 and configure locales.

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
RELEASE=$(lsb_release -cs) ; \
echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list ; \
sudo apt update ; \
sudo apt -y install postgresql-11 ; \
sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
export LANGUAGE=ru_RU.UTF-8 ; \
export LANG=ru_RU.UTF-8 ; \
export LC_ALL=ru_RU.UTF-8 ; \
sudo locale-gen ru_RU.UTF-8 ; \
sudo dpkg-reconfigure locales
```

Add locales to `/etc/profile`:

```
sudo vim /etc/profile
    export LANGUAGE=ru_RU.UTF-8
    export LANG=ru_RU.UTF-8
    export LC_ALL=ru_RU.UTF-8
```

Change `postgres` password, create clear database named `dbms_db`:

```
sudo passwd postgres
su - postgres
export PATH=$PATH:/usr/lib/postgresql/11/bin
createdb --encoding UNICODE dbms_db --username postgres
exit
```

Create `dbms` db user and grand privileges to him:

```
sudo -u postgres psql
postgres=# ...
create user dbms with password 'some_password';
ALTER USER dbms CREATEDB;
grant all privileges on database dbms_db to dbms;
\c dbms_db
GRANT ALL ON ALL TABLES IN SCHEMA public to dbms;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public to dbms;
GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to dbms;
CREATE EXTENSION pg_trgm;
ALTER EXTENSION pg_trgm SET SCHEMA public;
UPDATE pg_opclass SET opcdefault = true WHERE opcname='gin_trgm_ops';
\q
exit
```

Now we can test connection. Create `~/.pgpass` with login and password to db for fast connect:

```
vim ~/.pgpass
	localhost:5432:dbms_db:dbms:some_password
chmod 600 ~/.pgpass
psql -h localhost -U dbms dbms_db
```

Run SQL dump, if you have:

```
psql -h localhost dbms_db dbms  < dump.sql
```
