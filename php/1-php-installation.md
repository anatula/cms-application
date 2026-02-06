# 1) Install PHP latest version 8.5

PHP itself is written primarily in the C programming language

## Steps for Ubuntu: 22.04.2 LTS (Jammy Jellyfish)
For Ubuntu, guide: https://www.php.net/downloads.php?usage=web&os=linux&osvariant=linux-ubuntu&version=8.5

```bash
# Add the ondrej/php repository.
sudo apt update
sudo apt install -y software-properties-common
sudo LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php -y
sudo apt update

# Install PHP.
sudo apt install -y php8.5
```
### LC_ALL=C.UTF-8

`C` reserved identifier triggering byte-wise sorting and US-ASCII encoding
`C.UTF-8` Variant of C locale that keeps ASCII rules (collation, formatting) but uses UTF-8 character encoding. Allows Unicode text while maintaining simple byte-order sorting. 
`LC_ALL` is the environment variable that overrides all the other localisation settings

### What is software-properties-common
As described in `apt show software-properties-common`: This software provides an abstraction of the used apt repositories. It allows you to easily manage your distribution and independent software vendor software sources.
It enables to:
- Add/Remove Software Repositories - Easily add new sources of software (like PPAs - Personal Package Archives) through simple commands
- Manage Repository Settings - Enable/disable repositories without editing configuration files manually
- Access to Handy Commands like:
```
add-apt-repository - Adds a new software repository
apt-add-repository - Same as above
apt-key - Manages authentication keys for repositories
```
Instead of manually editing `/etc/apt/sources.list`, you can simply do:
```
sudo add-apt-repository ppa:some-software/ppa
# Add a PPA for a software
sudo add-apt-repository ppa:some-software/ppa
# Remove it
sudo add-apt-repository --remove ppa:some-software/ppa
```

## What Was Installed:

Installation Source & Version:
Source: PPA (ppa:ondrej/php)
PHP Version: 8.5.x (latest)
API Version: 20250925 (PHP 8.5 internal API)

### 1. Core Components:
- **PHP Interpreter**: `/usr/bin/php8.5` (C binary that executes PHP code)
- **PHAR Tool**: `/usr/bin/phar8.5` (PHP archive handler)
- **Symlink**: `/usr/bin/php → /usr/bin/php8.5`

### 2. PHP Extensions (C Libraries):
**17 .so files** in `/usr/lib/php/20250925/`:
- `pdo.so` - Database abstraction (empty shell)
- `readline.so` - CLI command editing
- `sockets.so` - Network programming
- `calendar.so`, `ctype.so`, `exif.so`, `fileinfo.so`, `ftp.so`
- `gettext.so`, `iconv.so`, `phar.so`, `posix.so`, `shmop.so`
- `sysvmsg.so`, `sysvsem.so`, `sysvshm.so`, `tokenizer.so`, `ffi.so`

### 3. Configuration Structure:
`/etc/php/8.5/` contains:
- `mods-available/` - 17 .ini files (extension=pdo.so, etc.)
- `cli/` - Command Line Interface config
  - `php.ini` - Main CLI configuration
  - `conf.d/` - 17 symlinks → mods-available/
- `apache2/` - Apache config (unused, ignore)

## Current State Verification:

### Key Command: `php -m` or `php8.5 -m`
**Output shows 37 "modules":**
- **15 Built-in components**: Core, date, json, session, filter, hash, libxml, openssl, pcre, random, Reflection, SPL, standard, zlib, uri
- **22 Loaded extensions**: From the 17 .so files (calendar, ctype, exif, FFI, fileinfo, ftp, gettext, iconv, lexbor, pcntl, PDO, Phar, posix, readline, shmop, sockets, sodium, sysvmsg, sysvsem, sysvshm, tokenizer, Zend OPcache)

## System Impact:

### What Changed:
✅ `/usr/bin/php8.5` - PHP command available
✅ `/usr/bin/php` - Symlink to php8.5  
✅ `php -m` / `php8.5 -m` - Shows 37 available modules
✅ `phpsessionclean.timer` - Cleans session files every 30 minutes
✅ Man pages: `man php8.5`, `man phar8.5`

