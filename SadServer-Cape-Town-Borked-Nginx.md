## Cape Town: Borked Nginx

There's an Nginx web server installed and managed by systemd. Running `curl -I 127.0.0.1:80` returns `curl: (7) Failed to connect to localhost port 80: Connection refused` , fix it so when you curl you get the default Nginx page.

Looking at the status of nginx with `sudo systemctl status nginx`, we see that there's an unexpected `;` at the start of the config file that's preventing it from starting up:

```shell
admin@i-01cf192773f802812:/$ sudo systemctl  status nginx | less -FRX
● nginx.service - The NGINX HTTP and reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Tue 2024-08-06 06:47:57 UTC; 25s ago
    Process: 570 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
        CPU: 31ms

Aug 06 06:47:56 i-01cf192773f802812 systemd[1]: Starting The NGINX HTTP and reverse proxy server...
Aug 06 06:47:57 i-01cf192773f802812 nginx[570]: nginx: [emerg] unexpected ";" in /etc/nginx/sites-enabled/default:1
Aug 06 06:47:57 i-01cf192773f802812 nginx[570]: nginx: configuration file /etc/nginx/nginx.conf test failed
Aug 06 06:47:57 i-01cf192773f802812 systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE
Aug 06 06:47:57 i-01cf192773f802812 systemd[1]: nginx.service: Failed with result 'exit-code'.
Aug 06 06:47:57 i-01cf192773f802812 systemd[1]: Failed to start The NGINX HTTP and reverse proxy server.
```

Reviewing the error log entries in `/var/log/nginx/error.log`, the last line shows the same issue, along with alerts about `too many open files`. The old timestamps suggest this might be how the box was configured:

```h
REDACTED....
2022/09/11 16:32:21 [emerg] 5841#5841: eventfd() failed (24: Too many open files)
2022/09/11 16:32:21 [alert] 5841#5841: socketpair() failed (24: Too many open files)
2022/09/11 16:35:57 [crit] 5841#5841: *6 open() "/var/www/html/index.nginx-debian.html" failed (24: Too many open files), client: 127.0.0.1, server: _, request: "HEAD / HTTP/1.1", host: "localhost"
2022/09/11 17:07:03 [emerg] 6146#6146: unexpected ";" in /etc/nginx/sites-enabled/default:1
2024/08/06 06:47:56 [emerg] 570#570: unexpected ";" in /etc/nginx/sites-enabled/default:1
```

### Nginx config file

Remove the `;` in `/etc/nginx/sites-enabled/default`, run a configtest and restart the service:

```shell
admin@i-01cf192773f802812:/$ sudo vim /etc/nginx/sites-enabled/default

admin@i-01cf192773f802812:/$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

admin@i-01cf192773f802812:/$ sudo systemctl restart nginx

admin@i-01cf192773f802812:/$ sudo systemctl  status nginx | less -FRX
● nginx.service - The NGINX HTTP and reverse proxy server
     Loaded: loaded (/etc/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-08-06 06:49:38 UTC; 11s ago
    Process: 829 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 830 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 831 (nginx)
      Tasks: 2 (limit: 524)
     Memory: 2.4M
        CPU: 32ms
     CGroup: /system.slice/nginx.service
             ├─831 nginx: master process /usr/sbin/nginx
             └─832 nginx: worker process

Aug 06 06:49:38 i-01cf192773f802812 systemd[1]: Starting The NGINX HTTP and reverse proxy server...
Aug 06 06:49:38 i-01cf192773f802812 nginx[829]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Aug 06 06:49:38 i-01cf192773f802812 nginx[829]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Aug 06 06:49:38 i-01cf192773f802812 systemd[1]: Started The NGINX HTTP and reverse proxy server.
```

Nginx started fine, but it's throwing a `500 Internal Server Error`:

```shell
admin@i-01cf192773f802812:/$ curl -I localhost
HTTP/1.1 500 Internal Server Error
Server: nginx/1.18.0
Date: Tue, 06 Aug 2024 06:50:14 GMT
Content-Type: text/html
Content-Length: 177
Connection: close
```

### Open files limit

