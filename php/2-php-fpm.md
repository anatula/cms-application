# 2) PHP-FPM Installation

## FastCGI: The Protocol That Made PHP Scalable

## Historical Context & The Problem

In the early web (circa 1993), dynamic content was served via **CGI (Common Gateway Interface)**, designed by the NCSA team. CGI worked by having the web server (like Apache) `fork()` a new OS process for every single request. This child process would `setenv()` with request data (URL, headers, etc.) and then `execve()` to replace itself with a runtime like the PHP interpreter. After generating a response, the process died. This "fork-and-exec-per-request" model incurred massive OS overhead, making it slow and unable to handle concurrent traffic.

## FastCGI: The 1996 Specification

**FastCGI**, created by Open Market, solved this by defining a **binary protocol** for efficient, persistent communication between a web server and an application process. It replaced CGI's process creation with **socket communication** (over TCP or Unix sockets) and process pools.

## Core Technical Mechanism

FastCGI is a **binary RPC protocol**. The web server (client) and application (server) exchange structured records over a persistent connection.

*   **Binary Protocol**: Unlike text-based HTTP, FastCGI uses tightly packed binary records for speed. A record header defines type, request ID, and content length, allowing for direct memory mapping without text parsing.
*   **Request Multiplexing**: A single connection can carry multiple simultaneous requests, identified by a unique request ID in each record.
*   **Environment Passing**: To maintain backward compatibility with CGI's environment variable model, request metadata is transmitted as a series of key-value pairs in `FCGI_PARAMS` records. The application process parses these and uses `setenv()` to create the environment expected by the scripting runtime (e.g., PHP).
*   **Process Persistence**: The application process (e.g., PHP-FPM) starts once and remains alive in a pool. It waits for connections, parses incoming FastCGI records, executes the requested script, and sends back the response in `FCGI_STDOUT` records—all within the same process, eliminating fork/exec overhead.

## PHP-FPM: The PHP FastCGI Implementation

PHP-FPM (FastCGI Process Manager) is PHP's implementation of the FastCGI spec. It runs as a master daemon that manages pools of persistent **worker processes**.

1.  **Master Process**: Binds to a socket (`/run/php/php8.5-fpm.sock`), manages worker lifecycles, and reads configuration.
2.  **Worker Processes**: Long-lived PHP processes that wait on the socket. When a request arrives:
    *   They parse the FastCGI `FCGI_PARAMS` records to extract environment variables.
    *   They call `clearenv()` and `setenv()` to simulate a fresh CGI environment for the request.
    *   They execute the PHP script via an internal function call (`php_execute_script()`), **not** `execve()`. This keeps the worker alive.
    *   They capture the output, package it into `FCGI_STDOUT` records, and send it back to the web server.
    *   They reset and wait for the next request, reusing their compiled bytecode cache (OPcache) and database connections.

## Other modern alternatives
- WSGI/ASGI (Python, e.g., with Gunicorn/Uvicorn)
- Servlets (Java)
- Running the application as its own HTTP server and using Nginx as a reverse proxy (common with Node.js, modern Python/Go apps).

## SAPI (Server API)

SAPI is a C-level interface (C struct with function pointers) in PHP's source code that defines a contract for how PHP communicates with its environment. Each SAPI implementation (CLI, FPM, CGI, etc.) provides concrete implementations of these functions, enabling the same PHP engine to work in different contexts by connecting to:
- Terminals (via stdin/stdout - CLI)
- Web servers (via sockets - FPM/FastCGI)

**SAPI (Server API)** is PHP's C-level interface layer that enables a single PHP engine to operate in multiple execution environments through specialized binaries.

