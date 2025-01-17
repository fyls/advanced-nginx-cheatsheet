# advanced-nginx-cheatsheet

**Table of content**
<!-- TOC -->
- [Typical configuration](#config-file)
  - [listen-ip](#listen-ip)
  - [listen-ip](#listen-host)
  - [Static_Assets](#Static_Assets)
  - [Set_variable](#Set_variable)
  - [Redirect](#Redirect)
  - [Reverse_Proxy](#Reverse_Proxy)
  - [Load_Balancing](#Load_Balancing)
  - [Let’s_Encrypt](#Let’s_Encrypt)
- [Nginx Performance](#nginx-performance)
  - [Load-Balancing](#load-balancing)
    - [php-fpm Unix socket](#php-fpm-unix-socket)
    - [php-fpm TCP](#php-fpm-tcp)
    - [HTTP load-balancing](#http-load-balancing)
  - [WordPress Fastcgi cache](#wordpress-fastcgi-cache)
    - [mapping fastcgi_cache_bypass conditions](#mapping-fastcgi_cache_bypass-conditions)
    - [Define fastcgi_cache settings](#define-fastcgi_cache-settings)
    - [fastcgi_cache vhost example](#fastcgi_cache-vhost-example)
- [Nginx as a Proxy](#nginx-as-a-proxy)
  - [Simple Proxy](#simple-proxy)
  - [Proxy in a subfolder](#proxy-in-a-subfolder)
  - [Proxy keepalive for websocket](#proxy-keepalive-for-websocket)
  - [Reverse-Proxy for Apache](#reverse-proxy-configuration-to-handle-static-files-and-pass-other-requests-to-apache)
- [Nginx Security](#nginx-security)
  - [Denying access](#denying-access)
    - [common backup and archives files](#common-backup-and-archives-files)
    - [Deny access to hidden files & directory](#deny-access-to-hidden-files--directory)
  - [Blocking common attacks](#blocking-common-attacks)
    - [base64 encoded url](#base64-encoded-url)
    - [javascript eval() url](#javascript-eval-url)
- [Nginx SEO](#nginx-seo)
  - [robots.txt location](#robotstxt-location)
  - [Make a website not indexable](#make-a-website-not-indexable)
- [Nginx Media](#nginx-media)
  - [MP4 stream module](#mp4-stream-module)
  - [WebP images](#webp-images)

<!-- /TOC -->
## Typical configuration
### config-file
FIND OUT: what happens if we get a port 443 ssl request for a hostname that we’re not serving? Does nginx reject it completely, or try to serve it with some existing hostname configuration?

This started from https://ssl-config.mozilla.org but is heavily modified.

Put this at
/etc/nginx/conf.d/00-default-vhost.conf
It returns a 410 for any port 80 request for a domain name we're not serving with
a more specific configuration.
#### listen-ip
```nginx
server {
  listen 80 default_server;
  listen [::]:80 default_server;

  server_name _;
  return 410;
  log_not_found off;
  server_tokens off;
}
```
#### listen-host
```nginx
server {
  # Listen to yourdomain.com
  server_name yourdomain.com;

  # Listen to multiple domains
  server_name yourdomain.com www.yourdomain.com;

  # Listen to all domains
  server_name *.yourdomain.com;

  # Listen to all top-level domains
  server_name yourdomain.*;

  # Listen to unspecified Hostnames (Listens to IP address itself)
  server_name "";
```
  
Now for each site mysite.example.com that you want to serve…

#### Static_Assets
```nginx
server {
  listen 80;
  server_name yourdomain.com;

  location / {
          root /path/to/website;
  } 
}
```

#### Redirect
```nginx
server {
  listen 80;
  server_name www.yourdomain.com;
  return 301 http://yourdomain.com$request_uri;
}
```
```nginx
server {
  listen 80;
  server_name www.yourdomain.com;

  location /redirect-url {
     return 301 http://otherdomain.com;
  }
}
```
#### Reverse_Proxy
```nginx
server {
  listen 80;
  server_name yourdomain.com;

  location / {
     proxy_pass http://0.0.0.0:3000;
     # where 0.0.0.0:3000 is your application server (Ex: node.js) bound on 0.0.0.0 listening on port 3000
  }

}
```

#### Load_Balancing
```nginx
upstream node_js {
  server 0.0.0.0:3000;
  server 0.0.0.0:4000;
  server 123.131.121.122;
}

server {
  listen 80;
  server_name yourdomain.com;

  location / {
     proxy_pass http://node_js;
  }
}

```
#### Most_useful_variables
```nginx
$host
in this order of precedence: host name from the request line, or host name from the “Host” request header field, or the server name matching a request

$http_host
Value of the “Host:” header in the request (same as all $http_<headername> variables)

$https
“on” if connection operates in SSL mode, or an empty string otherwise

$request_method
request method, usually “GET” or “POST”

$request_uri
full original request URI (with arguments)

$scheme
request scheme, e.g. “http” or “https”

$server_name
name of the server which accepted a request

$server_port
port of the server which accepted a request
```
  
#### Variables_in_configuration_files
  
See above for “variables” that get set automatically for each request (and that we cannot modify).

The ability to set variables at runtime and control logic flow based on them is part of the rewrite module and not a general feature of nginx.

##### Set_variable:
```nginx
Syntax:     set $variable value;
Default:    —
Context:    server, location, if
```
“The value can contain text, variables, and their combination.” – but I have not yet found the documentation on how these can be “combined”.
Then use if etc.:

```nginx
Syntax:     if (condition) { rewrite directives... }
Default:    —
Context:    server, location2
```

#### Conditions can include:

* a variable name; false if the value of a variable is an empty string or “0”;
* comparison of a variable with a string using the “=” and “!=” operators;
* matching of a variable against a regular expression using the “~” (for case-sensitive matching) and “~*” (for case-insensitive matching) operators. Regular expressions can contain captures that are made available for later reuse in the $1..$9 variables. Negative operators “!~” and “!~*” are also available. If a regular expression includes the “}” or “;” characters, the whole expressions should be enclosed in single or double quotes.
* checking of a file existence with the “-f” and “!-f” operators;
* checking of a directory existence with the “-d” and “!-d” operators;
* checking of a file, directory, or symbolic link existence with the “-e” and “!-e” operators;
* checking for an executable file with the “-x” and “!-x” operators.

###### Examples:
```nginx
if ($http_user_agent ~ MSIE) {
    rewrite ^(.*)$ /msie/$1 break;
}

if ($http_cookie ~* "id=([^;]+)(?:;|$)") {
    set $id $1;
}

if ($request_method = POST) {
    return 405;
}

if ($slow) {
    limit_rate 10k;
}

if ($invalid_referer) {
    return 403;
}
```
Warning
You CANNOT put any directive you want inside the if, only rewrite directives like set, rewrite, return, etc.

Warning
The values of variables you set this way can ONLY be used in if conditions, and maybe rewrite directives; don’t try to use them elsewhere.

#### Let’s Encrypt
Based rather loosely on https://certbot.eff.org/lets-encrypt/pip-nginx.

Before you start, your site must already be on the internet accessible using all the domain names you want certificates for, at port 80, and without any automatic redirect to port 443. If that makes you paranoid, you can configure nginx to redirect 80 to 443 except for /.well-known/acme-challenge. Here’s an unsupported example:

```nginx
server {
  listen 80;

  location '/.well-known/acme-challenge' {
    root        /var/www/demo;
  }

  location / {
    if ($scheme = http) {
      return 301 https://$server_name$request_uri;
    }
}
```

Install certbot. Assuming Ubuntu, “sudo apt install certbot python3-certbot-nginx” should do it.

Run “sudo certbot certonly –nginx” and follow the instructions.

Set up automatic renewal. This will add a cron command to do it:

echo "0 0,12 * * * root /usr/bin/python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew -q" | sudo tee -a /etc/crontab > /dev/null
run “sudo certbot renew –dry-run” to test renewal

## Nginx Performance

### Load-Balancing

#### php-fpm Unix socket

```nginx
upstream php {
    least_conn;

    server unix:/var/run/php/php-fpm.sock;
    server unix:/var/run/php/php-two-fpm.sock;

    keepalive 5;
}
```

#### php-fpm TCP

```nginx
upstream php {
    least_conn;

    server 127.0.0.1:9090;
    server 127.0.0.1:9091;

    keepalive 5;
}
```

#### HTTP load-balancing

```nginx
# Upstreams
upstream backend {
    least_conn;

    server 10.10.10.1:80;
    server 10.10.10.2:80;
}

server {

    server_name site.ltd;

    location / {
        proxy_pass http://backend;
        proxy_redirect      off;
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### WordPress Fastcgi cache

#### mapping fastcgi_cache_bypass conditions

To put inside a configuration file in /etc/nginx/conf.d/

```nginx
# do not cache xmlhttp requests
map $http_x_requested_with $http_request_no_cache {
    default 0;
    XMLHttpRequest 1;
}
# do not cache requests for the following cookies
map $http_cookie $cookie_no_cache {
    default 0;
    "~*wordpress_[a-f0-9]+" 1;
    "~*wp-postpass" 1;
    "~*wordpress_logged_in" 1;
    "~*wordpress_no_cache" 1;
    "~*comment_author" 1;
    "~*woocommerce_items_in_cart" 1;
    "~*woocommerce_cart_hash" 1;
    "~*wptouch_switch_toogle" 1;
    "~*comment_author_email_" 1;
}
# do not cache requests for the following uri
map $request_uri $uri_no_cache {
    default 0;
    "~*/wp-admin/" 1;
    "~*/wp-[a-zA-Z0-9-]+.php" 1;
    "~*/feed/" 1;
    "~*/index.php" 1;
    "~*/[a-z0-9_-]+-sitemap([0-9]+)?.xml" 1;
    "~*/sitemap(_index)?.xml" 1;
    "~*/wp-comments-popup.php" 1;
    "~*/wp-links-opml.php" 1;
    "~*/wp-.*.php" 1;
    "~*/xmlrpc.php" 1;
}
# do not cache request with args (like site.tld/index.php?id=1)
map $query_string $query_no_cache {
    default 1;
    "" 0;
}
# map previous conditions with the variable $skip_cache
map $http_request_no_cache$cookie_no_cache$uri_no_cache$query_no_cache $skip_cache {
    default 1;
    0000 0;
}
```

#### Define fastcgi_cache settings

To put inside another configuration file in /etc/nginx/conf.d

```nginx
# FastCGI cache settings
fastcgi_cache_path /var/run/nginx-cache levels=1:2 keys_zone=WORDPRESS:360m inactive=24h max_size=256M;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_use_stale error timeout invalid_header updating http_500 http_503;
fastcgi_cache_methods GET HEAD;
fastcgi_buffers 256 32k;
fastcgi_buffer_size 256k;
fastcgi_connect_timeout 4s;
fastcgi_send_timeout 120s;
fastcgi_busy_buffers_size 512k;
fastcgi_temp_file_write_size 512K;
fastcgi_param SERVER_NAME $http_host;
fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
fastcgi_keep_conn on;
fastcgi_cache_lock on;
fastcgi_cache_lock_age 1s;
fastcgi_cache_lock_timeout 3s;
```

To work with cookies, you can edit the fastcgi_cache_key. Cookie can be added with variable `$cookie_{COOKIE_NAME}`. For example, the WordPress plugin Polylang use a cookie named `pll_language`, so the directive fastcgi_cache_key would be :

```nginx
fastcgi_cache_key "$scheme$request_method$host$request_uri$cookie_pll_language";
```

#### fastcgi_cache vhost example

```nginx
server {

    server_name domain.tld;

    access_log /var/log/nginx/domain.tld.access.log;
    error_log /var/log/nginx/domain.tld.error.log;

    root /var/www/domain.tld/htdocs;
    index index.php index.html index.htm;

    # add X-fastcgi-cache header to see if requests are cached
    add_header X-fastcgi-cache $upstream_cache_status;

    # default try_files directive for WP 5.0+ with pretty URLs
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }
    # pass requests to fastcgi with fastcgi_cache enabled
    location ~ \.php$ {
        try_files $uri =404;
        include fastcgi_params;
        fastcgi_pass php;
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 30m;
    }
    # block to purge nginx cache with nginx was built with ngx_cache_purge module
    location ~ /purge(/.*) {
        fastcgi_cache_purge WORDPRESS "$scheme$request_method$host$1";
        access_log off;
    }

}
```

## Nginx as a Proxy

### Simple Proxy

```nginx
location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_redirect      off;
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }
```

### Proxy in a subfolder

```nginx
location /folder/ { # The / is important!
        proxy_pass http://127.0.0.1:3000/;# The / is important!
        proxy_redirect      off;
        proxy_set_header    Host            $host;
        proxy_set_header    X-Real-IP       $remote_addr;
        proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    }
```

### Proxy keepalive for websocket

```nginx
# Upstreams
upstream backend {
    server 127.0.0.1:3000;
    keepalive 5;
}
# HTTP Server
server {
    server_name site.tld;
    error_log /var/log/nginx/site.tld.access.log;
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forward-Proto http;
        proxy_set_header X-Nginx-Proxy true;
        proxy_redirect off;
    }
}
```

### Reverse-Proxy For Apache

```nginx
server {

    server_name domain.tld;

    access_log /var/log/nginx/domain.tld.access.log;
    error_log /var/log/nginx/domain.tld.error.log;

    root /var/www/domain.tld/htdocs;

    # pass requests to Apache backend
    location / {
        proxy_pass http://backend;
    }
    # handle static files with a fallback
    location ~* \.(ogg|ogv|svg|svgz|eot|otf|woff|woff2|ttf|m4a|mp4|ttf|rss|atom|jpe?g|gif|cur|heic|png|tiff|ico|zip|webm|mp3|aac|tgz|gz|rar|bz2|doc|xls|exe|ppt|tar|mid|midi|wav|bmp|rtf|swf|webp)$ {
        add_header "Access-Control-Allow-Origin" "*";
        access_log off;
        log_not_found off;
        expires max;
        try_files $uri @fallback;
    }
    # fallback to pass requests to Apache if files are not found
    location @fallback {
        proxy_pass http://backend;
    }
}
```

## Nginx Security

### Denying access

#### common backup and archives files

```nginx
location ~* "\.(old|orig|original|php#|php~|php_bak|save|swo|aspx?|tpl|sh|bash|bak?|cfg|cgi|dll|exe|git|hg|ini|jsp|log|mdb|out|sql|svn|swp|tar|rdf)$" {
    deny all;
}
```

#### Deny access to hidden files & directory

```nginx
location ~ /\.(?!well-known\/) {
    deny all;
}
```

### Blocking common attacks

#### base64 encoded url

```nginx
location ~* "(base64_encode)(.*)(\()" {
    deny all;
}
```

#### javascript eval() url

```nginx
location ~* "(eval\()" {
    deny all;
}
```

## Nginx SEO

### robots.txt location

```nginx
location = /robots.txt {
# Some WordPress plugin gererate robots.txt file
# Refer #340 issue
    try_files $uri $uri/ /index.php?$args @robots;
    access_log off;
    log_not_found off;
}
location @robots {
    return 200 "User-agent: *\nDisallow: /wp-admin/\nAllow: /wp-admin/admin-ajax.php\n";
}
```

### Make a website not indexable

```nginx
add_header X-Robots-Tag "noindex";

location = /robots.txt {
  return 200 "User-agent: *\nDisallow: /\n";
}
```

## Nginx Media

### MP4 stream module

```nginx
location /videos {
    location ~ \.(mp4)$ {
        mp4;
        mp4_buffer_size       1m;
        mp4_max_buffer_size   5m;
        add_header Vary "Accept-Encoding";
        add_header "Access-Control-Allow-Origin" "*";
        add_header Cache-Control "public, no-transform";
        access_log off;
        log_not_found off;
        expires max;
    }
}
```

### WebP images

Mapping conditions to display WebP images

```nginx
# serve WebP images if web browser support WebP
map $http_accept $webp_suffix {
   default "";
   "~*webp" ".webp";
}
```

Set conditional try_files to server WebP image :

- if web browser support WebP
- if WebP alternative exist

```nginx


# webp rewrite rules for jpg and png images
# try to load alternative image.png.webp before image.png
location /wp-content/uploads {
    location ~ \.(png|jpe?g)$ {
        add_header Vary "Accept-Encoding";
        add_header "Access-Control-Allow-Origin" "*";
        add_header Cache-Control "public, no-transform";
        access_log off;
        log_not_found off;
        expires max;
        try_files $uri$webp_suffix $uri =404;
    }
}
```
