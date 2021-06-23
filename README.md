# Snowstorm installation and configuration

## Server requirements
Installed on Ubuntu 20.04 LTS VPS with 4 vCPUs, 8GB RAM and 160GB SSD.

The server contains the Elasticsearch node, the Snowstorm java runtime and Nginx as a proxy.

## Versions

- Ubuntu 20.04 LTS
- Elasticsearch 7.7.0
- Snowstorm 7.0.1
- Java 11.0.11
- Apache Maven 3.6.3
- Nginx 1.18.0

## Modifications to code (optional)

To make the Swagger/API docs "a little" less generic. SecurityAndUriConfig.java:
```java
docket.apiInfo(new ApiInfo("Snowstorm - Scandinavian Sandbox", "A Scandinavian SNOMED CT sandbox environment for testing & learning. Courtesy of Snomed International at https://github.com/IHTSDO/snowstorm", version, null,
				new Contact("Oskar Hurtig", "https://hurtig.care", "oskar@hurtig.care"), "Snowstorm Apache 2.0", "http://www.apache.org/licenses/LICENSE-2.0"));

```
To change default header language. Config.java:
```java
public static final String DEFAULT_ACCEPT_LANG_HEADER = "no";
```

## Elasticsearch config

/etc/elasticsearch/jvm.options:

```
# Memory limits for Elasticsearch, very important to set these so it has enough memory.
-Xms4g
-Xmx4g
```

## Terminology import
Terminology is imported in two ways - from command line or from the Swagger interface once Snowstorm is booted. 

Import order:
1. [International edition into MAIN](https://github.com/IHTSDO/snowstorm/blob/master/docs/loading-snomed.md)
2. (Optional) [import national extensions](https://github.com/IHTSDO/snowstorm/blob/master/docs/updating-snomed-and-extensions.md) into MAIN/SNOMEDCT-xx
3. (Optional) import unreleased content as DELTA files 
4. (Optional) import standalone maps and refsets

## Dailybuild config
TBD 

## Nginx configuration

This is the nginx config for the domain where Snowstorm is accessible.

This is what it does:

- Caches requests (currently disabled)
- Requires a password for all write requests (POST, PATCH/PUT etc)
- Enable CORS
- Use certbot and Let's encrypt for SSL in Nginx

/etc/nginx/sites-available/snowstorm.hurtig.no:

```nginx
proxy_cache_path        /var/cache/nginx levels=1:2 keys_zone=snowstorm:10m max_size=10g
                        inactive=60m use_temp_path=off;

map $http_origin $cors_origin_header {
    default "*";
    "~(^|^http:\/\/)(localhost$|localhost:[0-9]{1,4}$)" "$http_origin";
    "https://snowstorm.hurtig.no" "$http_origin";
}

map $http_origin $cors_cred {
    default "";
    "~(^|^http:\/\/)(localhost$|localhost:[0-9]{1,4}$)" "true";
    "https://snowstorm.hurtig.no" "true";
}

map $http_origin $cors_methods {
    default "GET, OPTIONS, HEAD";
    "~(^|^http:\/\/)(localhost$|localhost:[0-9]{1,4}$)" "GET, POST, DELETE, PUT, OPTIONS, HEAD";
    "https://snowstorm.hurtig.no" "GET, POST, DELETE, PUT, OPTIONS, HEAD";
}

server {

    server_name snowstorm.hurtig.no;

    location / {
        limit_except GET HEAD OPTIONS {
                auth_basic "Snowstorm";
                auth_basic_user_file /etc/nginx/.htpasswd;
        }

        add_header "Access-Control-Allow-Origin" $cors_origin_header always;
        add_header "Access-Control-Allow-Credentials" $cors_cred always;
        add_header "Access-Control-Allow-Methods" $cors_methods always;
        add_header "Access-Control-Allow-Headers" "Authorization, Origin, X-Requested-With, Content-Type, Accept";

        if ($request_method = 'OPTIONS' ) {
            return 204 no-content;
        }

        # Enable or disable the cache by uncommenting/commenting the next line
#        proxy_cache                     snowstorm;
        add_header                      X-Cache-Status          $upstream_cache_status;
        proxy_ignore_headers            Cache-Control Expires;
        proxy_cache_valid               30d;
        proxy_cache_valid               any 1h;
        proxy_cache_use_stale           error timeout http_500 http_502 http_503 http_504;
        proxy_cache_background_update   on;
        proxy_cache_lock                on;

        proxy_pass                      http://localhost:8080;
        proxy_redirect                  off;
        proxy_set_header                Host                    $host;
        proxy_set_header                X-Real-IP               $remote_addr;
        proxy_set_header                X-Forwarded-For         $proxy_add_x_forwarded_for;
        proxy_max_temp_file_size        0;

        client_max_body_size            1024M;
        client_body_buffer_size         128k;

        proxy_connect_timeout           90;
        proxy_send_timeout              90;
        proxy_read_timeout              90;

        proxy_buffer_size               4k;
        proxy_buffers                   4 32k;
        proxy_busy_buffers_size         64k;
        proxy_temp_file_write_size      64k;
    }

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/snowstorm.hurtig.no/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/snowstorm.hurtig.no/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = snowstorm.hurtig.no) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    listen 80;
    listen [::]:80;
    server_name snowstorm.hurtig.no;
    return 404; # managed by Certbot
}
```

## Firewall ufw config

```
To                         Action      From
--                         ------      ----
22/tcp (OpenSSH)           ALLOW IN    Anywhere
80,443/tcp (Nginx Full)    ALLOW IN    Anywhere
8080                       DENY IN     Anywhere
22/tcp (OpenSSH (v6))      ALLOW IN    Anywhere (v6)
80,443/tcp (Nginx Full (v6)) ALLOW IN    Anywhere (v6)
8080 (v6)                  DENY IN     Anywhere (v6)
```

## Start snowstorm on boot

### Step 1:

Find your user defined services mine was at /etc/systemd/system/

### Step 2:

Create a text file with your favorite text editor name it snowstorm.service

### Step 3:
Input:
```
[Unit]
Description=snowstorm
Requires=elasticsearch.service
After=elasticsearch.service

[Service]
WorkingDirectory=/home/snowstorm
User=root
ExecStartPre=/bin/sleep 60
ExecStart=/usr/bin/java -Xms2g -Xmx2g -jar /home/snowstorm/snowstorm*.jar -jar --spring.config.location=/home/snowstorm/application-local.properties
StandardOutput=journal
StandardError=journal
SyslogIdentifier=snowstorm
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

Note: It's necessary to start elastic service before snowstorm. 

Also, readonly is configured in application-local.properties but can also be run as a flag/option. 

### Step 4:

Run your service:

```shell
$ systemctl daemon-reload # reloads the config
$ systemctl start snowstorm.service # starts the service
$ systemctl enable snowstorm.service # auto starts the service
$ systemctl disable snowstorm.service # stops autostart
$ systemctl stop snowstorm.service # stops the service
$ systemctl restart snowstorm.service # restarts the service
```

## Helpful commands

### Check logs
```shell
journalctl -u snowstorm.service
```
- appending -b will print only log messages for the current boot
- appending --no-pager will print full log, so you wont have to scroll
- appending -e will start the log at the end removing the need to scroll, but without printing the entire log beforehand.
- appending -f will follow (print) updates to the log

### Delete all Elasticsearch indices

```shell
curl -XDELETE 'http://localhost:9200/*'
```

