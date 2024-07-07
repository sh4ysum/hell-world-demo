## Hello-World

A simple hello-world NGINX server listening over 8080.

### Config

For this demo, I opted for a basic server configuration to test using a mapped port in the Kubernetes service to the listening port in the NGINX server.

There is the default endpoint served via `index.html` as stated in the config. There is also a `health` endpoint used for testing within the demo. 

```bash
server {
    listen       8080; 
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    ## REDIRECT SERVER ERROR PAGES TO THE STATIC PAGE /50X.HTML
    
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```