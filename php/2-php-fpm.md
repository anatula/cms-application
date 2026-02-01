SAPI = Server API
It's a C interface in PHP's source code that defines how PHP receives input and sends output.

PHP Source Code Structure:
In the PHP source code (written in C), there are different main() functions:

1. CLI SAPI (sapi/cli/php_cli.c):
```c
// Simplified actual C code from PHP source
int main(int argc, char *argv[]) {
    // Reads: command line arguments (argc, argv)
    // Processes: One PHP file
    // Outputs: To stdout (terminal)
    // Then: Exits
}
```
Compiled to: /usr/bin/php8.5

2. FPM SAPI (sapi/fpm/fpm/fpm_main.c):
```c
// Simplified actual C code from PHP source  
int main(int argc, char *argv[]) {
    // Reads: FastCGI protocol from socket
    // Processes: Multiple PHP requests over time
    // Outputs: Back through FastCGI to web server
    // Runs: Forever as a daemon
}
```
Compiled to: /usr/sbin/php-fpm8.5

When PHP is compiled, it builds different executables:

```text
php-8.5.0-source/
├── sapi/
│   ├── cli/           → Compiles to: php8.5 (CLI binary)
│   ├── fpm/           → Compiles to: php-fpm8.5 (FPM binary)
└── Zend/              (PHP engine core - used by ALL SAPIs)
```

Input/Output Methods:
CLI SAPI:

- Input: Command line arguments, STDIN
- Output: STDOUT, STDERR
- Environment: Terminal, pipes

FPM SAPI:

- Input: FastCGI protocol over socket
- Output: FastCGI response over same socket
- Environment: Web server requests

When Ondřej builds the packages:

text
For CLI SAPI package (php8.5):
`gcc -o php8.5 sapi/cli/*.c Zend/*.c main/*.c ext/*/*.c -l options`

For FPM SAPI package (php8.5-fpm):
`gcc -o php-fpm8.5 sapi/fpm/*.c Zend/*.c main/*.c ext/*/*.c -l options`
Result: Two different binaries using the same PHP core.

Check the Actual Binaries:
```bash
# After installing php8.5-fpm:
ls -l /usr/bin/php8.5 /usr/sbin/php-fpm8.5

# They're different files, different sizes:
# -rwxr-xr-x 1 root root 10210464 Jan 18 14:10 /usr/bin/php8.5
# -rwxr-xr-x 1 root root 10345678 Jan 18 14:10 /usr/sbin/php-fpm8.5

# Check with file command:
file /usr/bin/php8.5
# ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked...

file /usr/sbin/php-fpm8.5
```

SAPI = Different main() functions in C code that compile to different binaries.
- CLI SAPI = Binary that reads from terminal, outputs to terminal
- FPM SAPI = Binary that reads from web server socket, outputs back to socket

They're independent binaries that can be installed separately. We happened to install CLI first (via the php8.5 meta-package), but we could have installed only FPM for a web server.

https://php.watch/articles/php-8-5-installation-upgrade-guide-debian-ubuntu#fpm

Install PHP-FP = PHP FastCGI Process Manager
PHP-FPM is the recommended way to integrate PHP with web servers such as Apache (with mpm_event), Nginx, Caddy.

The php8.5-fpm package (same ondrej PPA) installs PHP FPM server along with systemd units to automatically start the FPM server when the server starts.

`sudo apt install php8.5-fpm`

To check the installation and that the php-fpm server is running, run the following:

What it installs:
- Binary: /usr/sbin/php-fpm8.5
- Service: php8.5-fpm.service
- Config: /etc/php/8.5/fpm/
- Socket: /run/php/php8.5-fpm.sock

Check the new FPM binary
```bash
ls -la /usr/sbin/php-fpm8.5

-rwxr-xr-x 1 root root 10194456 Jan 18 14:10 /usr/sbin/php-fpm8.5
```

`sudo systemctl status php8.5-fpm`

The FPM service is running:

