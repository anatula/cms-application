# 4) PHP-FPM and Nginx Configuration

With the help of this https://www.php.net/manual/en/install.unix.nginx.php
old guide (published in 2016) intended for people who compiled Nginx and PHP from source

## Check a Critical PHP Setting (Security)

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
## NGINX Configuation File
Edit `/etc/nginx/conf.d/default.conf`:

```bash
sserver {
    listen       80;
    server_name  localhost;
    
    # Document root - ALL files served from here
    root   /usr/share/nginx/html;
    
    # Default location for all requests
    location / {
        # Files to try when directory is requested
        index  index.php index.html index.htm;
    }
    
    # Custom error page for 5xx errors
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        # Uses server-level root (/usr/share/nginx/html/50x.html)
    }
    
    # PHP files handler
    location ~ \.php$ {
        # Send PHP requests to PHP-FPM via Unix socket
        fastcgi_pass   unix:/run/php/php8.5-fpm.sock;
        
        # Default file if directory is requested
        fastcgi_index  index.php;
        
        # CRITICAL: Tells PHP-FPM which file to execute
        # Combines: document root + requested script path
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        
        # Include standard FastCGI parameters
        include        fastcgi_params;
    }
}
```

To test configuration syntax `sudo nginx -t`
To reload nginx `sudo systemctl reload nginx`

`$document_root`
- Source: The root directive at server level (root /usr/share/nginx/html;)
-Value: /usr/share/nginx/html
- Purpose: Base directory where website files are stored on disk
- Set by: You, in nginx configuration

`$fastcgi_script_name`
- Source: Automatically extracted from the URL request
- Example: User visits http://example.com/blog/post.php → $fastcgi_script_name = /blog/post.php
- Purpose: The specific PHP file requested (relative to document root)
- Set by: NGINX from user's browser request

`SCRIPT_FILENAME Parameter`
```nginx
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
```
- What it does: Combines document root with script path
- Example: `/usr/share/nginx/html + /blog/post.php = /usr/share/nginx/html/blog/post.php`
- Delivered to: PHP-FPM as a FastCGI parameter
- Result in PHP: Available as $_SERVER['SCRIPT_FILENAME']

### Unix Socket vs TCP
- Unix Socket: unix:/run/php/php8.5-fpm.sock (faster, local filesystem communication)
- TCP Socket: 127.0.0.1:9000 (network communication, slower but simpler permissions)

### Why you need sudo to edit /etc/nginx/conf.d/default.conf ?
```bash
ls -la /etc/nginx/conf.d/default.conf
# Output: -rw-r--r-- 1 root root 1234 Feb  3 10:00 /etc/nginx/conf.d/default.conf
```
Reason:
- Owned by root: System configuration files are protected
- Security: Prevents unauthorized users from modifying web server config
- Consistency: System services require consistent, validated configurations

## Test Now PHP is Configured but Permissions Wrong
Create a file with `<?php echo "PHP TEST - "; echo date("H:i:s"); ?>` in /usr/share/nginx/html/phpinfo.php
```bash
# Create a simple test file
lab-admin@lab-server:~$ echo '<?php echo "PHP TEST - "; echo date("H:i:s"); ?>' | sudo tee /usr/share/nginx/html/phpinfo.php

# Test it
lab-admin@lab-server:~$ curl -v http://localhost/phpinfo.php

## The Socket Permission Problem
```

Default setup:

- PHP-FPM socket: Owned by www-data:www-data, permissions 0660
- NGINX worker: Runs as nginx:nginx user
- Problem: Different user/group → Permission denied

### Solution 1: Fix PHP-FPM config (Recommended)

```bash
# Edit PHP-FPM configuration
sudo nano /etc/php/8.5/fpm/pool.d/www.conf

# Change these lines:
listen.owner = www-data
listen.group = nginx    # ← CHANGE from www-data to nginx
listen.mode = 0660

# Restart PHP-FPM
sudo systemctl restart php8.5-fpm

# Verify
ls -la /run/php/php8.5-fpm.sock
# Should show: srw-rw---- 1 www-data nginx
```

### Solution 2: Alternative approaches
```bash
# Option A: Add nginx to www-data group
sudo usermod -a -G www-data nginx
sudo systemctl restart nginx

# Option B: Use TCP port instead of socket
# In PHP-FPM: listen = 127.0.0.1:9000
# In NGINX: fastcgi_pass 127.0.0.1:9000;
```

### Solution 3
```bash
# Temporarily change socket permissions
sudo chmod 0666 /run/php/php8.5-fpm.sock
# Test if it works
`curl http://localhost/test.php`
```

Output: 

```bash
lab-admin@lab-server:~$ cat /usr/share/nginx/html/phpinfo.php
<?php echo "PHP TEST - "; echo date("H:i:s"); ?>
lab-admin@lab-server:~$ curl -v http://localhost/phpinfo.php
*   Trying 127.0.0.1:80...
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET /phpinfo.php HTTP/1.1
> Host: localhost
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 502 Bad Gateway
< Server: nginx/1.28.1
< Date: Sun, 08 Feb 2026 15:01:45 GMT
< Content-Type: text/html
< Content-Length: 497
< Connection: keep-alive
< ETag: "694ae221-1f1"
<
<!DOCTYPE html>
<html>
<head>
<title>Error</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>An error occurred.</h1>
<p>Sorry, the page you are looking for is currently unavailable.<br/>
Please try again later.</p>
<p>If you are the system administrator of this resource then you should check
the error log for details.</p>
<p><em>Faithfully yours, nginx.</em></p>
</body>
</html>
* Connection #0 to host localhost left intact
```

### Troubleshoot
Check the NGINX Error Log
```bash
# View recent nginx errors
sudo tail -20 /var/log/nginx/error.log