### Source Structure
Located at: [github.com/php/php-src/tree/master/sapi](https://github.com/php/php-src/tree/master/sapi)

Within the php-src repository:
- `sapi/` - SAPI implementations (different binaries)
  - `cli/` → `php` binary (CLI)
  - `fpm/` → `php-fpm` binary (FastCGI Process Manager)
  - `cgi/` → `php-cgi` binary (CGI)
  - `embed/` → Embedded PHP
- `Zend/` - Shared PHP execution engine
- `main/` - Shared PHP runtime

## Core Mechanism
Each SAPI implements a C `struct _sapi_module_struct` containing function pointers for:
- `ub_write()` - Environment-specific output (stdout/socket)
- `read_post()` - Environment-specific input (stdin/socket)
- `flush()`, `send_header()`, etc. - 20+ communication methods

The **same Zend engine** calls these function pointers, while each SAPI provides concrete implementations for its environment.

## Binary Compilation
CLI binary: links cli/*.c with shared core
`gcc -o php sapi/cli/.c Zend/.c main/*.c`

FPM binary: links fpm/*.c with same shared core
`gcc -o php-fpm sapi/fpm/.c Zend/.c main/*.c`

Result: Separate executables (`/usr/bin/php`, `/usr/sbin/php-fpm`) sharing identical PHP semantics but different communication channels.

## Environment Mapping
| SAPI | Binary | Input | Output | Execution Model |
|------|--------|-------|--------|-----------------|
| CLI | `php` | stdin/args | stdout/stderr | Single script, exit |
| FPM | `php-fpm` | FastCGI socket | FastCGI socket | Persistent daemon |


PHP maintains **one codebase** with **multiple interface implementations**. The SAPI layer abstracts environment communication, allowing identical PHP code to execute across terminal, web server, and embedded contexts via different compiled binaries.


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


## Install PHP-FP = PHP FastCGI Process Manager

```
┌─────────────┐     FastCGI     ┌─────────────┐
│   Nginx     │◄──over socket──►│   php-fpm   │
│  (web       │                 │   (PHP      │
│   server)   │                 │    server)  │
└─────────────┘                 └─────────────┘
       │                              │
       ▼                              ▼
  Serves static                 Executes PHP
  files, proxies                scripts, returns
  PHP requests                  HTML/JSON
```

Follow guide https://php.watch/articles/php-8-5-installation-upgrade-guide-debian-ubuntu#fpm

**PHP-FPM (FastCGI Process Manager)** is a production-grade PHP server daemon that runs as a standalone process, listening for PHP execution requests from web servers (Nginx/Apache) via FastCGI protocol.

PHP-FPM is the recommended way to integrate PHP with web servers such as Apache (with mpm_event), Nginx, Caddy.

The php8.5-fpm package (found in the previous ondrej PPA) installs PHP FPM server along with systemd units to automatically start the FPM server when the server starts.

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
```bash
u_str LISTEN 0      4096                     /run/php/php8.5-fpm.sock 76636            * 0    users:(("php-fpm8.5",pid=13755,fd=10),("php-fpm8.5",pid=13754,fd=10),("php-fpm8.5",pid=13752,fd=8))
```
Breaking it down:

`u_str` = Unix stream socket (not TCP/IP)

`LISTEN` = Socket is listening for connections

`/run/php/php8.5-fpm.sock` = Path to the Unix socket

`76636` = Inode number of the socket

`users` = (("php-fpm8.5",pid=13755,fd=10) = Processes using this socket:
pid=13755 = Worker process 1 (file descriptor 10)
pid=13754 = Worker process 2 (file descriptor 10)
pid=13752 = Master process (file descriptor 8)

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
┌─────────────────────────────────────────┐
│        PHP-FPM MASTER PROCESS           │
│        PID: 13752 (runs as root)        │
├─────────────────────────────────────────┤
│ • Creates socket                        │
│ • Manages workers                       │
│ • Monitors health                       │
│ • Handles reloads                       │
│ • Does NOT run PHP code                 │
│                                         │
│ Socket: /run/php/php8.5-fpm.sock (fd:8) │
└───────────────────┬─────────────────────┘
                    │
              Forks workers
                    │
     ┌──────────────┼──────────────┐
     │              │              │
     ▼              ▼              ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ WORKER1 │  │ WORKER2 │  │ WORKERN │
│ PID:    │  │ PID:    │  │ ...     │
│ 13754   │  │ 13755   │  │         │
│ www-data│  │ www-data│  │ www-data│
├─────────┤  ├─────────┤  ├─────────┤
│ • Runs  │  │ • Runs  │  │ • Runs  │
│   PHP   │  │   PHP   │  │   PHP   │
│   code  │  │   code  │  │   code  │
│ • fd:10 │  │ • fd:10 │  │ • fd:10 │
└─────────┘  └─────────┘  └─────────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
          All connect to same socket
                    │
                    ▼
        ┌────────────────────────────┐
        │  UNIX SOCKET               │
        │  /run/php/php8.5-fpm.sock  │
        └────────────────────────────┘
                    ▲
                    │
        ┌───────────┴───────────┐
        │      NGINX            │
        │  fastcgi_pass unix:...│
        └───────────────────────┘

FLOW: Nginx → Socket → Any available Worker → PHP execution → Response
```

What This Means for Nginx:
When you configure Nginx, this line will work:

`fastcgi_pass unix:/run/php/php8.5-fpm.sock;`

Because:
- Socket exists at that path
- PHP-FPM master is listening on it (fd=8)
- Workers are ready to process requests (fd=10)

Bottom Line:
PHP-FPM socket is perfectly configured and ready!

- Unix stream socket listening
- Master process (fd=8) accepting connections
- Two worker processes (fd=10) ready to execute PHP
- All processes sharing the same socket inode (76636)


PHP-FPM (What you'll install):
```bash
sudo systemctl start php8.5-fpm
 1. Master process starts
 2. Creates worker pool (e.g., 10 workers)
 3. Workers wait for requests
 4. Nginx sends PHP requests via FastCGI
 5. Worker processes request, returns response
 6. Worker waits for next request
 7. Process stays alive (persistent)
```

### Security

The PHP-FPM service is properly configured with security best practices. It runs under the dedicated `www-data` user with restrictive file permissions (0660), ensuring *only the web server can communicate with PHP processes*, effectively isolating web applications from system resources and preventing privilege escalation in case of compromise.

## System Impact

### What Changed:

✅ `/usr/sbin/php-fpm8.5` - PHP-FPM daemon binary installed

✅ `php8.5-fpm.service` - Auto-started systemd service that launches PHP-FPM master process, which then spawns www-data worker processes

✅ `/run/php/php8.5-fpm.sock` - Unix socket created for web server to PHP communication

✅ `/run/php/php-fpm.sock` → Symlink via /etc/alternatives/ system for version switching (allows changing PHP versions without updating web server configs)

✅ `/etc/php/8.5/fpm/` - Configuration directory with pool settings and php-fpm.conf

✅ Master/Worker Architecture - Root-owned master process manages www-data worker pool for request processing

### What Did NOT Change/Not Included:

❌ No CLI PHP binary (/usr/bin/php8.5 not installed - that's separate php8.5 package)

❌ No new users created (uses existing www-data user for worker processes)

❌ No environment variables or PATH modifications

❌ No web server configuration modified (nginx/apache need manual setup to use the PHP-FPM socket)

❌ No PHP modules loaded (need separate php8.5-* packages for additional functionality)

## Extra 

### /usr/bin/ vs /usr/sbin/

- `/usr/bin/` - User binaries (commands for ALL users) These are tools any user might need for daily work.
Examples: php, python, ls, cat, grep, nano, vim, curl, wget, git, node, docker (client), mysql (client), gcc, make, tar, zip, ssh, scp

- `/usr/sbin/` - System binaries (commands for SYSTEM ADMINISTRATION) These are for system management, daemons, and low-level operations.
Examples: php-fpm8.5, nginx, apache2, mysqld, iptables, ip, ifconfig (deprecated), route, fdisk, parted, lvcreate, vgcreate

### what is /etc/alternatives ?

It's a directory managed by the Debian alternatives system (`update-alternatives`) that acts as a symlink-based selector for commands with multiple implementations. 
```bash
lab-admin@lab-server:~$ which update-alternatives
/usr/bin/update-alternatives

lab-admin@lab-server:~$ man update-alternatives
```

It lets you install several versions of similar software (like different text editors or Java runtimes) and choose which one runs by default when you type a generic command (e.g., `editor`). You can view the current selection with `update-alternatives --display editor` or `ls -l /etc/alternatives/`, list all options with `--list`, and change it interactively with `sudo update-alternatives --config editor`—this is the recommended way to safely switch, for example, from nano to vim as your system's default editor.

