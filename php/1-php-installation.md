# PHP 8.5 Installation Summary - Ubuntu 22.04

What .so Files Are (Shared Libraries):
Definition:
A .so file (Shared Object) is compiled C code that can be loaded by programs at runtime.

Running Services:

phpsessionclean.timer  # Cleans PHP session files every 30 mins
Not PHP itself - just a cleanup job

Runs phpsessionclean script periodically

Cleans up old session files in /var/lib/php/sessions/

What ACTUALLY loads when you run php8.5 script.php:
text
1. Binary: /usr/bin/php8.5 (C program)
   ↓
2. Built-in: Core, date, json, etc. (compiled into binary)
   ↓
3. Extensions: Loads .so files via .ini configs:
   - /usr/lib/php/20250925/pdo.so
   - /usr/lib/php/20250925/readline.so
   - etc. (all .so files you saw)
   ↓
4. Your script runs with all functions available

## What Was Installed:

Installation Source & Version:
Source: PPA (ppa:ondrej/php)
PHP Version: 8.5.x (latest)
Ubuntu: 22.04.2 LTS (Jammy Jellyfish)
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

Current API

php8.5 -i | grep "Server API"
Server API => Command Line Interface