#NGINX Sample Config Guide
###Scenario:
Nginx as a proxy server with Gunicorn as the web server.

###TO DO:
* update to a scenario where SSL is involved
* update to a scenario that involves load balancing [https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)
* add a scenario using uwsgi as the web server.

```bash
upstream example_app_server {
  # fail_timeout=0 means we always retry an upstream even if it failed
  # to return a good HTTP response (in case the Unicorn master nukes a
  # single worker for timing out).

  #this is where the request is redirected from port 80.
  server unix:/webapps/example_app/run/gunicorn.sock fail_timeout=0; 
  
}
​
server {
​
    #will listen to requests from port 80. ex: 127:0.0.1:80
    listen   80;
    
    # This will only be considered if there are multiple server block directive lisening to 80.
    # If nothing matches, nginx will get the first written server block or the one with a 
    # default_server.
    server_name .example.com www.example.com example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    include snippets/ssl-example.com.conf;
    include snippets/ssl-params.conf;
  
    # The default is 1mb which is believed to be reasonably high for non-upload use cases,
    # and reasonably low to prevent DoS by uploading large files. 
    # Currently set to 4G which I think is bad.
    client_max_body_size 4G; 
    
    access_log /webapps/example_app/logs/nginx-access.log;
    error_log /webapps/example_app/logs/nginx-error.log;
  
    root /var/www/html;
    location ^~ /.well-known/acme-challenge/ {
        allow-all;
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
​
    location / {
        # Contains the IP address of the client.
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
​
        # The X-Forwarded-Proto request header helps you identify the protocol (HTTP or HTTPS)
        # that a client used to connect to your server.
        # proxy_set_header X-Forwarded-Proto https;
​
        # pass the Host: header from the client.
        proxy_set_header Host $host;
​
        # This checks if the request is not a request for a file.
        if (!-f $request_filename) {
            #proxy_pass redirects the request from 80 to the specified upstream.
            proxy_pass http://example_app_server; 
            break;
        }
    }
​
    # Error pages
    error_page 500 502 503 504 /500.html;
    location = /500.html {
        root /webapps/example_app/static/;
    }
}
