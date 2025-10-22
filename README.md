## Hello 👋! There you will learn how to deploy web application (React + Django) with nginx
### Firstly we will install all required packages

#### Get update

```
apt-get update -y
```

#### Install Required Packages
```
apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx -y
```

#### Configure PostgreSQL
```
sudo -u postgres psql
```

#### Next, create a PostgreSQL database and user for your Django project using the following command:
```
CREATE DATABASE project_name;
```
#### Create user for postgres
```
CREATE USER root WITH PASSWORD 'password'; 
```

#### And Next, setting the default encoding to UTF-8, setting the default transaction isolation scheme to "read committed" and setting the timezone to UTC with the following command:
```
ALTER ROLE root SET client_encoding TO 'utf8';
ALTER ROLE root SET default_transaction_isolation TO 'read committed';
ALTER ROLE root SET timezone TO 'UTC';
```

#### Next, grant all the privileges to your database and exit from the PostgreSQL session with the following command: 

```
postgres=# GRANT ALL PRIVILEGES ON DATABASE project_name TO root;
postgres=#  \q
```


#### Then we will create folders where our projects will be stored
```
mkdir var
```

```
cd /var
```

```
mkdir www
```

```
cd www
```

### And now we will clone our project to server

#### Download git

```
apt install git
```

#### Clone the project

```
git clone git@github.com:username/project_name.git
```

#### Enter to the project

```
cd project_name
```

### And let's install packages

#### Install virtual environment to project
```
pip install virtualenv
```

#### Create virtual environment
```
python3 -m venv myenv
```

#### Activate it
```
. myenv/bin/activate
```
#### Or like that
```
source myenv/bin/activate
```

#### Install all packages from requirements.txt

```
pip install -r requirements.txt
```

#### Make migrations of your project
```
py manage.py makemigrations
```

```
py manage.py migrate
```

#### Create Super User

```
py manage.py createsuperuser
```

<pre>
Username (leave blank to use 'root'): <br>
Email address:<br> 
Password: <br> 
Password (again): <br>
Superuser created successfully.
</pre>

#### Collect Static files

```
py manage.py collectstatic
```

### After collecting static files we can start deploying our website 👌

#### Create a Systemd Service file for Gunicorn
```
nano /etc/systemd/system/gunicorn.service
```
#### Add the following lines:
<pre>
[Unit]
Description=gunicorn daemon for project_name
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/project_name/backend
ExecStart=/var/www/agency/backend/myenv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn/project_name.sock config.wsgi:application
EnvironmentFile=/var/www/project_name/backend/.env

Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
</pre>

#### Install gunicorn
```
pip install gunicorn
```

#### Start gunicorn and enable it
```
systemctl start gunicorn
systemctl enable gunicorn
```

#### You can check the status of Gunicorn with the following command:
```
systemctl status gunicorn
```

#### Output:
<pre>
 gunicorn.service - gunicorn daemon
 Loaded: loaded (/etc/systemd/system/gunicorn.service; enabled; vendor preset: enabled)
 Active: active (running) since Mon 2023-12-08 23:16:14 IST; 10s ago
 Main PID: 2377 (gunicorn)
   CGroup: /system.slice/gunicorn.service
           ├─2377 /root/testproject/.env/bin/python3 /root/project_name/.env/bin/gunicorn --access-logfile - --workers 3 --b
           ├─2384 /root/testproject/.env/bin/python3 /root/project_name/.env/bin/gunicorn --access-logfile - --workers 3 --b
           ├─2385 /root/testproject/.env/bin/python3 /root/project_name/.env/bin/gunicorn --access-logfile - --workers 3 --b
           └─2386 /root/testproject/.env/bin/python3 /root/project_name/.env/bin/gunicorn --access-logfile - --workers 3 --b

Sep 10 20:58:14 mail.example.com systemd[1]: Started gunicorn daemon.
Sep 10 20:58:14 mail.example.com gunicorn[2377]: [2018-09-10 20:58:14 +0530] [2377] [INFO] Starting gunicorn 19.9.0
Sep 10 20:58:14 mail.example.com gunicorn[2377]: [2018-09-10 20:58:14 +0530] [2377] [INFO] Listening at: unix:/root/project_name/project_name.soc
Sep 10 20:58:14 mail.example.com gunicorn[2377]: [2018-09-10 20:58:14 +0530] [2377] [INFO] Using worker: sync
Sep 10 20:58:14 mail.example.com gunicorn[2377]: [2018-09-10 20:58:14 +0530] [2384] [INFO] Booting worker with pid: 2384
Sep 10 20:58:15 mail.example.com gunicorn[2377]: [2018-09-10 20:58:15 +0530] [2385] [INFO] Booting worker with pid: 2385
Sep 10 20:58:15 mail.example.com gunicorn[2377]: [2018-09-10 20:58:15 +0530] [2386] [INFO] Booting worker with pid: 2386
</pre>

### Configure Nginx to Proxy Pass to Gunicorn
```
nano /etc/nginx/sites-available/gunicorn
```
<pre>
# /etc/nginx/sites-available/enzora

upstream gunicorn {
    server unix:/run/gunicorn/agency.sock;
    keepalive 16;
}

