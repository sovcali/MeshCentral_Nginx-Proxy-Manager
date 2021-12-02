# Meshcentral_Nginx-Proxy-Manager

I use these config to make MeshCentral work behind Nginx Proxy Manager, with Docker on AWS.

This is not a guide on how to make any of this services work, I'm just sharing my config files so maybe they could be useful to someone.

First of all, you need to have a running instance of Docker and Nginx Proxy Manager, in my case I use Portainer to make things easier but thats up to you.

* For MeshCentral I use jamesits/meshcentral2 image, with DB Mongo:latest. 
* For Nginx Proxy Manager I use jc21/nginx-proxy-manager:latest with DB jc21/mariadb-aria:latest. 
* Portainer is version 2.6.2.

Once you have all this running properly use these settings.

## MeshCentral config.json

Fields you need to change with your own data:
* "MongoDb"
* "cert"
* "TLSOffload"
* "certUrl"

> 
```json
{
  "$schema": "http://info.meshcentral.com/downloads/meshcentral-config-schema.json",
  "settings": {
    "MongoDb": "mongodb://LOCAL_IP_MONGO_CONTAINER:27017/meshcentral",
    "cert": "YOUR-DOMAIN.com",
    "WANonly": true,
    "_LANonly": true,
    "_sessionKey": "MyReallySecretPassword1",
    "port": 4430,
    "AliasPort": 443,
    "RedirPort": 800,
    "_redirAliasPort": 80,
    "AgentPong": 300,
    "TLSOffload": "LOCAL_IP-NGINX-CONTAINER",
    "SelfUpdate": false,
    "AllowFraming": "false",
    "WebRTC": "true"
  },
  "domains": {
        "": {
        "_title": "MyServer",
    "_title2": "Servername",
    "_minify": true,
    "NewAccounts": "false",
        "_userNameIsEmail": true,
    "certUrl": "https://LOCAL_IP-NGINX-CONTAINER:443"
        }
  },
  "_letsencrypt": {
    "__comment__": "Requires NodeJS 8.x or better, Go to https://letsdebug.net/ first before>",
    "_email": "myemail@mydomain.com",
    "_names": "myserver.mydomain.com",
        "production": false
  }
}
```
---
## Nginx Proxy Manager .conf

You need to set up your domain with NPM and have a SSL certificate so it can create this next file and then you may modify it.

Once done that, find conf file  under ../data/nginx/proxy_host/, filename may vary but usually is a "number".conf

Fields to change with your own data:

* "set $server"
* Both "server_name"
* Both "proxy_pass"
* "ssl_certificate"
* "ssl_certificate_key"

Yes, "Asset Caching" is commented because it does'nt work with it enabled. (Dont ask me why)

Also "#Proxy!" section is commented so you dont duplicate headers.

These headers are taken from the User Guide provided by MeshCentral.

>
```conf
# ------------------------------------------------------------
# YOUR.DOMAIN.com
# ------------------------------------------------------------


server {
  set $forward_scheme https;
  set $server         "LOCAL_IP-NGNIX-CONTAINER";
  set $port           8086;

listen 80;
listen [::]:80;

server_name YOURDOMAIN.com;

location / {
proxy_pass http://LOCAL_IP-MESHCENTRAL-CONTAINER:800/;
proxy_http_version 1.1;

 # Inform MeshCentral about the real host, port and protocol
 proxy_set_header X-Forwarded-Host $host:$server_port;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 proxy_set_header X-Forwarded-Proto $scheme;
 }
 }


#https

server {
listen 443 ssl http2;
listen [::]:443;

server_name YOURDOMAIN.com;

# MeshCentral uses long standing web socket connections, set longer timeouts.
 proxy_send_timeout 330s;
 proxy_read_timeout 330s;


  # Let's Encrypt SSL
  include conf.d/include/letsencrypt-acme-challenge.conf;
  include conf.d/include/ssl-ciphers.conf;
  ssl_certificate /etc/letsencrypt/live/YOURDOMAIN/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/YOURDOMAIN/privkey.pem;



# Asset Caching
#  include conf.d/include/assets.conf;


  # Block Exploits
  include conf.d/include/block-exploits.conf;


    # Force SSL
    include conf.d/include/force-ssl.conf;


  access_log /data/logs/proxy-host-1_access.log proxy;
  error_log /data/logs/proxy-host-1_error.log warn;


  location / {

proxy_pass http://LOCAL_IP-MESHCENTRAL-CONTAINER:4430/;
proxy_http_version 1.1;

 # Allows websockets over HTTPS.
 proxy_set_header Upgrade $http_upgrade;
 proxy_set_header Connection "upgrade";
 proxy_set_header Host $host;

 # Inform MeshCentral about the real host, port and protocol
 proxy_set_header X-Forwarded-Host $host:$server_port;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 proxy_set_header X-Forwarded-Proto $scheme;




    # Proxy!
#    include conf.d/include/proxy.conf;

}

 # Custom
  include /data/nginx/custom/server_proxy[.]conf;

}
```
---

Thats it, with these two files I'm able to have MeshCentral running behind Nginx Proxy Manager without any issue, sorry if you dont have success with these settings, I wont be offering further support.