After checking the error logs again, I noticed `too many open files` alerts with timestamps from the day I ran this box:

```h
2024/08/06 06:49:38 [emerg] 832#832: eventfd() failed (24: Too many open files)
2024/08/06 06:49:38 [alert] 832#832: socketpair() failed (24: Too many open files)
2024/08/06 06:50:14 [crit] 832#832: *1 open() "/var/www/html/index.nginx-debian.html" failed (24: Too many open files), client: 127.0.0.1, server: _, request: "HEAD / HTTP/1.1", host: "localhost"
```

As seen in the `/var/log/nginx/access.log`, it seems the server is being flooded with requests, likely from an automated script:

```h
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
127.0.0.1 - - [11/Sep/2022:16:04:21 +0000] "GET / HTTP/1.1" 200 396 "-" "hey/0.0.1"
```

Looking at the resource limits of `nginx` will shed some light on this. First let's find the `pid` of nginx:

```shell
admin@i-01cf192773f802812:/$ ps -ef --forest | grep nginx | grep -v grep
root         831       1  0 06:49 ?        00:00:00 nginx: master process /usr/sbin/nginx
www-data     832     831  0 06:49 ?        00:00:00  \_ nginx: worker process
```

The maximum files that can be opened is set to `10` for the master process:

```shell
admin@i-01cf192773f802812:/$ cat /proc/831/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             1748                 1748                 processes 
Max open files            10                   10                   files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       1748                 1748                 signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us        
```

Same for worker process(inherit from parent - fork()):

```shell
admin@i-01cf192773f802812:/$ cat /proc/832/limits | grep open
Max open files            10                   10                   files     
```

> [!NOTE]
> According to RHEL: When in doubt, `/proc/PID/limits` is the authoritative source of what ulimits or resource limits have been set to.


The resource limits of the `www-data` user is not taken into account because `nginx`  is managed by `systemd`:

```shell
admin@i-01cf192773f802812:/$ sudo runuser -u www-data -- bash
www-data@i-01cf192773f802812:/$ ulimit -a 
real-time non-blocking time  (microseconds, -R) unlimited
core file size              (blocks, -c) 0
data seg size               (kbytes, -d) unlimited
scheduling priority                 (-e) 0
file size                   (blocks, -f) unlimited
pending signals                     (-i) 1748
max locked memory           (kbytes, -l) 65536
max memory size             (kbytes, -m) unlimited
open files                          (-n) 1024
pipe size                (512 bytes, -p) 8
POSIX message queues         (bytes, -q) 819200
real-time priority                  (-r) 0
stack size                  (kbytes, -s) 8192
cpu time                   (seconds, -t) unlimited
max user processes                  (-u) 1748
virtual memory              (kbytes, -v) unlimited
file locks                          (-x) unlimited

www-data@i-01cf192773f802812:/$ ulimit -Hn
1048576
www-data@i-01cf192773f802812:/$ ulimit -Sn
1024
www-data@i-01cf192773f802812:/$ 
```

> [!IMPORTANT]
> If a service is managed by `systemd`, system limits are ignored, and all resources are managed by `systemd`.

### Nginx systemd unit override

As expected the `nginx.service` unit file has `LimitNOFILE` set to `10`:

```shell
admin@i-01cf192773f802812:/$ cat etc/systemd/system/nginx.service
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
LimitNOFILE=10

[Install]
WantedBy=multi-user.target
```

Another way to look at it:

```shell
admin@i-0af5dcc58cec252e7:/$ systemctl show nginx | grep -i limitnofile
LimitNOFILE=10
LimitNOFILESoft=10
```

Here's the list of files that are currently [opened](https://man7.org/linux/man-pages/man5/proc_pid_fd.5.html):