```
● php8.5-fpm.service - The PHP 8.5 FastCGI Process Manager
     Loaded: loaded (/lib/systemd/system/php8.5-fpm.service; enabled; vendor prese>
     Active: active (running) since Sun 2026-02-01 20:02:20 UTC; 34s ago
       Docs: man:php-fpm8.5(8)
    Process: 13756 ExecStartPost=/usr/lib/php/php-fpm-socket-helper install /run/p>
   Main PID: 13752 (php-fpm8.5)
     Status: "Processes active: 0, idle: 2, Requests: 0, slow: 0, Traffic: 0.00req>
      Tasks: 3 (limit: 4372)
     Memory: 7.9M
        CPU: 62ms
     CGroup: /system.slice/php8.5-fpm.service
             ├─13752 "php-fpm: master process (/etc/php/8.5/fpm/php-fpm.conf)" "" >
             ├─13754 "php-fpm: pool www" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             └─13755 "php-fpm: pool www" "" "" "" "" "" "" "" "" "" "" "" "" "" "">

Feb 01 20:02:20 lab-server systemd[1]: Starting The PHP 8.5 FastCGI Process Manage>
Feb 01 20:02:20 lab-server systemd[1]: Started The PHP 8.5 FastCGI Process Manager.
```
New directory structure (new fpm folder)
```bash
ls -la /etc/php/8.5/
total 24
drwxr-xr-x 6 root root 4096 Feb  1 20:02 .
drwxr-xr-x 3 root root 4096 Feb  1 13:55 ..
drwxr-xr-x 3 root root 4096 Feb  1 13:56 apache2
drwxr-xr-x 3 root root 4096 Feb  1 13:56 cli
drwxr-xr-x 4 root root 4096 Feb  1 20:02 fpm
drwxr-xr-x 2 root root 4096 Feb  1 13:56 mods-available
```

FPM config directory:

```bash
tree /etc/php/8.5/fpm/
/etc/php/8.5/fpm/
├── conf.d
│   ├── 10-pdo.ini -> /etc/php/8.5/mods-available/pdo.ini
│   ├── 20-calendar.ini -> /etc/php/8.5/mods-available/calendar.ini
│   ├── 20-ctype.ini -> /etc/php/8.5/mods-available/ctype.ini
│   ├── 20-exif.ini -> /etc/php/8.5/mods-available/exif.ini
│   ├── 20-ffi.ini -> /etc/php/8.5/mods-available/ffi.ini
│   ├── 20-fileinfo.ini -> /etc/php/8.5/mods-available/fileinfo.ini
│   ├── 20-ftp.ini -> /etc/php/8.5/mods-available/ftp.ini
│   ├── 20-gettext.ini -> /etc/php/8.5/mods-available/gettext.ini
│   ├── 20-iconv.ini -> /etc/php/8.5/mods-available/iconv.ini
│   ├── 20-phar.ini -> /etc/php/8.5/mods-available/phar.ini
│   ├── 20-posix.ini -> /etc/php/8.5/mods-available/posix.ini
│   ├── 20-readline.ini -> /etc/php/8.5/mods-available/readline.ini
│   ├── 20-shmop.ini -> /etc/php/8.5/mods-available/shmop.ini
│   ├── 20-sockets.ini -> /etc/php/8.5/mods-available/sockets.ini
│   ├── 20-sysvmsg.ini -> /etc/php/8.5/mods-available/sysvmsg.ini
│   ├── 20-sysvsem.ini -> /etc/php/8.5/mods-available/sysvsem.ini
│   ├── 20-sysvshm.ini -> /etc/php/8.5/mods-available/sysvshm.ini
│   └── 20-tokenizer.ini -> /etc/php/8.5/mods-available/tokenizer.ini
├── php-fpm.conf
├── php.ini
└── pool.d
    └── www.conf
```

Socket created:

```bash
lab-admin@lab-server:~$ ls -la /run/php/
total 4
drwxr-xr-x  2 www-data www-data  100 Feb  1 20:02 .
drwxr-xr-x 36 root     root     1040 Feb  1 20:02 ..
-rw-r--r--  1 root     root        5 Feb  1 20:02 php8.5-fpm.pid
srw-rw----  1 www-data www-data    0 Feb  1 20:02 php8.5-fpm.sock
lrwxrwxrwx  1 root     root       30 Feb  1 20:02 php-fpm.sock -> /etc/alternatives/php-fpm.sock
lab-admin@lab-server:~$ sudo ls -la /run/php/php8.5-fpm.sock
srw-rw---- 1 www-data www-data 0 Feb  1 20:02 /run/php/php8.5-fpm.sock
```

Process Tree Created:

```bash
lab-admin@lab-server:~$ ps aux | grep php-fpm
root       13752  0.0  0.5 209956 21724 ?        Ss   20:02   0:00 php-fpm: master process (/etc/php/8.5/fpm/php-fpm.conf)
www-data   13754  0.0  0.2 210464  7980 ?        S    20:02   0:00 php-fpm: pool www
www-data   13755  0.0  0.2 210464  7980 ?        S    20:02   0:00 php-fpm: pool www
lab-adm+   14029  0.0  0.0   6480  2244 pts/0    S+   20:10   0:00 grep --color=auto php-fpm
l
```

