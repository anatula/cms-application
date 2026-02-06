# 4) PHP-FPM and Nginx Configuration
With the help of this https://www.php.net/manual/en/install.unix.nginx.php
old guide (published in 2016) intended for people who compiled Nginx and PHP from source

1. Check a Critical PHP Setting (Security)

```bash
lab-admin@lab-server:~$ grep "cgi.fix_pathinfo" /etc/php/8.5/fpm/php.ini
; cgi.fix_pathinfo provides *real* PATH_INFO/PATH_TRANSLATED support for CGI.  PHP's
;cgi.fix_pathinfo=1
```

```bash
lab-admin@lab-server:~$ sudo sed -i '797s/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/' /etc/php/8.5/fpm/php.ini
lab-admin@lab-server:~$ grep "cgi.fix_pathinfo" /etc/php/8.5/fpm/php.ini
; cgi.fix_pathinfo provides *real* PATH_INFO/PATH_TRANSLATED support for CGI.  PHP's
cgi.fix_pathinfo=0
lab-admin@lab-server:~$ sudo systemctl restart php8.5-fpm
lab-admin@lab-server:~$ sudo systemctl status php8.5-fpm --no-pager | head -5
● php8.5-fpm.service - The PHP 8.5 FastCGI Process Manager
     Loaded: loaded (/lib/systemd/system/php8.5-fpm.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2026-02-02 20:43:54 UTC; 4s ago
       Docs: man:php-fpm8.5(8)
    Process: 1339 ExecStartPost=/usr/lib/php/php-fpm-socket-helper install /run/php/php-fpm.sock /etc/php/8.5/fpm/pool.d/www.conf 85 (code=exited, status=0/SUCCESS)

```
Edit `/etc/nginx/conf.d/default.conf` like:

```bash
server {
    listen       80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.php index.html index.htm;  # <- Added index.php
    }

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    location ~ \.php$ {
        root           html;
        fastcgi_pass   unix:/run/php/php8.5-fpm.sock;  # <- Changed to socket
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;  # <- Fixed path
        include        fastcgi_params;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

Essentially, this block is a translator. Nginx cannot read PHP code on its own; it only knows how to serve static files like images or HTML. This configuration tells Nginx: "If you see a file ending in .php, don't try to open it yourself—send it to the PHP engine instead." 

Here is the breakdown of what each line is doing:

- `location ~ \.php$`: This identifies the request. The ~ tells Nginx to use a "regular expression" to look for any web address ending specifically in .php.

- `root html;`: Tells Nginx where your website files are stored on the hard drive (e.g., in a folder named html).

- `fastcgi_pass unix:/run/php/php8.5-fpm.sock;`: This is the "hand-off." It tells Nginx to send the request to the PHP-FPM service using a Unix Socket. Sockets are faster than network ports because they communicate directly through the local file system without network overhead.

- `fastcgi_index index.php;`: If a user visits a folder (like example.com/blog/) instead of a specific file, Nginx will look for index.php inside that folder to serve as the default.

- `fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;`: This tells the PHP engine exactly which file to run. It combines your website's folder path ($document_root) with the specific file requested ($fastcgi_script_name) so PHP can find it on the disk.

- `include fastcgi_params;`: This imports a standard list of "helper" settings (like the user’s IP address and browser type) so that your PHP scripts can access that information using $_SERVER variables. 

# Test configuration syntax
sudo nginx -t
# Should show: syntax is ok... test is successful

# Reload nginx
sudo systemctl reload nginx

# Create test PHP file
echo '<?php echo "PHP is working via Nginx!"; ?>' | sudo tee /usr/share/nginx/html/test.php

# Test
curl http://localhost/test.php
# Should show: PHP is working via Nginx!

Error:

```text
connect() to unix:/run/php/php8.5-fpm.sock failed (13: Permission denied)
The nginx worker process (running as user nginx) doesn't have permission to connect to the PHP-FPM socket (owned by www-data).
```

The Problem: Socket Permission Mismatch

Check current socket permissions:

bash
ls -la /run/php/php8.5-fpm.sock
# Probably shows: srw-rw---- 1 www-data www-data
The socket is owned by www-data:www-data, but nginx runs as nginx:nginx.

## Solution: Fix Socket Permissions

### Option 1: Make PHP-FPM Socket Accessible to Nginx (Recommended)
Edit PHP-FPM pool config to allow nginx group access:

```bash
sudo nano /etc/php/8.5/fpm/pool.d/www.conf
```
Find these lines and change them:

```
ini
; Unix user/group of processes
user = www-data
group = www-data

