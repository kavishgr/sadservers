# Tokyo: can't serve web file

There's a web server serving a file `/var/www/html/index.html` with content "hello sadserver" but when we try to check it locally with an HTTP client like `curl 127.0.0.1:80`, nothing is returned, and it hangs.

A `curl localhost` is throwing a `403 Forbidden`:

```html
➜ curl localhost
<html><head>
<title>403 Forbidden</title>
</head><body>
<h1>Forbidden</h1>
<p>You don't have permission to access this resource.</p>
<hr>
<address>Apache/2.4.52 (Ubuntu) Server at localhost Port 80</address>
</body></html>
```

### Permission issues

The error logs shows that it's a file permission issue on `/var/www/html/index.html`:

```shell
➜ cat /var/log/apache2/error.log
[Mon Jul 29 05:34:46.543698 2024] [mpm_event:notice] [pid 641:tid 140068313917312] AH00489: Apache/2.4.52 (Ubuntu) configured -- resuming normal operations
[Mon Jul 29 05:34:46.543729 2024] [core:notice] [pid 641:tid 140068313917312] AH00094: Command line: '/usr/sbin/apache2'
[Mon Jul 29 05:35:27.717301 2024] [core:error] [pid 778:tid 140068085761600] (13)Permission denied: [client ::1:52428] AH00132: file permissions deny server access: /var/www/html/index.html
[Mon Jul 29 05:37:32.312918 2024] [core:error] [pid 779:tid 140068303267392] (13)Permission denied: [client ::1:52430] AH00132: file permissions deny server access: /var/www/html/index.html
[Mon Jul 29 05:37:58.877792 2024] [mpm_event:notice] [pid 641:tid 140068313917312] AH00493: SIGUSR1 received.  Doing graceful restart
[Mon Jul 29 05:37:58.885820 2024] [mpm_event:notice] [pid 641:tid 140068313917312] AH00489: Apache/2.4.52 (Ubuntu) configured -- resuming normal operations
```

> [!NOTE]
> Don't mind the graceful restart, I ran this box multiple times for silly reasons.


Notice the request came from the loopback IPV6 address(`::1`) instead of IPV4(`127.0.0.1`):

```
[client ::1:52428] AH00132: file permissions deny server access: /var/www/html/index.html
```

Let's fix the file permission error first. Fixing it with `chmod 664` because `apache` needs to be able to read the file:

```shell
➜ ls -l /var/www/html/index.html 
-rw------- 1 root root 16 Aug  1  2022 /var/www/html/index.html
root@ip-172-31-21-14:/# chmod 664 !$ 
chmod 664 /var/www/html/index.html 
```

Now a `curl localhost` is returning the `index.html`:

```shell
➜ curl localhost
hello sadserver
```

But 127.0.0.1 still hangs.

Taking a look at the `VirtualHost` config shows that it is listening on all IPs:

