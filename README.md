#!/bin/bash
set -e

OUT_DIR="$HOME/lab6-outputs"
mkdir -p "$OUT_DIR"

# nginx
sudo apt-get update -y
sudo apt-get install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx

sudo tee /etc/nginx/sites-available/site1.local > /dev/null <<EOF
server {
    listen 80;
    server_name site1.local;
    root /var/www/site1;
    index index.html;
    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOF

sudo tee /etc/nginx/sites-available/app.local > /dev/null <<EOF
server {
    listen 80;
    server_name app.local;
    root /var/www/app;
    index index.html;
    location / {
        try_files \$uri \$uri/ =404;
    }
}
EOF

sudo mkdir -p /var/www/site1 /var/www/app
echo "<h1>Site1</h1>" | sudo tee /var/www/site1/index.html
echo "<h1>App</h1>"   | sudo tee /var/www/app/index.html

sudo ln -sf /etc/nginx/sites-available/site1.local /etc/nginx/sites-enabled/
sudo ln -sf /etc/nginx/sites-available/app.local /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# firewall
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw allow 2222/tcp
sudo ufw allow 'Nginx Full'
sudo ufw --force enable
sudo ufw status verbose | tee "$OUT_DIR/ufw_status.txt"

# fail2ban
sudo apt-get install -y fail2ban
sudo tee /etc/fail2ban/jail.local > /dev/null <<EOF
[DEFAULT]
bantime = 10m
findtime = 10m
maxretry = 5
backend = systemd

[sshd]
enabled = true
port = 2222
logpath = /var/log/auth.log
maxretry = 5

[nginx-http-auth]
enabled = true

[nginx-botsearch]
enabled = true
EOF
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
sudo fail2ban-client status | tee "$OUT_DIR/fail2ban_status.txt"

# security
sudo tee /etc/nginx/conf.d/security.conf > /dev/null <<EOF
server_tokens off;
add_header X-Content-Type-Options "nosniff";
add_header X-Frame-Options "SAMEORIGIN";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "no-referrer-when-downgrade";
add_header Content-Security-Policy "default-src 'self'";
EOF
sudo nginx -t
sudo systemctl reload nginx

# reverse proxy
nohup node -e "require('http').createServer((req,res)=>res.end(JSON.stringify({service:'API Service',port:3002}))).listen(3002)" &
nohup node -e "require('http').createServer((req,res)=>res.end(JSON.stringify({service:'Service2',port:3003}))).listen(3003)" &

sudo tee /etc/nginx/sites-available/api.local > /dev/null <<EOF
server {
    listen 80;
    server_name api.local;
    location / {
        proxy_pass http://127.0.0.1:3002;
    }
}
EOF

sudo tee /etc/nginx/sites-available/service2.local > /dev/null <<EOF
server {
    listen 80;
    server_name service2.local;
    location / {
        proxy_pass http://127.0.0.1:3003;
    }
}
EOF

sudo ln -sf /etc/nginx/sites-available/api.local /etc/nginx/sites-enabled/
sudo ln -sf /etc/nginx/sites-available/service2.local /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

curl api.local | tee "$OUT_DIR/curl_api.txt"
curl service2.local | tee "$OUT_DIR/curl_service2.txt"

# rate limit
sudo tee /etc/nginx/conf.d/ratelimit.conf > /dev/null <<EOF
limit_req_zone \$binary_remote_addr zone=one:10m rate=5r/s;
limit_req_zone \$binary_remote_addr zone=admin:10m rate=1r/s;
EOF

sudo tee /etc/nginx/sites-available/api.local > /dev/null <<EOF
server {
    listen 80;
    server_name api.local;
    location / {
        limit_req zone=one burst=10 nodelay;
        proxy_pass http://127.0.0.1:3002;
    }
    location /admin {
        limit_req zone=admin burst=3 nodelay;
        proxy_pass http://127.0.0.1:3002;
    }
}
EOF

sudo nginx -t
sudo systemctl reload nginx

# monitoring
free -h    | tee "$OUT_DIR/snapshot_free.txt"
df -h      | tee "$OUT_DIR/snapshot_df.txt"
ss -tln    | tee "$OUT_DIR/snapshot_ss.txt"
uptime     | tee "$OUT_DIR/snapshot_uptime.txt"

sudo logrotate -f /etc/logrotate.conf
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10 | tee "$OUT_DIR/top_ips.txt"
grep " 404 " /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -nr | head -10 | tee "$OUT_DIR/top_404.txt"
awk -F\" '{print $6}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10 | tee "$OUT_DIR/top_agents.txt"

# deploy
if [ -x /usr/local/bin/deploy_site1.sh ]; then
  sudo /usr/local/bin/deploy_site1.sh | tee "$OUT_DIR/deploy_log.txt"
fi

# backup
if [ -x /usr/local/bin/backup_site1.sh ]; then
  crontab -l | tee "$OUT_DIR/cron_jobs.txt"
fi

echo "=== done ==="
