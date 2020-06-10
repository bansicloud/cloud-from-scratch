[ Modified 2020.06.10 ] [ untested ]

# Nginx
A simple static file setup using nginx.

Nginx (pronounced "engine-x") is an open source reverse proxy server for HTTP, HTTPS, SMTP, POP3, and IMAP protocols, as well as a load balancer, HTTP cache, and a web server (origin server). The nginx project started with a strong focus on high concurrency, high performance and low memory usage. It is licensed under the 2-clause BSD-like license and it runs on Linux, BSD variants, Mac OS X, Solaris, AIX, HP-UX, as well as on other *nix flavors. It also has a proof of concept port for Microsoft Windows.

Make static content directory
```
$ mkdir -p ~/static/example.com
$ cd ~/static/example.com
```

Edit index.html
```
$ vim www/index.html
Hello!
```

Start Sever
```
$ sudo docker run -d --restart=always --name nginx_example_com --network=master  -v /home/pi/static/www/example.com:/usr/share/nginx/html:ro nginx
```

Map app into your domain
```
$ cd ~
$ vim Caddyfile
```

Add route to your domain
```
http://example.com {

# add this bit ####################

route /example* {
		redir /example /example/
		uri strip_prefix /example
		reverse_proxy nginx_example_com:80
    }
    
###################################    
    
}
```

Restart Caddy
```
$ docker restart caddy_web_server
```

browse 	[http://example.com/example]  --- Hello!