; Listen socket permissions
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
```
Change to:

```
ini
; Unix user/group of processes
user = www-data
group = www-data

; Listen socket permissions - ALLOW nginx group
listen.owner = www-data
listen.group = nginx    # ← CHANGE THIS
listen.mode = 0660      # rw-rw---- (www-data & nginx can read/write)
```

Or even simpler (allow everyone on system to connect):

```
ini
listen.mode = 0666      # rw-rw-rw- (everyone can read/write)
```

### Option 2: Change Nginx to Run as www-data
Edit nginx config to match PHP-FPM:

```bash
sudo nano /etc/nginx/nginx.conf
Find the user directive (usually at the top):

nginx
user nginx;
Change to:

nginx
user www-data;
```

### Option 3: Quick Fix (Temporary Test)
```bash
# Temporarily change socket permissions
sudo chmod 0666 /run/php/php8.5-fpm.sock
```
# Test if it works
`curl http://localhost/test.php`

Step-by-Step Fix (Recommend Option 1):
Edit PHP-FPM config:

```bash
sudo nano /etc/php/8.5/fpm/pool.d/www.conf
```

Change: `listen.group = www-data → listen.group = nginx`

Restart PHP-FPM:

```bash
sudo systemctl restart php8.5-fpm
```
Check new socket permissions:

```bash
ls -la /run/php/php8.5-fpm.sock
show: srw-rw---- 1 www-data nginx
```

```bash
curl http://localhost/test.php
```
Verify Nginx Can Connect
```bash
# Test as nginx user
sudo -u nginx ls -la /run/php/php8.5-fpm.sock
# Should show the socket (not "Permission denied")
```
Why This Happened
PHP-FPM default: Creates socket owned by www-data:www-data (mode 0660)
Nginx default: Runs as nginx:nginx user/group
Result: Nginx can't connect to socket (different group, no permissions)

# Nginx + PHP-FPM Connection Fixed: Explanation & Next Steps

## ✅ **Current Status: WORKING (via Temporary Fix)**
The command `sudo chmod 0666 /run/php/php8.5-fpm.sock` resolved the "Permission denied" error. PHP is now executing (`curl http://localhost/test.php` returns "PHP is working via Nginx!").

## 🔍 **Why It Works Now**
**The Problem**: The Unix socket file (`/run/php/php8.5-fpm.sock`) is the communication channel between Nginx and PHP-FPM. By default, it was created with permissions `0660` (`rw-rw----`), owned by `www-data:www-data`. The Nginx worker processes run under the `nginx` user, which is **not** in the `www-data` group, causing a "Permission denied" error when trying to connect.

**The Temporary Fix**: Changing permissions to `0666` (`rw-rw-rw-`) allows **any user** on the system (including `nginx`) to read from and write to the socket.

## ⚠️ **Why This is a Temporary & Risky Fix**
1.  **Security Vulnerability**: A world-writable socket (`0666`) means any process or user on the server can connect to and interact with your PHP application backend.
2.  **Not Persistent**: The socket file is recreated every time the `php8.5-fpm` service restarts, reverting to its default, secure permissions and breaking the connection again.
3.  **Bad Practice**: It violates the security principle of least privilege.

## 🔧 **Recommended Permanent Solutions**

### **Option 1: Fix PHP-FPM Configuration (Recommended)**
Edit the PHP-FPM pool configuration to grant the `nginx` group access to the socket.

```bash
sudo nano /etc/php/8.5/fpm/pool.d/www.conf
Find and modify these lines:

ini
listen.owner = www-data
listen.group = nginx      # Change from 'www-data' to 'nginx'
listen.mode = 0660
Apply: sudo systemctl restart php8.5-fpm
Result: Socket will be rw-rw---- for www-data:nginx.
```

### Option 2: Add Nginx User to www-data Group
Modify system groups so the nginx user belongs to the www-data group.

```bash
sudo usermod -a -G www-data nginx
# Then restart Nginx: sudo systemctl restart nginx
Requires: Ensuring the socket's group remains www-data.
```

### Option 3: Use TCP Port Instead of Unix Socket
Avoid filesystem permissions by using a network port.

In www.conf: Change listen = /run/php/php8.5-fpm.sock to listen = 127.0.0.1:9000.

In Nginx config: Change fastcgi_pass unix:/run/php/php8.5-fpm.sock; to fastcgi_pass 127.0.0.1:9000;.
