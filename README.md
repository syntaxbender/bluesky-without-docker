
# bluesky custom install steps
This doc installs bskypds on your server on a subdomain without docker.

## prerequisites:
- node18(i prefer)
- a smtp server like gmail smtp for reset password etc.
## notes:
 - bskypds uses xrpc on node http backend.
 - bskypds uses websocket on node so configure apache/nginx as reverse socket mod.
 - you need to verify every user subdomain in browser(bsky.app ui)  like username.yoursubdomain.exampledomain.com. if this step is skipped, an "Invalid Handle" error will appear in the username field.
 - created this steps mined from https://github.com/bluesky-social/pds/blob/main/installer.sh

```bash
root$: apt install -y ca-certificates curl gnupg jq lsb-release openssl sqlite3 xxd
root$: sudo a2enmod proxy proxy_http proxy_wstunnel rewrite
root$: useradd -m -d /home/bskypds/ -s /bin/bash -U bskypds 
root$: nano /etc/group
```
```bash
#bskypds:x:1018:
bskypds:x:1018:www-data # add www-data user to bskypds group
```
```bash
root$: mkdir -p /home/bskypds/public_html
root$: chown -R bskypds:bskypds /home/bskypds 
root$: certbot certonly -a apache --agree-tos --no-eff-email --staple-ocsp --email info@domain.com -d domain.com,www.domain.com
root$: nano /etc/apache2/sites-available/bskypds-ssl.conf
```
```bash
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerAdmin info@exampledomain.com
    ServerName social.exampledomain.com
    ServerAlias *.social.exampledomain.com
    DocumentRoot /home/bskypds/public_html
	
	# websocket upgrade rules
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} ^websocket$ [NC] 
    RewriteCond %{HTTP:Connection} Upgrade [NC]
    RewriteRule /(.*)$ ws://127.0.0.1:3030/$1 [P,L]

	# reverse proxy rules
    ProxyPass /xrpc/ http://127.0.0.1:3030/xrpc/
    ProxyPassReverse /xrpc/ http://127.0.0.1:3030/xrpc/
    ProxyRequests off

	# userspace rules for path based verification(via .well-known/atproto-did file) like username.subdomain.domain.com
    <Directory /home/bskypds/public_html>
         Options -Indexes
         AllowOverride All
         Require all granted
     </Directory>
    <Location />
        Require all granted
    </Location>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    SSLCertificateFile /etc/letsencrypt/live/exampledomain.com-0001/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/exampledomain.com-0001/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>

```
```bash
root$: npm install -g pnpm
root$: nano /etc/systemd/system/pds.service
```
```bash
[Unit]
Description=atproto personal data server

[Service]
User=bskypds
Group=bskypds
WorkingDirectory=/home/bskypds/public_html/pds/service
ExecStart=/usr/bin/node --enable-source-maps index.js
Restart=on-failure
EnvironmentFile=/home/bskypds/public_html/pds/service/.env

[Install]
WantedBy=default.target

```
```bash
root$: sudo su bskypds
bskypds$: cd /home/bskypds/public_html
bskypds$: git clone https://github.com/bluesky-social/pds
bskypds$: cd /home/bskypds/public_html/pds/service
bskypds$: pnpm install --production --frozen-lockfile
bskypds$: mkdir -p data/blocks
bskypds$: nano .env
```
```bash
PDS_HOSTNAME="EXAMPLESUBDOMAIN.EXAMPLEDOMAIN.com"
PDS_JWT_SECRET="GENERATED SECRET"                                   # openssl rand --hex 16
PDS_ADMIN_PASSWORD="GENERATED KEY"                                  # openssl ecparam --name secp256k1 --genkey --noout --outform DER | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32
PDS_PLC_ROTATION_KEY_K256_PRIVATE_KEY_HEX="GENERATED KEY"   # openssl ecparam --name secp256k1 --genkey --noout --outform DER | tail --bytes=+8 | head --bytes=32 | xxd --plain --cols 32
PDS_DATA_DIRECTORY=./data2
PDS_BLOBSTORE_DISK_LOCATION=./data2/blocks
PDS_DID_PLC_URL=https://plc.directory
PDS_BLOB_UPLOAD_LIMIT=52428800
PDS_BSKY_APP_VIEW_URL=https://api.bsky.app
PDS_BSKY_APP_VIEW_DID=did:web:api.bsky.app
PDS_REPORT_SERVICE_URL=https://mod.bsky.app
PDS_REPORT_SERVICE_DID=did:plc:ar7c4by46qjdydhdevvrndac
PDS_CRAWLERS=https://bsky.network
LOG_ENABLED=truehome/bskypds/public_html/pds/pdsadmin
NODE_ENV=production
PDS_PORT=3030
PDS_EMAIL_SMTP_URL="smtp://YOURGMAILADDRESSFORSMTP@gmail.com:YOURGENERATEDAPPKEY@smtp.gmail.com:587"
PDS_EMAIL_FROM_ADDRESS="YOURGMAILADDRESS@gmail.com"
```
```bash
bskypds$: cd /home/bskypds/public_html/pds/pdsadmin
bskypds$: cp create-invite-code.sh invite_custom.sh
bskypds$: nano invite_custom.sh

```
```bash
#PDS_ENV_FILE=${PDS_ENV_FILE:-"/pds/pds.env"}               #replace this
PDS_ENV_FILE="/home/bskypds/public_html/pds2/service/.env"  #with this
```
```bash
bskypds$: chmod +x invite_custom.sh
bskypds$: exit
root$: systemctl daemon-reload
root$: systemctl enable pds
root$: systemctl start pds
root$: sudo su bskypds
bskypds$: ./invite_custom.sh

