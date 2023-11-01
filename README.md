# Dj_project
Deploy stack for Debian + Nginx + Postgresql + Django

## SINGLE HOST DEPLOY
https://habr.com/ru/articles/546778/

https://timeweb.cloud/docs/unix-guides/creating-and-resizing-swap


### Create instance
minimum requirements:
- Debian 10
- 2 CPU
- 2 Gb RAM
- 40 Gb SSD

### Connect to your server via SSH:
`ssh root@<ip>`

### Configure swap:
```bash
swapoff -a
dd if=/dev/zero of=/swap bs=1M count=2048
chmod 600 /swap && mkswap /swap
swapon /swap
echo "/swap swap swap defaults 0 0"| tee -a /etc/fstab
```
### Install essentials
```bash
apt update && apt upgrade -y
apt install -y python3-pip python3.11-venv python3-dev git vim curl tree build-essential libpq-dev postgresql postgresql-contrib
```

### Configure VIM:
```bash
git clone https://github.com/preservim/nerdtree.git ~/.vim/pack/vendor/start/nerdtree
vim -u NONE -c "helptags ~/.vim/pack/vendor/start/nerdtree/doc" -c q

echo ":set number
:syntax on
:set expandtab
:set smarttab
:set tabstop=4
:set softtabstop=4
:set shiftwidth=4
:set mouse=a
:nmap <F6> :NERDTreeToggle<CR>
:set encoding=utf8
:set ffs=unix,dos,mac
:set ignorecase
:set smartcase
:set hlsearch" > ~/.vimrc
```

### Configure PostgreSQL:
```bash
sudo -u postgres psql
ALTER USER postgres WITH PASSWORD 'postgres';
create database mybase;
ALTER ROLE postgres SET client_encoding TO 'utf8';
ALTER ROLE postgres SET default_transaction_isolation TO 'read committed';
ALTER ROLE postgres SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE mybase TO postgres;
\q
```

- Open postgresql.conf `vim /etc/postgresql/15/main/postgresql.conf`
- Find section "Connection Settings" and uncomment the string "listen_addresses"
- Change it to `listen_addresses = '*'`

```bash
echo 'host    all        all        0.0.0.0/0        md5' >> /etc/postgresql/15/main/pg_hba.conf
systemctl restart postgresql.service
```


### Configure Django:

- Сreate a working directory and go to it: `mkdir /var/www && cd /var/www`
- Download this project and go to it: `git clone https://github.com/andryplekhanov/Dj_project.git && cd Dj_project`
- Create ".env" out of the ".env.dist" and edit it: `mv .env.dist .env && vim .env`
- Create a virtual enviroment: `python3 -m venv env`
- Activate a virtual enviroment: `. ./env/bin/activate`
- Upgrade PIP: `pip install -U pip`
- Install requirements: `python3 -m pip install -r requirements.txt`
- Apply migrations: `python3 manage.py migrate`
- Create a superuser: `python3 manage.py createsuperuser`
- Collect static: `python3 manage.py collectstatic --no-input --clear`


### Configure Gunicorn:
- Install Gunicorn: `pip install gunicorn`
- Deactivate a virtual enviroment: `deactivate`
- Go here: `cd /etc/systemd/system/`
- Create "gunicorn.service":
```bash
echo "[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=root
WorkingDirectory=/var/www/Dj_project
ExecStart=/var/www/Dj_project/env/bin/gunicorn --workers 5 --bind unix:/run/gunicorn.sock Dj_project.wsgi:application

[Install]
WantedBy=multi-user.target" > gunicorn.service
```
- Create "gunicorn.socket":
```bash
echo "[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target" > gunicorn.socket
```


### Configure NGINX:
- Install NGINX: `apt install -y nginx`
- Go here: `cd /etc/nginx/sites-available/`
- Create config-file "Dj_project":
```bash
echo "server {
    listen 80;
    server_name <ip>;
    
    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /var/www/Dj_project;
    }
    
    location /media/ {
        root /var/www/Dj_project;
    }
    
    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}" > Dj_project
```
- Create a link: `sudo ln -s /etc/nginx/sites-available/Dj_project /etc/nginx/sites-enabled/`
```bash
systemctl restart nginx
systemctl enable gunicorn
systemctl start gunicorn
```