# --------------------------------------------------------------------------
# HTTP - redirect everything to https://enzora.uz (force non-www)
# --------------------------------------------------------------------------
server {
    listen 80;
    server_name enzora.uz www.enzora.uz;

    # Redirect all HTTP -> HTTPS non-www
    return 301 https://enzora.uz$request_uri;
}

# --------------------------------------------------------------------------
# HTTPS - main site (non-www) - serves frontend build and proxies to gunicorn
# --------------------------------------------------------------------------
server {
    listen 443 ssl http2;
    server_name enzora.uz;

    root /var/www/agency/frontend/build;
    index index.html;

    # SSL cert (must cover both enzora.uz and www.enzora.uz ideally)
    ssl_certificate /etc/letsencrypt/live/enzora.uz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/enzora.uz/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Logging + performance
    access_log /var/log/nginx/agency_access.log;
    error_log  /var/log/nginx/agency_error.log;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 50M;
  
    # -----------------------------
    # Static (try frontend build first, then Django staticfiles)
    # -----------------------------
    location ^~ /static/ {
        alias /var/www/agency/backend/staticfiles/;
        access_log off;
        log_not_found off;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000, immutable";
    }

    # -----------------------------
    # Media (user-uploaded) — short-term browser cache + open_file_cache for nginx
    # -----------------------------
    location /media/ {
        alias /var/www/agency/backend/media/;
        try_files $uri =404;

        # Browser caching: short-term (1 hour). Switch to long cache if filenames are versioned.
        expires 1h;
        add_header Cache-Control "public, max-age=3600, must-revalidate";

        # Keep logs off for static media
        access_log off;
        log_not_found off;

        # Speed up serving by caching open file descriptors
        open_file_cache max=1000 inactive=30s;
        open_file_cache_valid 30s;
        open_file_cache_min_uses 1;
        open_file_cache_errors on;
    }

    # Health check
    location = /healthz {
        access_log off;
        return 200 'OK';
        add_header Content-Type text/plain;
    }

    # -----------------------------
    # API caching rules (GET only)
    # -----------------------------
    location ~ ^/api/ {
        proxy_pass http://gunicorn;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_cache apicache;
        proxy_cache_key "$scheme$request_method$host$request_uri|lang=$req_lang";

        set $no_cache 0;
        if ($http_authorization != "") { set $no_cache 1; }
        if ($http_cookie != "")        { set $no_cache 1; }
        if ($http_cache_control ~* "no-cache") { set $no_cache 1; }
        if ($arg__nocache = "1") { set $no_cache 1; }

        proxy_cache_bypass $no_cache;
        proxy_no_cache $no_cache;

        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_valid any 1m;

        proxy_cache_lock on;
        proxy_cache_lock_timeout 10s;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 5s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;

        proxy_buffering on;
        proxy_buffers 8 32k;
        proxy_buffer_size 64k;
        proxy_busy_buffers_size 128k;

        add_header X-Proxy-Cache $upstream_cache_status;
        add_header X-Cache-Key $upstream_cache_status always;

        gzip on;
        gzip_proxied any;
    }
  
    # Admin proxy (no cache)
    location /admin/ {
        proxy_pass http://gunicorn;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # -----------------------------
    # SEO bot handling (pre-rendered files)
    # -----------------------------
    error_page 418 = @seo_for_bots;
    location ~* ^/(?<lang>en|ru|uz|ar|de|es|fr|tr|uk)(/.*)?$ {
        if ($is_bot = 1) {
            return 418;
        }
        try_files $uri $uri/ /index.html;
    }

    location @seo_for_bots {
        try_files /var/www/agency/frontend/build/seo/$lang.html /index.html;
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Compression and security headers
    gzip on;
    gzip_http_version 1.1;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 5;
    gzip_min_length 256;
    gzip_types text/plain text/css application/json application/javascript text/javascript application/ld+json application/xml text/xml application/rss+xml image/svg+xml;

    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
      
    # HSTS (no includeSubDomains here to avoid accidental lockout of subdomains while testing)
    add_header Strict-Transport-Security "max-age=63072000" always;
}

# --------------------------------------------------------------------------
# HTTPS - www -> redirect to non-www (requires cert that covers www)
# --------------------------------------------------------------------------
server {
    listen 443 ssl http2;
    server_name www.enzora.uz;

    ssl_certificate /etc/letsencrypt/live/enzora.uz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/enzora.uz/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    # Redirect to canonical non-www
    return 301 https://enzora.uz$request_uri;
}
</pre>

#### Save and close the file. Then, enable the Nginx virtual host by creating symlink:
```
ln -s /etc/nginx/sites-available/gunicorn /etc/nginx/sites-enabled/
```

#### Check is it works correctly
```
nginx -t
```
#### Output:
<pre>
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
</pre>

#### Finally, restart Nginx and gunicorn by running the following command:
```
systemctl daemon-reload
systemctl restart gunicorn
systemctl restart nginx
```

<hr>
<br>

<h3>This was an example of how you can deploy your Django project, there are many more ways but this is the best one that can be, good luck in development 🍀</h3>
