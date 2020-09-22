# NodeJS Deployment Guide

> Deployment guide for NodeJS apps with Nginx, PM2, and Certbot.

## Deployment Types
1. [Initial Server Setup](https://github.com/babegadev/node-deploy/#initial-server-deployment)
2. [New App Deployment](https://github.com/babegadev/node-deploy/#initial-server-deployment)

## Resources
1. [Coderocketfuel](https://coderrocketfuel.com/article/deploy-a-nodejs-application-to-digital-ocean-with-https)

# Initial server deployment

## 1. Set up Ubuntu Server 20.04
```
sudo apt update
sudo apt upgrade
```

## 2. Set up OpenSSH Server and copy SSH keys
Remote:
```
sudo ufw enable
sudo ufw app list
sudo ufw allow OpenSSH
sudo ufw status
```
Local:
```
ssh-copy-id user@host
ssh user@host
```

## 3. Install Required Packages
1. NodeJS and NPM
```
curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
sudo apt install nodejs
node --version
```
2. PM2 Process Manager
```
sudo npm install -g pm2

# Other pm2 commands
pm2 show app
pm2 status
pm2 restart app
pm2 stop app
pm2 logs (Show log stream)
pm2 flush (Clear logs)
```
3. Nginx
```
sudo apt update
sudo apt install nginx

# Disable Default Site
cd /etc/nginx/sites-enabled
sudo unlink default
```
4. Certbot
```
sudo add-apt-repository ppa:certbot/certbot
sudo apt install certbot
```

## 4. Create app directory for all apps
```
mkdir apps
```

## 5. Install NoIP dynamic update client (if needed)
- Sign up for NoIP
- Create a hostname

Install build-essential
```
sudo apt install build-essential
```
Install NoIP client
```
cd /usr/local/src
wget http://www.no-ip.com/client/linux/noip-duc-linux.tar.gz
tar xzf noip-duc-linux.tar.gz
cd noip-2.1.9-1
make
make install
```
Configure NoIP
```
sudo /usr/local/bin/noip2 -C
```
Create a new service
```
sudo nano /etc/systemd/system/noip2.service
```
noip2.service file content
```
[Unit]
Description=No-ip.com dynamic IP address updater
After=network.target
After=syslog.target

[Install]
WantedBy=multi-user.target
Alias=noip.service

[Service]
# Start main service
ExecStart=/usr/local/bin/noip2
Restart=always
Type=forking
```
Enable on startup
```
sudo systemctl daemon-reload
sudo systemctl enable noip2
sudo systemctl start noip2
sudo systemctl status noip2
sudo systemctl stop noip2
sudo reboot now
sudo systemctl status noip2
```

---

# Adding New Apps to server

## 1. Clone Project in App directory
```
cd apps

git clone yourproject.git
```

## 2. Set upp NodeJS App
1. Set up application port. Make sure it is not in use 
2. Set up environment variables (if used)
3. Install dependencies and test the app
```
cd yourproject
npm install
npm start

curl http://localhost:APP_PORT

ctrl + c
```

## 3. Setup PM2 process manager to keep your app running
```
sudo pm2 start <app.js> --name <appname>
sudo pm2 startup systemd
sudo pm2 save
```

## 4. Configure domain name and SSL Certificate
Configure subdomain records in cloudflare
- Add needed records in cloudflare

Issue SSL Certificate with cerbot
```
sudo certbot --manual -d domain.com -d www.domain.com --preferred-challenge dns certonly
# Add a txt record with the prompted config in cloudflare
```

## 5. Configure Nginx Reverse Proxy
Create a new firewall rule first
```
sudo ufw app list
sudo ufw allow 'Nginx Full'
sudo ufw status
```
Create a new server block
```
cd /etc/nginx/conf.d
sudo nano app_name.conf
```
Add the following to the file. Change <yourdomain> and <APP_PORT>
```
server {
  listen 80;
  server_name yourdomain.com www.yourdomain.com;
  return 301 https://$host$request_uri;
}

server {
	
  server_name yourdomain.com www.yourdomain.com;

  location / {
    proxy_pass http://localhost:APP_PORT;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }
  
  listen 443 ssl;
  ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

}
```
Then check nginx config and restart nginx
```
sudo nginx -t
# If no error
sudo service nginx restart
```

Now visit https://yourdomain.com and you should see your Node app