```shell
admin@i-01cf192773f802812:/$ sudo ls -lah /proc/831/fd
total 0
dr-x------ 2 root root  0 Aug  6 06:54 .
dr-xr-xr-x 9 root root  0 Aug  6 06:49 ..
lrwx------ 1 root root 64 Aug  6 06:59 0 -> /dev/null
lrwx------ 1 root root 64 Aug  6 06:59 1 -> /dev/null
l-wx------ 1 root root 64 Aug  6 06:59 2 -> /var/log/nginx/error.log
lrwx------ 1 root root 64 Aug  6 06:59 3 -> 'socket:[12212]'
l-wx------ 1 root root 64 Aug  6 06:59 4 -> /var/log/nginx/access.log
l-wx------ 1 root root 64 Aug  6 06:59 5 -> /var/log/nginx/error.log
lrwx------ 1 root root 64 Aug  6 06:59 6 -> 'socket:[12332]'
lrwx------ 1 root root 64 Aug  6 06:59 7 -> 'socket:[12333]'
lrwx------ 1 root root 64 Aug  6 06:59 8 -> 'socket:[12213]'
```

With `lsof`:

```shell
admin@i-01cf192773f802812:/$ sudo lsof -p 831
--REDACTED
nginx   831 root    0u   CHR                1,3       0t0      4 /dev/null
nginx   831 root    1u   CHR                1,3       0t0      4 /dev/null
nginx   831 root    2w   REG              259,1    208576 273038 /var/log/nginx/error.log
nginx   831 root    3u  unix 0x000000007ccd9f9f       0t0  12212 type=STREAM
nginx   831 root    4w   REG              259,1 239657277 273033 /var/log/nginx/access.log
nginx   831 root    5w   REG              259,1    208576 273038 /var/log/nginx/error.log
nginx   831 root    6u  IPv4              12332       0t0    TCP *:http (LISTEN)
nginx   831 root    7u  IPv6              12333       0t0    TCP *:http (LISTEN)
nginx   831 root    8u  unix 0x000000004c03ecff       0t0  12213 type=STREAM
```

Instead of editing the unit file directly, we should create a new directory called `/etc/systemd/system/nginx.service.d`, and add our modified values inside a file called `override.conf`:

```shell
[Service]
LimitNOFILE=infinity
```

Reload the configuration with `systemctl daemon-reload` for these changes to take effect, and restart the service:

```shell
> sudo systemctl daemon-reload && sudo systemctl restart nginx
``` 

You can view everything with `sudo systemctl cat nginx`:

```shell
admin@i-01cf192773f802812:/etc/systemd/system/nginx.service.d$ sudo systemctl cat nginx
# /etc/systemd/system/nginx.service
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=syslog.target network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t
ExecStart=/usr/sbin/nginx
ExecReload=/usr/sbin/nginx -s reload
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
LimitNOFILE=10

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=infinity
```

> [!NOTE]
> On RHEL, the systemd unit configuration is in `/usr/lib/systemd/system/nginx.service`. The override directory: `/etc/systemd/system/nginx.service.d` is available by default.


Let's view the limits again:

```shell
root@i-08e73843d9bef5371:~# pidof nginx
1031 1030 1028
root@i-08e73843d9bef5371:~# cat /proc/1030/limits 
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            8388608              unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             1748                 1748                 processes 
Max open files            1024                 524288               files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       1748                 1748                 signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us   
```

Looks good. The default limits are set by systemd:

```shell
admin@i-0d8c81fbd09a91972:/$ sudo systemctl show | grep -i limitnofile
DefaultLimitNOFILE=524288
DefaultLimitNOFILESoft=1024
admin@i-0d8c81fbd09a91972:/$ 
```

Now you should be able to curl the webserver:

```shell
admin@i-01cf192773f802812:/$ curl -I localhost
HTTP/1.1 200 OK
Server: nginx/1.18.0
Date: Tue, 06 Aug 2024 07:00:56 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Sun, 11 Sep 2022 15:58:42 GMT
Connection: keep-alive
ETag: "631e05b2-264"
Accept-Ranges: bytes
```


### Useful

Refs:
- https://www.baeldung.com/linux/stale-file-handles#file-handle
- https://blog.nginx.org/blog/inside-nginx-how-we-designed-for-performance-scale
- https://serverfault.com/questions/628610/increasing-nproc-for-processes-launched-by-systemd-on-centos-7/678861#678861
- https://access.redhat.com/solutions/4482841
- https://access.redhat.com/solutions/6973808