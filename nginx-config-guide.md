#NGINX Sample Config Guide
###Scenario:
NGINX 1.9.5 as a proxy server with Gunicorn as the web server. Load balancing implemented but only for a certain path.

###TO DO:
* ~~update to a scenario where SSL is involved~~
* ~~update to a scenario that involves load balancing [https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)~~
* ~~update to follow [https://www.digitalocean.com/community/tutorials/how-to-optimize-nginx-configuration](https://www.digitalocean.com/community/tutorials/how-to-optimize-nginx-configuration)~~
* ~~update to use [https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-14-04)~~
* ~~add a scenario using uwsgi as the web server.~~


### nginx.conf
```bash
...
# worker_processes should be the same as the number of course of the server.
worker_processes 1;

# This means we can server 1024 clients/second. Please check what value this should be using
# ulimit -n
worker_connections 1024;

# gzip files aside from html so it can be transferred faster to the client.
# The client will handle the decompression. JPEG and PNG should not be
# included for gzip since they are already compressed by nature.
# Current configs are the default.
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_min_length 256;
gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;

# increase server name size if some of your domain names are really long
server_names_hash_bucket_size 64;
...
```

### example.conf
```bash
upstream example_app_server {
  # fail_timeout=0 means we always retry an upstream even if it failed
  # to return a good HTTP response (in case the Unicorn master nukes a
  # single worker for timing out).

  #this is where the request is redirected from port 80.
  server unix:/webapps/example_app/run/gunicorn.sock fail_timeout=0; 
  
}

upstream backend_hosts {
    # Servers for load balancing. The current balancing algorithm
    # is round robin but that can be modified.
    server host1.example.com;
    server host2.example.com;
    server host3.example.com;
}

upstream example_uwsgi {
    server unix:/wwebapps/example_app/run/uwsgi.sock fail_timeout=0;
}

server {
​
    #will listen to requests from port 80. ex: 127:0.0.1:80
    listen 80;
    
    # This will only be considered if there are multiple server block directive lisening to 80.
    # If nothing matches, nginx will get the first written server block or the one with a 
    # default_server.

    # Optimize search by not using regex, and wildcard names without an asterisk
    # http://nginx.org/en/docs/http/server_names.html
    server_name example.com  www.example.com  *.example.com;
    
    # Status 301 means that the resource (page) is moved permanently to a new location.
    # The client/browser should not attempt to request the original location but use
    # the new location from now on.
    
    # Status 302 means that the resource is temporarily located somewhere else,
    # and the client/browser should continue requesting the original url.
    
    # $server_name - name of the server which accepted the request.
    # $request_uri - full original request URI (with arguments).
    return 301 https://$server_name$request_uri;
}

server {
    # ssl, by standard listens at port 443.
    # http2 was just implemented on NGINX 1.9.5 so this will crash on lower versions.
    listen 443 ssl http2;
    include snippets/ssl-example.com.conf;
    include snippets/ssl-params.conf;
  
    # The default is 1mb which is believed to be reasonably high for non-upload use cases,
    # and reasonably low to prevent DoS by uploading large files. 
    # Currently set to 4G which I think is bad.
    client_max_body_size 4G; 
    
    access_log /webapps/example_app/logs/nginx-access.log;
    error_log /webapps/example_app/logs/nginx-error.log;
    
    # the NGINX webroot plugin works by placing a special file in the
    # .well-known/acme-challenge/ used to request an SSL Certificate.
    
    # The tilde instructs nginx to perform a case-sensitive regular expression match,
    # instead of a straight string comparison.
    root /var/www/html;
    location ^~ /.well-known/acme-challenge/ {
        allow all;
    }

    # The default is 1mb which is believed to be reasonably high for non-upload use cases,
    # and reasonably low to prevent DoS by uploading large files. 
    # Currently set to 4G which I think is bad.
    client_max_body_size 4G; 

    # Nginx checks the uri of the request. if it has /static/ e.g.,
    # http://localhost/static/example.png, 
    # then it will look for the location block matching that.
  
    # Location matching priority is from longest to shortest prefix ex: 
    # http://localhost/static/example.png matches /static/ and / but since /static/ is longer then 
    # it will be matched first.
  
    # Alias specifies the path to be called. ex: 
    # http://localhost/static/dog/cat/example.png will call 
    # /webapps/example_app/static/dog/cat/example.png
  
    # Root is the same as alias but it includes the location parameter part. 
    # ex: http://localhost/media/dog/cat/example.png 
    # will call /webapps/example_app/media/media/dog/cat/example.png
    location /static/ {
        alias   /webapps/example_app/static/;
    }
    
    location /media/ {
        alias   /webapps/example_app/media/;
    }

    location / {
        root /webapps/example_app/;
        # first try if $uri exists, it not, then redirect to location
        # @proxy
        try_files $uri @proxy;
        
        location ~* .(jpg|jpeg|png|gif|ico|css|js)$ {
            # Turn off logging for this location block.
            access_log off;
            # Add expire headers to requested files that don't change too much.
            # max means itll expire at some date at 2037.
            expires max;
        }
    }

    location @proxy {
        # Contains the IP address of the client.
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # pass the Host: header from the client.
        proxy_set_header HOST $host;
 
        # The X-Forwarded-Proto request header helps you identify the protocol (HTTP or HTTPS)
        # that a client used to connect to your server.
        proxy_set_header X-Forwarded-Proto $scheme;

        # This is set to the IP address of the client so that the proxy can correctly make 
        # decisions or log based on this information
        proxy_set_Header X-Real-IP $remote_addr;

        # If a request goes through multiple proxies, the clientIPAddress in the X-Forwarded-For
        # request header is followed by the IP addresses of each successive proxy that the
        # request goes through before it reaches your load balancer. Therefore, the right-most
        # IP address is the IP address of the most recent proxy and the left-most IP address
        # is the IP address of the originating client
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        #proxy_pass redirects the request from 80 to the specified upstream.
        proxy_pass http://example_app_server; 
    }
    
    # As of now, nothing is calling is this location block but we only include it here
    # for reference.
    location @uwsgi {
        # uwsgi params is used mainly for convenience and includes the following parameters:
        # uwsgi_param QUERY_STRING $query_string;
        # uwsgi_param REQUEST_METHOD $request_method;
        # uwsgi_param CONTENT_TYPE $content_type;
        # uwsgi_param CONTENT_LENGTH $content_length;
        # uwsgi_param REQUEST_URI $request_uri;
        # uwsgi_param PATH_INFO $document_uri;
        # uwsgi_param DOCUMENT_ROOT $document_root;
        # uwsgi_param SERVER_PROTOCOL $server_protocol;
        # uwsgi_param REMOTE_ADDR $remote_addr;
        # uwsgi_param REMOTE_PORT $remote_port;
        # uwsgi_param SERVER_ADDR $server_addr;
        # uwsgi_param SERVER_PORT $server_port;
        # uwsgi_param SERVER_NAME $server_name;
        include uwsgi_params;
        
        uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;
        uwsgi_param HOST $host;
        uwsgi_param X-Forwarded-Proto $scheme;
        uwsgi_param X-Real-IP $remote_addr;
        uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for;

        uwsgi_pass chefxchange-uwsgi;
    
    
    # Sample location for load balancing proxied connections. Not sure if this should also
    # have the X headers or that could be better set in the receiving servers.
    location /proxy-me {
        proxy_pass http://backend_hosts;
    }

    # Error pages
    error_page 500 502 503 504 /500.html;
    location = /500.html {
        root /webapps/example_app/static/;
    }
}
```

### snippets/ssl-example.com.conf
```bash
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```

### snippets/ssl-params.conf
```bash
# from https://cipherli.st/
# and https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html

ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
#add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;


# This should be generate following this guide:
# https://www.digitalocean.com/community/tutorials
# /how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04
ssl_dhparam /etc/ssl/certs/dhparam.pem;
```

###Reference:
* [https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/](https://www.nginx.com/resources/wiki/start/topics/tutorials/config_pitfalls/#)
* [https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)
* [https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)
* [https://cipherli.st/](https://cipherli.st/)
* [https://www.digitalocean.com/community/tutorials/how-to-optimize-nginx-configuration](https://www.digitalocean.com/community/tutorials/how-to-optimize-nginx-configuration)
* [https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-14-04](https://www.digitalocean.com/community/tutorials/how-to-add-the-gzip-module-to-nginx-on-ubuntu-14-04)
* [https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-set-up-nginx-server-blocks-virtual-hosts-on-ubuntu-16-04)
* [http://charles.lescampeurs.org/2008/11/14/fix-nginx-increase-server_names_hash_bucket_size](http://charles.lescampeurs.org/2008/11/14/fix-nginx-increase-server_names_hash_bucket_size)
