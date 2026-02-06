# 3) Install Nginx

## Previous steps

After installing PHP 8.5 via PPA, we have the CLI SAPI (/usr/bin/php8.5) for terminal scripts. Installing php8.5-fpm added the FPM SAPI as a persistent service with 3 processes (master + 2 workers), created a Unix socket at `/run/php/php8.5-fpm.sock` listening for FastCGI connections, and established separate web configuration at /etc/php/8.5/fpm/. CLI runs ephemerally as our user, while FPM runs continuously with workers as www-data. 

## Nginx's Role

Nginx acts as the web server and FastCGI client. For PHP requests, it:
1.  Handles the initial HTTP connection and static file serving.
2.  Translates the HTTP request into FastCGI parameters (environment variables).
3.  Connects to the PHP-FPM socket.
4.  Transmits the request using the FastCGI binary protocol.
5.  Receives the response and delivers it to the client.

HTTP Client -> Nginx (HTTP Parsing) -> FastCGI Protocol (Binary Socket) -> PHP-FPM Worker (Persistent Process) -> PHP Execution -> FastCGI Response -> Nginx -> HTTP Client

Install and configure Nginx to connect to the PHP-FPM socket via `fastcgi_pass unix:/run/php/php8.5-fpm.sock;`.

## NGINX official Installation Guide
https://nginx.org/en/linux_packages.html#Ubuntu

## System Changes After Installing Nginx & Verification Commands

**1. New Binary**: `/usr/sbin/nginx` installed (version 1.24.0 from nginx.org).  
`which nginx && nginx -v`

```bash
lab-admin@lab-server:~$ which nginx && nginx -v
/usr/sbin/nginx
nginx version: nginx/1.28.1
```

**2. Configuration**: Complete `/etc/nginx/` directory created.  
`ls -la /etc/nginx/`

```bash
lab-admin@lab-server:~$ ls -la /etc/nginx
total 36
drwxr-xr-x   3 root root 4096 Feb  2 19:58 .
drwxr-xr-x 104 root root 4096 Feb  2 19:58 ..
drwxr-xr-x   2 root root 4096 Feb  2 19:58 conf.d
-rw-r--r--   1 root root 1007 Dec 23 18:40 fastcgi_params
-rw-r--r--   1 root root 5349 Dec 23 18:40 mime.types
lrwxrwxrwx   1 root root   22 Dec 23 18:52 modules -> /usr/lib/nginx/modules
-rw-r--r--   1 root root  644 Dec 23 18:52 nginx.conf
-rw-r--r--   1 root root  636 Dec 23 18:40 scgi_params
-rw-r--r--   1 root root  664 Dec 23 18:40 uwsgi_params
```

**3. Systemd Service**: `nginx.service` installed, running, and enabled.  
`sudo systemctl status nginx && sudo systemctl is-enabled nginx`

```bash
lab-admin@lab-server:~$ sudo systemctl status nginx && sudo systemctl is-enabled nginx
● nginx.service - nginx - high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-02-02 20:07:41 UTC; 16min ago
       Docs: https://nginx.org/en/docs/
    Process: 722 ExecStart=/usr/sbin/nginx -c ${CONFFILE} (code=exited, status=0/SUCCESS)
   Main PID: 773 (nginx)
      Tasks: 3 (limit: 4372)
     Memory: 4.9M
        CPU: 20ms
     CGroup: /system.slice/nginx.service
             ├─773 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf"
             ├─774 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""
             └─775 "nginx: worker process" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ""

Feb 02 20:07:41 lab-server systemd[1]: Starting nginx - high performance web server...
Feb 02 20:07:41 lab-server systemd[1]: Started nginx - high performance web server.
enabled
l
```

**4. Network**: Listening on HTTP port 80 (`0.0.0.0:80`).  
`sudo ss -tlnp | grep nginx`

```bash
lab-admin@lab-server:~$ sudo ss -tlnp | grep nginx
LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=775,fd=6),("nginx",pid=774,fd=6),("nginx",pid=773,fd=6))
```

