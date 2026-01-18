# Rainloop-Mail-Portal-on-Ubuntu
This project sets up a Rainloop webmail portal on an Ubuntu server with nginx and PHP-FPM, running on port 8081. HTTPS is enforced with a Let's Encrypt SSL certificate, and HTTP traffic is automatically redirected to HTTPS.   This setup is suitable for hosting a self-managed webmail portal accessible via a public domain

Features

Rainloop webmail interface accessible at https://<domain>:8081

PHP-FPM backend for fast PHP execution

Nginx reverse proxy with SSL termination

Automatic HTTP → HTTPS redirect

SMTP configuration for outgoing email (Postfix)

UFW firewall configured for ports:

HTTP/HTTPS: 8081

SMTP: 25, 587

SSH: 22

Server Requirements

Ubuntu Server 18.04+

Nginx (1.14+)

PHP 7.2+ (php7.2-fpm recommended)

Postfix configured for SMTP

Let's Encrypt SSL certificate (or equivalent)

Installation and Setup
1. Install Nginx, PHP, and Rainloop
sudo apt update
sudo apt install nginx php7.2-fpm php7.2-mbstring php7.2-xml php7.2-curl php7.2-gd php7.2-zip unzip -y


Download Rainloop:

mkdir -p /var/www/html/rainloop
cd /var/www/html/rainloop
wget https://www.rainloop.net/repository/webmail/rainloop-latest.zip
unzip rainloop-latest.zip
rm rainloop-latest.zip

2. Configure Nginx

Create the site configuration in /etc/nginx/sites-available/rainloop-https:

server {
    listen 8081;
    server_name yourdomain.ddns.net;
    return 301 https://$host:8081$request_uri;
}

server {
    listen 8081 ssl;
    server_name yourdomain.ddns.net;

    root /var/www/html/rainloop;
    index index.php;

    ssl_certificate /etc/letsencrypt/live/yourdomain.ddns.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.ddns.net/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 50M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    access_log /var/log/nginx/rainloop-access.log;
    error_log /var/log/nginx/rainloop-error.log;
}


Enable the site:

sudo ln -s /etc/nginx/sites-available/rainloop-https /etc/nginx/sites-enabled/rainloop-https
sudo nginx -t
sudo systemctl reload nginx

3. Configure Postfix (SMTP)

Make sure Postfix is installed and running:

sudo apt install postfix
sudo systemctl enable postfix
sudo systemctl start postfix


Configure Postfix for SASL authentication with Dovecot and enable broken_sasl_auth_clients = yes if needed.

Allow authenticated users to send mail.

4. Configure UFW (Firewall)
sudo ufw allow 22/tcp        # SSH
sudo ufw allow 8081/tcp      # HTTP/HTTPS for Rainloop
sudo ufw allow 25/tcp        # SMTP
sudo ufw allow 587/tcp       # Submission (SMTP Auth)
sudo ufw enable
sudo ufw status

5. Access Rainloop

Webmail portal: https://yourdomain.ddns.net:8081/

Admin portal: https://yourdomain.ddns.net:8081/?admin (default login: admin / password 12345—change immediately)

6. Tips

Always edit /etc/nginx/sites-available/rainloop-https and reload Nginx after changes.

Back up your configuration before making edits:

sudo cp /etc/nginx/sites-available/rainloop-https /etc/nginx/sites-available/rainloop-https.bak_$(date +%F)


Keep your SSL certificate updated with Certbot:

sudo certbot renew --dry-run

7. Known Issues

If PHP-FPM isn’t running, Rainloop will show 502 Bad Gateway.

Ensure Postfix is active for sending mail; otherwise, the webmail will show Can’t send message errors.

Only one server block can listen on the same port for the same domain. Use HTTP → HTTPS redirect within a single configuration.

8. References

Rainloop Official Documentation

Nginx HTTPS Configuration

Postfix SASL Authentication