Systemd Symlink Created:
```bash
/etc/systemd/system/multi-user.target.wants/php8.5-fpm.service
→ /lib/systemd/system/php8.5-fpm.service
```
This means: PHP-FPM is enabled to start automatically when the system reaches "multi-user target" (normal boot without GUI).

Socket Analysis:
```
u_str LISTEN 0      4096                     /run/php/php8.5-fpm.sock 76636            * 0    users:(("php-fpm8.5",pid=13755,fd=10),("php-fpm8.5",pid=13754,fd=10),("php-fpm8.5",pid=13752,fd=8))
```
Breaking it down:
u_str = Unix stream socket (not TCP/IP)
LISTEN = Socket is listening for connections
/run/php/php8.5-fpm.sock = Path to the Unix socket
76636 = Inode number of the socket
users:(("php-fpm8.5",pid=13755,fd=10) = Processes using this socket:
pid=13755 = Worker process 1 (file descriptor 10)
pid=13754 = Worker process 2 (file descriptor 10)
pid=13752 = Master process (file descriptor 8) ← This is key!

1. Master Process Listens (pid=13752, fd=8):
- Master process (PID 13752) has the socket open on file descriptor 8
- It accepts new connections from Nginx
- Distributes requests to workers

2. Worker Processes Connected (pid=13754,13755, fd=10):
- Each worker has the socket on file descriptor 10
- They can receive requests from the master
- Actually execute PHP code

Communication Flow:
```
Nginx → Connects to socket → Master (fd=8) accepts → Master assigns to Worker → Worker (fd=10) processes → Returns response
```

```
┌─────────────────────────────────────────────────────────┐
│                 PHP-FPM Socket Architecture             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  MASTER PROCESS (pid=13752)                             │
│  File Descriptor 8 ──┐                                  │
│                      │ Listens for NEW connections      │
│  WORKER 1 (pid=13754)│                                  │
│  File Descriptor 10 ─┼─┐                                │
│                      │ │ Can PROCESS requests           │
│  WORKER 2 (pid=13755)│ │                                │
│  File Descriptor 10 ─┘ │                                │
│                        │                                │
│  ┌─────────────────────┘                                │
│  │                                                      │
│  ▼                                                      │
│  ┌─────────────────────────────────────────────────┐    │
│  │  UNIX SOCKET: /run/php/php8.5-fpm.sock (inode 76636) │
│  └─────────────────────────────────────────────────┘    │
│          ↑                                              │
│          │                                              │
│  ┌───────┴───────┐                                      │
│  │    Nginx      │  (When installed)                    │
│  │  fastcgi_pass │                                      │ 
│  └───────────────┘                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

What This Means for Nginx:
When you configure Nginx, this line will work:

nginx
fastcgi_pass unix:/run/php/php8.5-fpm.sock;
Because:

Socket exists at that path

PHP-FPM master is listening on it (fd=8)

Workers are ready to process requests (fd=10)

Bottom Line:
PHP-FPM socket is perfectly configured and ready! The ss output shows:

- Unix stream socket listening
- Master process (fd=8) accepting connections
- Two worker processes (fd=10) ready to execute PHP
- All processes sharing the same socket inode (76636)

```bash
php-fpm8.5 -i | grep "Server API"
Server API => FPM/FastCGI
Server API (SAPI) Abstraction Layer => Andi Gutmans, Shane Caraveo, Zeev Suraski
```


PHP-FPM (What you'll install):
bash
sudo systemctl start php8.5-fpm
 1. Master process starts
 2. Creates worker pool (e.g., 10 workers)
 3. Workers wait for requests
 4. Nginx sends PHP requests via FastCGI
 5. Worker processes request, returns response
 6. Worker waits for next request
 7. Process stays alive (persistent)
FastCGI Protocol Explained:
Nginx to PHP-FPM communication:

nginx
``` Nginx config line:
fastcgi_pass unix:/run/php/php8.5-fpm.sock;
```
What happens:
 1. Nginx receives: GET /index.php
 2. Nginx sends via FastCGI to socket:
    - Script filename: /var/www/html/index.php
    - Query string: (empty)
    - Request method: GET
    - Headers, cookies, etc.
 3. PHP-FPM receives, executes PHP
 4. Returns HTML via FastCGI
 5. Nginx sends HTML to browser