**5. User**: Dedicated `nginx` system user created.  
`grep nginx /etc/passwd && ps aux | grep nginx | head -3`

```bash
lab-admin@lab-server:~$ grep nginx /etc/passwd && ps aux | grep nginx | head -3
nginx:x:998:999:nginx user:/nonexistent:/usr/sbin/nologin
root         773  0.0  0.0  12004  1636 ?        Ss   20:07   0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
nginx        774  0.0  0.1  12744  4492 ?        S    20:07   0:00 nginx: worker process
nginx        775  0.0  0.1  12744  4492 ?        S    20:07   0:00 nginx: worker process
```

**6. Logs**: `/var/log/nginx/` directory with log files.  
`ls -la /var/log/nginx/`

```bash
lab-admin@lab-server:~$ ls -la /var/log/nginx/
total 12
drwxr-xr-x  2 root  root   4096 Feb  2 19:58 .
drwxrwxr-x 12 root  syslog 4096 Feb  2 20:07 ..
-rw-r-----  1 nginx adm       0 Feb  2 19:58 access.log
-rw-r-----  1 nginx adm     542 Feb  2 20:07 error.log
```

**7. Web Root**: Default content at `/usr/share/nginx/html/`.  
`ls -la /usr/share/nginx/html/ && curl -s http://localhost | grep -o "<title>.*</title>"`

```bash
lab-admin@lab-server:~$ ls -la /usr/share/nginx/html/ && curl -s http://localhost | grep -o "<title>.*</title>"
total 16
drwxr-xr-x 2 root root 4096 Feb  2 19:58 .
drwxr-xr-x 3 root root 4096 Feb  2 19:58 ..
-rw-r--r-- 1 root root  497 Dec 23 18:40 50x.html
-rw-r--r-- 1 root root  615 Dec 23 18:40 index.html
<title>Welcome to nginx!</title>
```

**8. Repository**: Official nginx.org APT repo added.  
`cat /etc/apt/sources.list.d/nginx.list && ls -la /usr/share/keyrings/nginx-archive-keyring.gpg`

```bash
lab-admin@lab-server:~$ cat /etc/apt/sources.list.d/nginx.list && ls -la /usr/share/keyrings/nginx-archive-keyring.gpg
deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] https://nginx.org/packages/ubuntu jammy nginx
-rw-r--r-- 1 root root 8538 Feb  2 19:56 /usr/share/keyrings/nginx-archive-keyring.gpg
```

**9. Processes**: Master + worker processes running.  
`pstree -p | grep nginx`

```bash
lab-admin@lab-server:~$ pstree -p | grep nginx
           |-nginx(773)-+-nginx(774)
           |            `-nginx(775)
```


## Nginx Configuration Structure & Service

**`nginx.conf`** - Main configuration file. Defines global settings (worker processes, connections, logging) and includes other configs via `include` directives. It's the entry point Nginx reads on startup.

**`conf.d/`** - Directory for additional configuration snippets. Files here ending in `.conf` are automatically loaded. Good for modular settings (e.g., `gzip.conf`, `security.conf`).

**`sites-available/`** - Stores virtual host configurations (one per site). These are **templates** that are not active unless symlinked to `sites-enabled/`.

**`sites-enabled/`** - Contains **symlinks** to configurations in `sites-available/`. Only files symlinked here are active. This allows easy site enabling/disabling.

## Systemd Service: `nginx.service`

**What it launches**: It starts **one master process** as defined by `Type=forking` in the service file.

**How many processes**:
- **1 Master Process** (root): Launched by systemd. Reads config, manages workers.
- **N Worker Processes** (`nginx` user): Forked by the master. Number is set by `worker_processes` directive in `nginx.conf` (default is `auto`, often equals CPU cores).

**Process tree created**:
```text
systemd (PID 1) → nginx.service → nginx master (root) → nginx worker (nginx user)
├→ nginx worker (nginx user)
└→ nginx worker (nginx user)
```

**Based on what**: The master process reads `nginx.conf`, which specifies `worker_processes`. It then forks that many worker processes. Each worker can handle thousands of concurrent connections via an event-driven, non-blocking I/O model.