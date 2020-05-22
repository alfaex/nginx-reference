# overview

random remarks

```
<- .png ->
<- .php ->        NGINX         <- .php ->  PHP FPM PROCESS
<- .text ->
```


```nginx
# |-- context/scope
http { 
  
#  |-- directive  
  server_name example.com 
}
```


Modules need to be set up on instalation

## NGINX service

```nginx
# start, stop and reload
nginx -s stop, quit, reopen, reloa

# check syntax
nginx -t 
```

## upstream logs (custom logs)

Go to a file, probably `nginx.conf` and define a new log format

```nginx
log_format rt_cache '$remote_addr ...';
```


Then add a new log using that format

```nginx
access_log   /var/log/nginx/example.com.access.log rt_cache;
```


## root 

the root path folder that nginx will be interpreting static requests *from*

`GET /images/cat.png` -> look for root + request

`root /etc/site/www` -> will go to `/etc/site/www/images/cat.png`

## location

prefix: `location /greet {`

anything that starting with '/greet', ex: '/greeting'

exact: `location = /greet {`

the path exact the text, only will match '/greet' and not '/greeting'

regex: `location ~ /greet[5-6] {`

**case sensitive**

for case insensitive use `~*`

anything with regex that your heart desire and not explode your head

will match only 'localhost/greet5' and 'localhost/greet6'

### location weight

more important to least
1. = exactly
2. ^~ forward match
3. ~ case sensitive regex
4. ~* case insensitive regex
5. prefix 

## query strings

nginx let get individual query strings using the prefix `$arg_`

`http://localhost/inspect?batata=doce`

is possible to get the batata value with `$arg_batata`

## rewrite

if the return use 3xx codes it accepts a location, otherwise it uses strings

`return 3xx` apply a 'REDIRECT' (it changes the url on the browser) 

`rewrite ` mutate the request internaly

**rewrite** when an uri is **re-writen**, it's **re-evaluated** again by nginx as a completely new request

rewrites cans use standard regex caption groups

## try_files

`try_files path1 path2 final` -> only the final results in a rewrite and re-evaluation

```nginx
server {

    root /sites/demo;
    
    try_files /files.jpg /greet /login;
    
}
```

it always try to serve '/sites/demo/files.jpg' if exists, serve it, don't matter the url requested.

if '/sites/demo/files.jpg' don't exists it tries to serve the **file** /greet, and for last do a rewrite to serve '/sites/demo/login'

if '/greet' is not the last it will fail even if exist a 'location /greet' because it will try to look for a file  called /greet

request: localhost/cebola will serve files.jpg

if don't exist will serve the file /greet

if don't exist will rewrite **internaly** to localhost/login

if a `location /login {` exists it will serve that content under the `localhost/cebola` url on the browser

### named locations

will not try to re write the request, it will do a direct call to the location, it uses `@`

`try_files file1 file2 @login`

`location @login { `

## Directive Types

**Array Directive** can be declared multiple times, like access_log. If a child context declare even once of the same, all the parent config of the same directive is overriten.

**Stand Directive** can be declared only once in a given context, like root. 

**Action directive** the one that invoke or break actions like a return or rewrite.


# Workers

main nginx process spaw workers processes, default it's 1 worker

It's good to have the same number of workers as cores

The directive `worker_processes auto;` does exactly that.

It sets the numbers of workers equals to core.

## worker_connections

Number of connections each workers process can accept.

By server limitation it has a number of files it can handle opened at once based on each cpu core

it's possible to check with `ulimit -n` 

Setting `worker_connections` to that number it will max out the server, and very important, this is the number of concurrent requests the server should be able to accept

```bash
     worker_processes
            X                =        max connections
    worker_connections
```



# buffers & timeouts

## buffering

process/worker read data into memory before writes into it destination

Ex:

**Request**

GET localhost:80 -> write that request to memory (if the buffer is to small for the amount of date been read writes some of it to disk )

**Response**

static file -> from disk to memory -> send data to client from memory

## Timeout

a cutoff time for a given event

ex:

if recieves a request from a client stop after a certain number of seconds (prevent fom a client send infinite amout of data) 



```nginx
 # Buffer size for POST submissions
  client_body_buffer_size 10K;
  client_max_body_size 8m;

  # Buffer size for Headers
  client_header_buffer_size 1k;

  # Max time to receive client headers/body
  client_body_timeout 12;
  client_header_timeout 12;

  # Max time to keep a connection open for
  keepalive_timeout 15;

  # Max time for the client accept/receive a response
  send_timeout 10;

  # Skip buffering for static files
  sendfile on;

  # Optimise sendfile packets
  tcp_nopush on;

```

# Dynamic Module {color:red}

`nginx -V` to see the enabled modules

use `load_module` on the *.conf 

and load the full path or the relative to the .conf file to the .so file

```nginx
load_module /usr/lib/nginx/modules/ndk_http_module.so;
```


# Headers

## Expires headers

https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers

```nginx

add_headers Cache-Control DIRECTIVE;

# expires is a shortcut for 'add_header Expires' for manipulating time
expires 1h;

```

# Compressed response

```nginx

http {

    # header that indicate that something change based on this header
    # ex: if the request send 'accept-encoding: gzip, deflate, br'
    # we can deliver the file gziped to that request
    add_header Very Accept-Encoding;

    # turn on compression for this context
    gzip on

    # the compression level going from 0 to 9
    # bigger number means more compression but is more taxing 
    # on the server
    gzip_comp_level 3;

    # what types of files to enable compression, use mime-types
    gzip_types text/javascript;
    gzip_types text/css;

}

```


# fastCGI micro_cache

On this example its on the http context to make it available to all servers

```nginx
http {
    # level is the depth of the directory for the cache
    # keys_zone 'name' of the cache so you could use in more than one place
    # with the key you set the size of that cache 
    # inactive how long to keep the cache before the last access
    fastcgi_cache_path /tmp/fastcgicache levels=1:2 keys_zone=ANY_NAME:100m inactive=60m;

    # base on this key the cache 'hash' will be created;
    fastcgi_cache_key "$scheme$request_method$host$request_uri";
}
```

On the location that we want the cache we can activate by the zone name

```nginx
location \.php$ {
    fastcgi_cache ANY_NAME;

    # the time to cache be valid for the type of request
    fastcgi_cache_valid 200 60m;
    fastcgi_cache_valid 404 10m;
}
```

usefull way to see if you hit the cache is with the use of `$upstream_cache_status`

```nginx
http {
    add_header X-Cache $upstream_cache_status;
}
```

It's possible to create cache exceptions to force to reload the cache

```nginx
server {
    set $no_cache 0;

    if ($request_method = POST){
        set $no_cache 1;
    }

    location \.php$ {
        
        # to miss the cache and do the normal flow of the request
        fastcgi_cache_bypass $no_cache;
        
        # to not cache this type of request;
        fastcgi_no_cache $no_cache;

    }
}
```

```nginx
```

```nginx
```

```nginx
```

```nginx
```

```nginx
```

```nginx
```