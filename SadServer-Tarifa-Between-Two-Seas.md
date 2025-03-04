# Tarifa: Between Two Seas

There are three Docker containers defined in the `docker-compose.yml` file: an `HAProxy` accepting connections on port `:5000` of the host, and `two nginx containers`, not exposed to the host.

The person who tried to set this up wanted to have HAProxy in front of the (backend or upstream) nginx containers load-balancing them but something is not working.

Running `curl localhost:5000` several times returns both `hello there from nginx_0` and `hello there from nginx_1` but only `nginx_0` is being served:

```shell
admin@i-0c6403c79fb500a19:~$ for i in `seq 1 6`; do curl localhost:5000; done
hello there from nginx_0
hello there from nginx_0
hello there from nginx_0
hello there from nginx_0
hello there from nginx_0
hello there from nginx_0
```

All containers seems to be up and running:

```shell
admin@i-0c6403c79fb500a19:~$ docker ps 
CONTAINER ID   IMAGE           COMMAND                  CREATED        STATUS          PORTS                                       NAMES
c79c9eb25143   haproxy:2.8.4   "docker-entrypoint.s…"   8 months ago   Up 56 seconds   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   haproxy
4052b12402a3   nginx:1.25.3    "/docker-entrypoint.…"   8 months ago   Up 56 seconds   80/tcp                                      nginx_1
2624cd20f3c4   nginx:1.25.3    "/docker-entrypoint.…"   8 months ago   Up 56 seconds   80/tcp                                      nginx_0
admin@i-0c6403c79fb500a19:~$ 
```

The `haproxy` config looks fine:

```shell
admin@i-0c6403c79fb500a19:~$ cat haproxy.cfg 
global
    daemon
    maxconn 256

defaults
    mode http
    default-server init-addr last,libc,none
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:5000
    default_backend nginx_backends

backend nginx_backends
    balance roundrobin
    server nginx_0 nginx_0:80 check
    server nginx_1 nginx_1:80 check
```

The configuration files of both `nginx` servers looks good but `nginx_1` was listening on `81` instead of `80`:

```
admin@i-0c6403c79fb500a19:~$ cat custom-nginx_0.conf 
server {
    listen 80;

    server_name localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }
}

admin@i-0c6403c79fb500a19:~$ cat custom-nginx_1.conf 
server {
    listen 81;

    server_name localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }
}
```

Fix it by replacing `81` with `80`. 

Looking at the `compose` file, we can see that `nginx_0` is part of the `frontend_network`, `nginx_1` is part of the `backend_network`, and `haproxy` is only part of the `frontend_network`:

```yaml
admin@i-0c6403c79fb500a19:~$ cat docker-compose.yml 
version: '3'

services:
  nginx_0:
    image: nginx:1.25.3
    container_name: nginx_0
    restart: always
    volumes:
      - ./custom_index/nginx_0:/usr/share/nginx/html
      - ./custom-nginx_0.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - frontend_network

  nginx_1:
    image: nginx:1.25.3
    container_name: nginx_1
    restart: always
    volumes:
      - ./custom_index/nginx_1:/usr/share/nginx/html
      - ./custom-nginx_1.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - backend_network

  haproxy:
    image: haproxy:2.8.4
    container_name: haproxy
    restart: always
    ports:
      - "5000:5000"
    depends_on:
      - nginx_0
      - nginx_1
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
    networks:
      - frontend_network

networks:
  frontend_network:
    driver: bridge
  backend_network:
    driver: bridge
admin@i-0c6403c79fb500a19:~$ 
```

Fix the `compose` file by adding the `haproxy` container to both `frontend_network` and `backend_network`. Also, change the `nginx_0` container to be part of `backend_network` instead of `frontend_network`, even though it would work without this modification. Finally, restart all the containers to ensure they join the updated networks:

```shell
admin@i-0c6403c79fb500a19:~$ docker compose down
[+] Running 5/5
 ✔ Container haproxy               Removed                                                          0.2s 
 ✔ Container nginx_0               Removed                                                          0.3s 
 ✔ Container nginx_1               Removed                                                          0.3s 
 ✔ Network admin_frontend_network  Removed                                                          0.2s 
 ✔ Network admin_backend_network   Removed                                                          0.1s 


admin@i-0c6403c79fb500a19:~$ docker compose up -d
[+] Running 5/5
 ✔ Network admin_frontend_network  Created                                                          0.1s 
 ✔ Network admin_backend_network   Created                                                          0.1s 
 ✔ Container nginx_0               Started                                                          0.1s 
 ✔ Container nginx_1               Started                                                          0.1s 
 ✔ Container haproxy               Started                                                          0.0s 
```

Now `haproxy` is able to reach both `nginx` containers:

```shell
admin@i-0c6403c79fb500a19:~$ for i in `seq 1 6`; do curl localhost:5000; done
hello there from nginx_0
hello there from nginx_1
hello there from nginx_0
hello there from nginx_1
hello there from nginx_0
hello there from nginx_1
```