# Or follow the log in real-time
sudo tail -f /var/log/nginx/error.log
```

Output:

```bash
2026/02/08 15:01:45 [crit] 19895#19895: *3 connect() to unix:/run/php/php8.5-fpm.sock failed (13: Permission denied) while connecting to upstream, client: 127.0.0.1, server: localhost, request: "GET /phpinfo.php HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.5-fpm.sock:", host: "localhost"
2026/02/08 15:02:55 [crit] 19896#19896: *5 connect() to unix:/run/php/php8.5-fpm.sock failed (13: Permission denied) while connecting to upstream, client: 127.0.0.1, server: localhost, request: "GET /phpinfo.php HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.5-fpm.sock:", host: "localhost"
```

### PERMISSION DENIED ERROR
The error log confirms exactly what we predicted: Permission denied on the socket. `connect() to unix:/run/php/php8.5-fpm.sock failed (13: Permission denied)`

Check Current Permissions `lab-admin@lab-server:~$ ls -la /run/php/php8.5-fpm.sock`
```bash
lab-admin@lab-server:~$ ls -la /run/php/php8.5-fpm.sock
srw-rw---- 1 www-data www-data 0 Feb  7 17:40 /run/php/php8.5-fpm.sock
```

- NGINX runs as: nginx user (NOT www-data)
- Socket allows: Only www-data user and www-data group members
- Result: nginx user is blocked (falls under "others" with --- no access)

#### Quick Temporary Fix (Test First)

- Changes permissions from 0660 (rw-rw----) to 0666 (rw-rw-rw-)
- Allows ANY user (including nginx) to access the socket
- Warning: Not secure, will reset on PHP-FPM restart (do `sudo systemctl restart php8.5-fpm` and try curl `curl http://localhost/phpinfo.php` it does not work again)

```bash
lab-admin@lab-server:~$ ls -la /run/php/php8.5-fpm.sock
srw-rw---- 1 www-data www-data 0 Feb  7 17:40 /run/php/php8.5-fpm.sock
lab-admin@lab-server:~$ sudo chmod 0666 /run/php/php8.5-fpm.sock
lab-admin@lab-server:~$ ls -la /run/php/php8.5-fpm.sock
srw-rw-rw- 1 www-data www-data 0 Feb  7 17:40 /run/php/php8.5-fpm.sock
lab-admin@lab-server:~$ curl http://localhost/phpinfo.php
PHP TEST - 15:08:07
```

### Apply Permanent Fix

Socket permissions are controlled by PHP-FPM configuration, not by manual chmod commands

```bash
# Edit PHP-FPM config to make the change permanent
lab-admin@lab-server:~$ sudo vim /etc/php/8.5/fpm/pool.d/www.conf

# Find and change:
# listen.group = www-data  →  listen.group = nginx

# Restart PHP-FPM again
lab-admin@lab-server:~$ sudo systemctl restart php8.5-fpm

# Check socket permissions - NOW PERMANENTLY FIXED
lab-admin@lab-server:~$ ls -la /run/php/php8.5-fpm.sock
# nginx GROUP!
srw-rw---- 1 www-data nginx 0 Feb  8 15:13 /run/php/php8.5-fpm.sock

# Test PHP - SHOULD WORK PERMANENTLY
lab-admin@lab-server:~$ curl http://localhost/phpinfo.php
# Now it works!
PHP TEST - 15:14:51
```

## Lifecycle Visualization:
```text
USER's BROWSER
     ↓
     http://example.com/test.php?id=123
     ↓
NGINX (variables created internally)
     ├── $uri = "/test.php"
     ├── $args = "id=123"
     ├── $document_root = "/usr/share/nginx/html" (from your config)
     └── $fastcgi_script_name = "/test.php"
     ↓
NGINX combines: $document_root + $fastcgi_script_name
     ↓
FASTCGI message to PHP-FPM:
     SCRIPT_FILENAME=/usr/share/nginx/html/test.php
     QUERY_STRING=id=123
     ↓
PHP-FPM executes /usr/share/nginx/html/test.php
     ↓
PHP sets: $_SERVER['SCRIPT_FILENAME'] = '/usr/share/nginx/html/test.php'
          $_SERVER['QUERY_STRING'] = 'id=123'
```

## PHP-FPM + NGINX Configuration Summary
✅ CHANGED:
- `/etc/php/8.5/fpm/pool.d/www.conf` - Changed l`isten.group` (Socket now shared with nginx group)
- `/etc/php/8.5/fpm/php.ini` - Enabled `cgi.fix_pathinfo=0` security setting

❌ UNCHANGED:
- No new packages
- No new users or group changes
- NGINX config - Core settings unchanged `/etc/nginx/nginx.conf unchanged`
- No new services - systemd service files unchanged
- No network changes, still only HTTP port 80, no SSL/HTTPS
- System architecture - Same process models

*Result: NGINX can now connect to PHP-FPM socket. PHP works.*
- ✅ Dynamic content support - Added PHP processing capability to web server
- ✅ PHP-FPM integration - NGINX now properly proxies PHP requests to PHP-FPM