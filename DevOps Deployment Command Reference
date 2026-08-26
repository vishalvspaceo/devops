# DevOps Deployment Command Reference

Deployment commands for PHP/Laravel, Node.js and React on Ubuntu servers (EC2 or any VPS).

**Read this first:** never paste a command you don't understand into a production server. Every destructive command in this file is marked with a warning. Test on a staging server before touching production.

Assumed stack: Ubuntu 22.04/24.04, Nginx, systemd.

---

## 1. Connect to the server

```bash
# SSH with a key file (AWS EC2 gives you a .pem)
chmod 400 my-key.pem                      # key must not be world-readable or SSH refuses it
ssh -i my-key.pem ubuntu@13.234.56.78     # Ubuntu AMI user is 'ubuntu', Amazon Linux is 'ec2-user'

# Save it in ~/.ssh/config so you can just type: ssh prod
# Host prod
#   HostName 13.234.56.78
#   User ubuntu
#   IdentityFile ~/.ssh/my-key.pem

# Copy files up
scp -i my-key.pem app.zip ubuntu@13.234.56.78:/tmp/
rsync -avz --delete ./dist/ ubuntu@13.234.56.78:/var/www/app/   # --delete removes remote files not in source
```

`rsync` beats `scp` for deployments — it transfers only changed files.

---

## 2. Prepare a fresh server

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx git curl unzip ufw

# Firewall (do this BEFORE opening the server to traffic)
sudo ufw allow OpenSSH        # do this first or you lock yourself out
sudo ufw allow 'Nginx Full'   # opens 80 and 443
sudo ufw enable
sudo ufw status

# Timezone
sudo timedatectl set-timezone Asia/Kolkata
```

On AWS, security groups do the same job as ufw at the network level. Use both — defence in depth.

---

## 3. Nginx — the front door for all three stacks

```bash
sudo nano /etc/nginx/sites-available/myapp     # write config here
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default       # remove the default welcome page

