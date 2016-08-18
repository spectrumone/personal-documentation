Included here are notes to serve as guide on how NGINX works using Gunicorn as the web server. 

TO DO:
* update to a scenario where SSL is involved
* update to a scenario that involves load balancing [https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching](https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching)

```bash
upstream crescent_moon_app_server {
  # fail_timeout=0 means we always retry an upstream even if it failed
  # to return a good HTTP response (in case the Unicorn master nukes a
  # single worker for timing out).

  #this is where the request is redirected from port 80.
  server unix:/webapps/crescent_moon/run/gunicorn.sock fail_timeout=0; 
  
}
​
server {
​
    #will listen to requests from port 80. ex: 127:0.0.1:80
    listen   80;
    
    # This will only be considered if there are multiple server block directive lisening to 80.
    # If nothing matches, nginx will get the first written server block or the one with a default_server.
    server_name .crescent-moon.tk; 
​
    # The default is 1mb which is believed to be reasonably high for non-upload use cases,
    # and reasonably low to prevent DoS by uploading large files. Currently set to 4G which I think is bad.
    client_max_body_size 4G; 
​
    access_log /webapps/crescent_moon/logs/nginx-access.log;
    error_log /webapps/crescent_moon/logs/nginx-error.log;


# Nginx checks the uri of the request. if it has /static/ e.g.,  http://localhost/static/example.png, 
# then it will look for the location block matching that.

# Location matching priority is from longest to shortest prefix ex: 
# http://localhost/static/example.png matches /static/ and / but since /static/ is longer then it will be matched first.

# Alias specifies the path to be called. ex: 
# http://localhost/static/dog/cat/example.png will call /webapps/crescent_moon/static/dog/cat/example.png

# Root is the same as alias but it includes the location parameter part. ex: http://localhost/media/dog/cat/example.png 
# will call /webapps/crescent_moon/media/media/dog/cat/example.png
 
    location /static/ {
        alias   /webapps/crescent_moon/crescent_moon/static/;
    }
    
    location /media/ {
        alias   /webapps/crescent_moon/crescent_moon/media/;
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
            proxy_pass http://crescent_moon_app_server; 
            break;
        }
    }
​
    # Error pages
    error_page 500 502 503 504 /500.html;
    location = /500.html {
        root /webapps/crescent_moon/crescent_moon/static/;
    }
}