```shell
➜ apache2ctl -S
VirtualHost configuration: *:80 ip-172-31-21-14.us-east-2.compute.internal 
(/etc/apache2/sites-enabled/000-default.conf:1)
ServerRoot: "/etc/apache2"
Main DocumentRoot: "/var/www/html"
Main ErrorLog: "/var/log/apache2/error.log"
Mutex default: dir="/var/run/apache2/" mechanism=default 
Mutex watchdog-callback: using_defaults
PidFile: "/var/run/apache2/apache2.pid"
Define: DUMP_VHOSTS
Define: DUMP_RUN_CFG
User: name="www-data" id=33
Group: name="www-data" id=33


➜ cat /etc/apache2/sites-enabled/000-default.conf 
<VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

### Listening address and ports

So what the heck is happening here? 

`ss -ntpl4` doesn't show port 80: 

```shell
➜ ss -ntpl4
State    Recv-Q   Send-Q     Local Address:Port     Peer Address:Port   Process                          
LISTEN   0        128              0.0.0.0:22            0.0.0.0:*       users:(("sshd",pid=638,fd=3))   
LISTEN   0        4096       127.0.0.53%lo:53            0.0.0.0:*       users:(("systemd-resolve",pid=437,fd=14))
```

`lsof -i:80` says that the apache socket is mapped in an IPv6 address:

```shell
➜ lsof -i:80
COMMAND PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
apache2 640     root    4u  IPv6  17684      0t0  TCP *:http (LISTEN)
apache2 772 www-data    4u  IPv6  17684      0t0  TCP *:http (LISTEN)
apache2 773 www-data    4u  IPv6  17684      0t0  TCP *:http (LISTEN)
```

> [!TIP]
> According to `man lsof`: When an open IPv4 network file's address is mapped in an IPv6 address, the open file's type will be IPv6, not IPv4, and its display will be selected by '6', not '4'.


The virtualhost config did confirms that it's mapped to both IP family, and `ss -ntpl6` confirms it just like `lsof`:

```shell
➜ ss -ntpl6
State        Recv-Q        Send-Q               Local Address:Port               Peer Address:Port       Process                                                                                                  
LISTEN       0             4096                             *:6767                          *:*           users:(("sadagent",pid=544,fd=7))                                                                       
LISTEN       0             511                              *:80                            *:*           users:(("apache2",pid=780,fd=4),("apache2",pid=779,fd=4),("apache2",pid=648,fd=4))                      
LISTEN       0             4096                             *:8080                          *:*           users:(("gotty",pid=558,fd=6))                                                                          
LISTEN       0             128                           [::]:22                         [::]:*           users:(("sshd",pid=638,fd=4))                   
```

I'm pretty sure that localhost is resolving to the IPV6 loopback address, `::1`. 
The error logs did show `::1`(IPV6 loopback address), and the access logs also reveals the same thing:

```shell
➜ cat /var/log/apache2/access.log
::1 - - [29/Jul/2024:05:37:32 +0000] "GET / HTTP/1.1" 403 435 "-" "curl/7.81.0"
::1 - - [29/Jul/2024:05:38:00 +0000] "GET / HTTP/1.1" 403 435 "-" "curl/7.81.0"
::1 - - [29/Jul/2024:05:38:21 +0000] "GET / HTTP/1.1" 200 243 "-" "curl/7.81.0"
```

A `curl -v` tells us that `127.0.0.1:80` was tried first and then fall back to `::1:80`:

```shell
➜ curl -v localhost
*   Trying 127.0.0.1:80...
*   Trying ::1:80...
* Connected to localhost (::1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.81.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Fri, 26 Jul 2024 08:32:09 GMT
< Server: Apache/2.4.52 (Ubuntu)
< Last-Modified: Mon, 01 Aug 2022 00:40:24 GMT
< ETag: "10-5e5233ed9edbf"
< Accept-Ranges: bytes
< Content-Length: 16
< Content-Type: text/html
< 
hello sadserver
* Connection #0 to host localhost left intact
```

### IPtables

The next place to look at now, is the firewall. As we can see, `iptables` does have a rule to drop incoming connection for `http` or `port 80` from anywhere(including IPV4 loopback address):

```shell
➜ iptables -L --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination         
1    DROP       tcp  --  anywhere             anywhere             tcp dpt:http

Chain FORWARD (policy ACCEPT)
num  target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
num  target     prot opt source               destination       
```

And `ip6tables` has none:

```shell
➜ ip6tables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
```

Removing the rule:

```shell
➜ iptables -D INPUT 1
➜ iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination 
```

Now `127.0.0.1` is returning: 

```shell
➜ curl 127.0.0.1:80
hello sadserver
```

### Socket vs. Network Binding: Apache and SSH

Changing `Listen` to `0.0.0.0:80`(to any port) in `ports.conf`(`httpd.conf` on RHEL) will only show IPV4:

```shell
➜ lsof -i:80
COMMAND  PID     USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
apache2 1626     root    3u  IPv4  19360      0t0  TCP *:http (LISTEN)
apache2 1627 www-data    3u  IPv4  19360      0t0  TCP *:http (LISTEN)
apache2 1628 www-data    3u  IPv4  19360      0t0  TCP *:http (LISTEN)
```

With `ss -ntpl4`:

```shell
➜ ss -ntpl4
State        Recv-Q        Send-Q               Local Address:Port               Peer Address:Port       Process                                                                                                  
LISTEN       0             511                        0.0.0.0:80                      0.0.0.0:*           users:(("apache2",pid=1519,fd=3),("apache2",pid=1518,fd=3),("apache2",pid=1517,fd=3))                   
LISTEN       0             4096                 127.0.0.53%lo:53                      0.0.0.0:*           users:(("systemd-resolve",pid=441,fd=14))                                                               
LISTEN       0             128                        0.0.0.0:22                      0.0.0.0:*           users:(("sshd",pid=636,fd=3))                                             
```

`Apache/httpd` relies on the Linux kernel networking stack to bind to IP addresses and Ports. It uses directives like `Listen` to specify the IP addresses and ports it should listen on.

`sshd` on the other hand creates a socket for both `IPV4` and `IPV6` connections:

```shell
➜ lsof -i:22
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    633 root    3u  IPv4  17620      0t0  TCP *:ssh (LISTEN)
sshd    633 root    4u  IPv6  17631      0t0  TCP *:ssh (LISTEN)
```

Here's the `ssh.socket` file on **Fedora CoreOS**:

```shell
root@coreos:/usr/lib/systemd/system# cat sshd.socket
[Unit]
Description=OpenSSH Server Socket
Documentation=man:sshd(8) man:sshd_config(5)
Conflicts=sshd.service

[Socket]
ListenStream=22
Accept=yes

[Install]
WantedBy=sockets.target
```

> [!NOTE]
> From `systemd` docs: If the address string is a single number, it is read as port number to listen on via IPv6. Address specified as IPv6, might still make the service available via IPv4 too.