sudo nginx -t                # ALWAYS test config before reloading
sudo systemctl reload nginx  # reload = no dropped connections
sudo systemctl restart nginx # restart = brief downtime, only if reload fails
sudo systemctl status nginx
```

`nginx -t` before every reload. A syntax error plus a blind restart equals a dead site.

### Config: React / static SPA

```nginx
server {
    listen 80;
    server_name myapp.com;
    root /var/www/myapp/dist;    # Vite uses dist/, Create React App uses build/
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;   # critical: makes client-side routing work
    }

    location ~* \.(js|css|png|jpg|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Without that `try_files` line, refreshing on `/dashboard` returns 404. It is the single most common React deployment bug.

### Config: Node.js (reverse proxy)

```nginx
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;      # needed for websockets
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Node listens on localhost only; Nginx handles the public port, TLS and static files.

### Config: PHP / Laravel

```nginx
server {
    listen 80;
    server_name myapp.com;
    root /var/www/myapp/public;    # note: /public, NOT the project root
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;   # match your installed PHP version
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* { deny all; }      # block .env, .git etc.
}
```

Pointing `root` at the project folder instead of `public/` exposes your `.env` file to the internet. Get this right.

---

## 4. Apache — the other front door

Apache and Nginx do the same job. Pick one. **Never run both on port 80** — the second one fails to start and you'll waste an hour on it.

Rough guidance: Nginx for Node/React and high-traffic static serving; Apache if the project already ships `.htaccess` files or your team knows it better. Both are production-grade.

```bash
sudo apt install -y apache2

# Enable the modules you need (nothing works without these)
sudo a2enmod rewrite headers ssl
sudo a2enmod proxy proxy_http proxy_fcgi setenvif     # proxy_* only needed for Node/PHP-FPM

# Site management — Apache's equivalent of Nginx's symlinks
sudo nano /etc/apache2/sites-available/myapp.conf
sudo a2ensite myapp.conf
sudo a2dissite 000-default.conf      # disable the default welcome page

sudo apache2ctl configtest           # ALWAYS test before reloading (Nginx's `nginx -t`)
sudo systemctl reload apache2        # graceful, no dropped connections
sudo systemctl restart apache2       # only if reload fails
sudo systemctl status apache2

# Logs
sudo tail -f /var/log/apache2/error.log
sudo tail -f /var/log/apache2/access.log
```

`a2enmod` / `a2ensite` just create symlinks in `mods-enabled/` and `sites-enabled/`. Knowing that makes Apache far less mysterious.

### Vhost: PHP / Laravel

```apache
<VirtualHost *:80>
    ServerName myapp.com
    DocumentRoot /var/www/myapp/public      # /public, NOT the project root

    <Directory /var/www/myapp/public>
        AllowOverride All                    # required for Laravel's .htaccess to work
        Require all granted
        Options -Indexes                     # don't list directory contents
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/myapp_error.log
    CustomLog ${APACHE_LOG_DIR}/myapp_access.log combined
</VirtualHost>
```

```bash
# Option A — mod_php (simplest)
sudo apt install -y libapache2-mod-php8.3

# Option B — PHP-FPM (better performance, same pool style as Nginx)
sudo apt install -y php8.3-fpm
sudo a2enconf php8.3-fpm
```

`AllowOverride All` is the Apache-specific gotcha. Without it Laravel's `.htaccess` is ignored and every route except `/` returns 404.

### Vhost: React / static SPA

```apache
<VirtualHost *:80>
    ServerName myapp.com
    DocumentRoot /var/www/myapp/dist

    <Directory /var/www/myapp/dist>
        Require all granted
        FallbackResource /index.html     # the Apache equivalent of Nginx try_files
    </Directory>
</VirtualHost>
```

`FallbackResource /index.html` is what fixes the 404-on-refresh problem. One line, same bug as the Nginx `try_files` case.

### Vhost: Node.js reverse proxy

```apache
<VirtualHost *:80>
    ServerName api.myapp.com

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    RequestHeader set X-Forwarded-Proto "http"
</VirtualHost>
```

Websockets need `mod_proxy_wstunnel` enabled plus a `ProxyPass /socket.io/ ws://127.0.0.1:3000/socket.io/` line.

### SSL on Apache

```bash
sudo certbot --apache -d myapp.com -d www.myapp.com
```

### Nginx ↔ Apache translation table

| Task | Nginx | Apache |
|---|---|---|
| Test config | `nginx -t` | `apache2ctl configtest` |
| Enable a site | `ln -s` into `sites-enabled/` | `a2ensite name.conf` |
| Enable a module | compiled in | `a2enmod name` |
| SPA fallback | `try_files $uri /index.html` | `FallbackResource /index.html` |
| Reverse proxy | `proxy_pass` | `ProxyPass` / `ProxyPassReverse` |
| Web root | `root` | `DocumentRoot` |
| Per-directory config | not supported | `.htaccess` (needs `AllowOverride All`) |
| Error log | `/var/log/nginx/error.log` | `/var/log/apache2/error.log` |

---

## 5. PHP / Laravel deployment

```bash
# Install stack
sudo apt install -y php8.3-fpm php8.3-mysql php8.3-mbstring php8.3-xml \
                    php8.3-curl php8.3-zip php8.3-bcmath php8.3-gd
sudo apt install -y composer

# Deploy
cd /var/www/myapp
git pull origin main
composer install --no-dev --optimize-autoloader     # --no-dev keeps dev packages off prod

cp .env.example .env        # first deploy only
php artisan key:generate    # first deploy only
nano .env                   # set APP_ENV=production, APP_DEBUG=false, DB creds

php artisan migrate --force          # --force required in production
php artisan storage:link

# Cache for speed (run in this order, after every deploy)
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache

# Permissions — Nginx runs as www-data and must write to these two dirs
sudo chown -R www-data:www-data /var/www/myapp/storage /var/www/myapp/bootstrap/cache
sudo chmod -R 775 /var/www/myapp/storage /var/www/myapp/bootstrap/cache

sudo systemctl reload php8.3-fpm
```

### Laravel troubleshooting

```bash
php artisan config:clear && php artisan cache:clear && php artisan view:clear
php artisan optimize:clear         # clears everything at once
tail -f storage/logs/laravel.log
sudo tail -f /var/log/nginx/error.log
php artisan queue:restart          # queue workers hold OLD code until restarted
```

`APP_DEBUG=true` in production leaks your database credentials on any error page. Check it on every deploy.

### Queue workers with systemd

```bash
sudo nano /etc/systemd/system/laravel-worker.service
```

```ini
[Unit]
Description=Laravel Queue Worker
After=network.target

[Service]
User=www-data
Restart=always
ExecStart=/usr/bin/php /var/www/myapp/artisan queue:work --sleep=3 --tries=3 --max-time=3600

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now laravel-worker
sudo journalctl -u laravel-worker -f
```

---

## 6. Node.js deployment

```bash
# Install Node via nvm (avoid apt's outdated version)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22
nvm alias default 22
node -v && npm -v

# Deploy
cd /var/www/api
git pull origin main
npm ci --omit=dev          # 'ci' respects package-lock exactly — always use it over 'install' on servers
npm run build              # only if TypeScript or a build step exists
```

`npm ci` gives reproducible installs. `npm install` can silently upgrade a dependency and break production.

### PM2 process manager

```bash
sudo npm install -g pm2

pm2 start dist/server.js --name api
pm2 start npm --name api -- start          # if you must go through an npm script
pm2 start dist/server.js -i max            # cluster mode: one process per CPU core

pm2 list
pm2 logs api
pm2 logs api --lines 200
pm2 monit                                  # live dashboard

pm2 reload api                             # zero-downtime reload — prefer this
pm2 restart api                            # hard restart, brief downtime
pm2 stop api
pm2 delete api

pm2 save                                   # snapshot the current process list
pm2 startup                                # prints a command — run it to survive reboots
pm2 flush                                  # clear logs when they get large
```

`pm2 save` after `pm2 startup`, or your apps won't come back after a server reboot.

---

## 7. React / Vue / Angular deployment

```bash
# Build (do this in CI, not on a small production server — builds are memory-hungry)
npm ci
npm run build          # Vite outputs dist/, CRA outputs build/

# Ship the built folder only
rsync -avz --delete dist/ ubuntu@server:/var/www/myapp/dist/

sudo chown -R www-data:www-data /var/www/myapp/dist
sudo nginx -t && sudo systemctl reload nginx
```

If the build gets killed on a t2.micro, it ran out of RAM. Add swap or build in CI:

```bash
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Environment variables are baked in at **build** time, not runtime. Changing `VITE_API_URL` means rebuilding — there is no way around this.

---

## 8. SSL / HTTPS with Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d myapp.com -d www.myapp.com    # edits your Nginx config automatically

sudo certbot renew --dry-run     # verify auto-renewal works
sudo certbot certificates        # list certs and expiry dates
```

Point your DNS A record at the server **before** running certbot — validation fails otherwise. Renewal is automatic via a systemd timer.

---

## 9. Databases

```bash
# MySQL
sudo apt install -y mysql-server
sudo mysql_secure_installation
sudo mysql -u root -p

mysqldump -u user -p dbname > backup_$(date +%F).sql       # backup
mysql -u user -p dbname < backup.sql                        # restore

# PostgreSQL
sudo apt install -y postgresql
sudo -u postgres psql
pg_dump -U user dbname > backup_$(date +%F).sql
psql -U user dbname < backup.sql
```

Take a backup before every migration. No exceptions.

---

## 10. Docker — one workflow for every stack

```bash
# Install
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER     # log out and back in for this to apply

# Build and run
docker build -t myapp:v1 .
docker build -t myapp:v1 --build-arg NODE_ENV=production .
docker run -d --name myapp -p 3000:3000 --env-file .env --restart unless-stopped myapp:v1

# Inspect
docker ps
docker ps -a
docker logs -f myapp
docker logs --tail 100 myapp
docker exec -it myapp sh          # shell inside the container
docker stats
docker inspect myapp

# Lifecycle
docker stop myapp && docker rm myapp
docker images
docker rmi myapp:v1

# Compose
docker compose up -d
docker compose up -d --build
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose down               # stops and removes containers
docker compose down -v            # WARNING: -v deletes volumes, i.e. your database data
docker compose restart api

# Cleanup
docker system df                  # see what's using disk
docker image prune                # remove dangling images
docker system prune -a            # WARNING: removes all unused images, networks, build cache
```

`docker compose down -v` on a production database is unrecoverable. Know which flag you're typing.

### Push to AWS ECR

```bash
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-south-1.amazonaws.com

docker tag myapp:v1 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:v1
docker push 123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp:v1
```

Tag with a git SHA or version number, never only `latest` — you cannot roll back to a tag that keeps moving.

---

## 11. Troubleshooting a broken deployment

Work through this in order:

```bash
# 1. Is the service running?
sudo systemctl status nginx
pm2 list
docker ps

# 2. What do the logs say?
sudo tail -f /var/log/nginx/error.log
sudo journalctl -u nginx -n 100 --no-pager
sudo journalctl -u myapp -f
pm2 logs api --err
docker logs --tail 100 myapp
tail -f storage/logs/laravel.log         # Laravel

# 3. Is the port actually listening?
sudo ss -tulpn | grep 3000
curl -I http://localhost:3000            # test locally, bypassing Nginx and firewall

# 4. Out of disk or memory?
df -h
du -sh /var/log/*
free -h

# 5. Permissions?
ls -la /var/www/myapp
namei -l /var/www/myapp/public/index.php  # shows permissions at every path level

# 6. Firewall or security group?
sudo ufw status
```

Interpreting the common failures:

| Symptom | Usual cause |
|---|---|
| 502 Bad Gateway | App process is down, or wrong port / wrong PHP-FPM socket |
| 403 Forbidden | Permissions, or `root` points at the wrong directory |
| 404 on refresh (SPA) | Missing `try_files ... /index.html` |
| 500 on Laravel | Check `storage/logs/laravel.log`; usually permissions or `.env` |
| Site serves old code | A cache: `artisan optimize:clear`, `pm2 reload`, or browser/CDN |
| Build killed | Out of RAM — add swap or build in CI |

---

## 12. Permissions cheat sheet

```bash
sudo chown -R www-data:www-data /var/www/myapp     # web server owns web files
sudo find /var/www/myapp -type d -exec chmod 755 {} \;   # directories
sudo find /var/www/myapp -type f -exec chmod 644 {} \;   # files
sudo chmod -R 775 storage bootstrap/cache                # Laravel writable dirs
```

Never `chmod 777`. It fixes the symptom, opens a hole, and marks you as a beginner in any code review.

---

## 13. Safe deploy and rollback

```bash
# Tag every release so rollback is one command
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin v1.4.0

# Rollback
git checkout v1.3.0
composer install --no-dev -o    # or: npm ci --omit=dev
php artisan migrate:rollback    # only if the bad release added migrations
php artisan optimize:clear
pm2 reload api

# Docker rollback — just run the previous tag
docker compose down && docker compose up -d   # after pointing the image tag back
```

A deploy you can't reverse in under five minutes isn't finished. Plan the rollback before you ship.

---

## 14. Jenkins — automating all of the above

Jenkins is what runs your deploy steps automatically on every push. Everything in sections 1–13 becomes a stage in a pipeline.

### Install

```bash
sudo apt install -y fontconfig openjdk-21-jre     # Jenkins needs Java
java -version

curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update && sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins

# First-login password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Then open `http://SERVER_IP:8080`. Two things immediately:

```bash
sudo ufw allow 8080          # or an AWS security group rule, ideally restricted to your IP
```

Never leave Jenkins open to the internet on plain HTTP with a weak admin password. A publicly exposed Jenkins with permissive permissions is a remote code execution box — it can run arbitrary commands on your infrastructure by design. Put it behind Nginx with TLS, or restrict access to your office IP or a VPN.

### Reverse proxy behind Nginx

```nginx
server {
    listen 80;
    server_name ci.myapp.com;
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect http:// https://;
    }
}
```

### Day-to-day operations

```bash
sudo systemctl restart jenkins
sudo journalctl -u jenkins -f              # Jenkins' own logs
sudo tail -f /var/log/jenkins/jenkins.log

# Jenkins state lives entirely here — this is what you back up
ls /var/lib/jenkins
sudo tar czf jenkins-backup-$(date +%F).tar.gz /var/lib/jenkins

# Let Jenkins run Docker builds
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins            # required for the group change to apply

# Give Jenkins an SSH key to reach deploy targets
sudo -u jenkins ssh-keygen -t ed25519
sudo cat /var/lib/jenkins/.ssh/id_ed25519.pub    # add to the target server's authorized_keys
```

Plugins worth installing on day one: **Git**, **Pipeline**, **Docker Pipeline**, **NodeJS**, **SSH Agent**, **Credentials Binding**, **Blue Ocean**.

### Jenkinsfile: React / static build

Put this at the repo root as `Jenkinsfile`. Jenkins reads it automatically.

```groovy
pipeline {
    agent any
    environment {
        DEPLOY_HOST = 'ubuntu@13.234.56.78'
        DEPLOY_PATH = '/var/www/myapp/dist/'
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install') {
            steps { sh 'npm ci' }
        }
        stage('Test') {
            steps { sh 'npm test -- --watchAll=false' }
        }
        stage('Build') {
            steps { sh 'npm run build' }
        }
        stage('Deploy') {
            when { branch 'main' }              // only main reaches production
            steps {
                sshagent(['deploy-ssh-key']) {
                    sh "rsync -avz --delete dist/ ${DEPLOY_HOST}:${DEPLOY_PATH}"
                }
            }
        }
    }
    post {
        failure { echo 'Build failed — deploy skipped.' }
        always  { cleanWs() }
    }
}
```

### Jenkinsfile: Node.js API with Docker

```groovy
pipeline {
    agent any
    environment {
        IMAGE = "myapp/api:${env.BUILD_NUMBER}"     // build number = rollback target
    }
    stages {
        stage('Install & Test') {
            steps {
                sh 'npm ci'
                sh 'npm test'
            }
        }
        stage('Build image') {
            steps { sh "docker build -t ${IMAGE} ." }
        }
        stage('Push to ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws ecr get-login-password --region ap-south-1 \
                          | docker login --username AWS --password-stdin $ECR_REGISTRY
                        docker tag $IMAGE $ECR_REGISTRY/$IMAGE
                        docker push $ECR_REGISTRY/$IMAGE
                    '''
                }
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sshagent(['deploy-ssh-key']) {
                    sh """
                        ssh ${DEPLOY_HOST} 'docker pull \$ECR_REGISTRY/${IMAGE} && \
                        docker stop api || true && docker rm api || true && \
                        docker run -d --name api -p 3000:3000 --restart unless-stopped \$ECR_REGISTRY/${IMAGE}'
                    """
                }
            }
        }
    }
}
```

Note the single-quoted `sh ''' ... '''` block in the credentials stage. Groovy does not interpolate single-quoted strings, which keeps your secrets out of the build log. Using double quotes there leaks them.

### Jenkinsfile: PHP / Laravel

```groovy
pipeline {
    agent any
    stages {
        stage('Install') {
            steps { sh 'composer install --no-interaction --prefer-dist' }
        }
        stage('Test') {
            steps { sh 'php artisan test' }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sshagent(['deploy-ssh-key']) {
                    sh '''
                        ssh $DEPLOY_HOST "cd /var/www/myapp && \
                        git pull origin main && \
                        composer install --no-dev --optimize-autoloader && \
                        php artisan migrate --force && \
                        php artisan optimize:clear && \
                        php artisan config:cache && php artisan route:cache && \
                        php artisan queue:restart"
                    '''
                }
            }
        }
    }
}
```

### Jenkins practices that separate working setups from fragile ones

- **Credentials go in Jenkins' credential store**, referenced by ID. Never hardcoded in a Jenkinsfile, which lives in Git.
- **The Jenkinsfile belongs in the repo**, not configured through the web UI. Pipeline-as-code means your build process is versioned and reviewable.
- **Gate deploys on branch**: `when { branch 'main' }`. Feature branches build and test, only main ships.
- **Fail loudly.** A pipeline that swallows test failures is worse than no pipeline.
- **Back up `/var/lib/jenkins`.** It holds every job, credential and config. Losing it means rebuilding from memory.
- **Don't run builds on the Jenkins controller** once you have more than a couple of jobs — add agents.

### Jenkins vs GitHub Actions

Jenkins wins when you need self-hosted control, have on-prem servers, or work somewhere that already runs it — very common in Indian service companies. GitHub Actions wins on setup time: no server to maintain, no plugin management, YAML in your repo and you're done.

Learn Jenkins because your job needs it. Learn GitHub Actions too, because it's what most new projects choose.

---

## 15. Commands that need a second look

| Command | Why |
|---|---|
| `rm -rf /path` | No undo. Read the path twice, especially with a variable in it. |
| `docker compose down -v` | Deletes volumes — your database data. |
| `docker system prune -a` | Removes all unused images and build cache. |
| `git reset --hard` | Discards uncommitted work permanently. |
| `git push --force` | Rewrites shared history. Use `--force-with-lease`. |
| `php artisan migrate:fresh` | **Drops every table.** Never on production. |
| `chmod -R 777` | Opens the app to anyone on the box. |
| `ufw enable` without allowing SSH | Locks you out of your own server. |
| `a2dissite` on the wrong vhost | Silently takes a live site offline. |
| Jenkins with "Anyone can do anything" | Public remote code execution on your infra. |
| `rm -rf /var/lib/jenkins` | Every job, credential and config, gone. |
| `mysql -e "DROP ..."` | Instant, silent, unrecoverable without a backup. |

---

## 16. Next step: stop doing this by hand

Everything above is the manual version. It teaches you what's happening, and you should do it manually a few times. But manual deployment doesn't scale and doesn't repeat reliably.

The progression from here:

1. **Bash script** — wrap your deploy steps in one file with `set -euo pipefail`
2. **Jenkins or GitHub Actions** — run that script automatically on every push to `main` (section 14)
3. **Docker** — make "works on my machine" mean something
4. **Terraform** — the server itself becomes code
5. **Kubernetes / ECS** — orchestrate many containers

Each step removes a class of human error. Aim to reach step 2 within a month of getting comfortable here.