### What Did NOT Change:
❌ No new users created (www-data existed previously)
❌ No environment variables added
❌ No PATH modifications
❌ No system services (except the cleanup timer)
❌ No shell/profile modifications

## Current Capabilities:

### Can Do:
- Run PHP scripts: `php script.php` or `php8.5 script.php`
- Check modules: `php -m` (shows 37 loaded components)
- Use CLI PHP: `php -a` (interactive mode)
- Built-in web server: `php -S localhost:8000` (development only)
- Basic functions: strings, arrays, files, JSON, dates

### Cannot Do (Yet):
- Connect to databases (no mysqli.so, pdo_mysql.so)
- Process images (no gd.so)
- Make HTTP requests (no curl.so)
- Serve PHP via web server (no PHP-FPM installed)
- Handle UTF-8 properly (no mbstring.so)

## Technical Reality:
- **PHP interpreter** = C program that reads/executes PHP code
- **Extensions** = C libraries (.so files) adding functionality
- **`php -m`** = Shows mix of built-in + loaded extensions
- **No PHP processes running** - Only executes when you run `php`
- **Minimal system footprint** - Just binaries and config files

## Bottom Line:
PHP 8.5 is installed as a command-line tool with basic functionality. `php -m` confirms 37 components available. It's dormant software on disk that only runs when explicitly executed. No web capabilities, no database support, no running services - just the PHP interpreter ready for CLI use.

### Current API

`php8.5 -i | grep "Server API"`
Server API => Command Line Interface

## Extra 

### **Shared Libraries (.so) vs Kernel Modules (.ko)**

#### **Similarities**
Both are compiled C code that can be loaded dynamically at runtime. Both allow extending system functionality without recompiling the core system. Both support versioning and dependency management.

#### **Key Differences**

**Shared Libraries (.so)**
- **Execution context**: User-space (ring 3)
- **Privilege level**: User process permissions only
- **Failure impact**: Only crashes the loading process
- **Memory management**: Uses malloc()/free() from user heap
- **Loading mechanism**: Loaded via `dlopen()` by the dynamic linker (`ld.so`)
- **Standard paths**: `/lib/`, `/usr/lib/`, `/usr/local/lib/`
- **Interface**: Normal function calls (ABI compliant)
- **Example**: `libssl.so` - Provides SSL/TLS encryption to applications like web browsers. A bug crashes only the browser.

**Kernel Modules (.ko)**
- **Execution context**: Kernel-space (ring 0)
- **Privilege level**: Full system access (supervisor mode)
- **Failure impact**: Can cause a kernel panic (total system crash)
- **Memory management**: Uses kmalloc()/kfree() from kernel memory pools
- **Loading mechanism**: Loaded via `insmod`/`modprobe` by the kernel module loader
- **Standard path**: `/lib/modules/$(uname -r)/kernel/`
- **Interface**: Kernel APIs and hooks into system call tables
- **Example**: `nf_conntrack.ko` - Linux kernel module for tracking network connections for firewalls. A bug crashes the entire OS.

#### **Finding .so Files in Ubuntu**

**Common Locations**
Core system libraries live in `/lib/` and `/usr/lib/`. Architecture-specific ones (for x86_64) are in `/usr/lib/x86_64-linux-gnu/`. Locally compiled software uses `/usr/local/lib/`.

**Essential System .so Files**
- `libc.so.6`: The GNU C Library, required by nearly every program.
- `ld-linux-x86-64.so.2`: The dynamic linker/loader itself.
- `libpthread.so.0`: POSIX threads implementation.
- `libm.so.6`: Standard math library.

**PHP Extensions (e.g., `pdo.so`)**
These are **shared libraries**, *not* kernel modules. They are loaded by the PHP interpreter (a user-space process).
- **Location**: `/usr/lib/php/<php-api-version>/` (e.g., `/usr/lib/php/20210902/pdo.so`)
- **Purpose**: Extend PHP's functionality at the application level.
- **Loading**: Via PHP's `dl()` function or the `extension=` directive in `php.ini`.
- **Check**: Run `php -m` to see loaded modules.

**How to Find Them**
```bash
# Find a specific library
find /usr/lib -name "libssl*.so*"

# See what libraries a program uses
ldd /usr/bin/php

# Locate PHP's pdo.so
find /usr -name "pdo.so" 2>/dev/null
```