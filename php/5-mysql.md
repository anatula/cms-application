## MySQL Architecture on Ubuntu 22.04

### Installation (Default Repository):
The most stable and common way to install MySQL is through the official Ubuntu repositories.
`sudo apt update`

### Install the MySQL Server package

 `sudo apt install mysql-server -y`

* Version & Distribution: Defaults to MySQL 8.0.x. Most users stick with the official Ubuntu repository version for stability. An Oracle-provided repository (via mysql-apt-config) is used by those requiring the absolute latest "point release" or MySQL 8.4 (LTS).

* Process & Service: Runs as a single background daemon named `mysqld`. It is managed via systemctl `sudo systemctl status mysql`

* Networking & Ports: By default, it listens on Port `3306`. It is configured to "bind" to 127.0.0.1 (localhost), meaning it refuses any external network connections for security.

* Connection Methods:
- TCP/IP: Used when apps connect via 127.0.0.1:3306.
- Unix Socket: A faster local connection method using the file located at `/var/run/mysqld/mysqld.sock`. This bypasses the network stack for better performance.

* Storage Directory: `/var/lib/mysql/`. This is the physical location on the disk where all databases (folders), tables (.ibd files), and system metadata live.

* Configuration Files: The primary settings (like memory limits and networking) are found in `/etc/mysql/mysql.conf.d/mysqld.cnf`

* Logging:
- Error Log: Located at `/var/log/mysql/error.log`. This is the primary file for troubleshooting crashes.
- System Log: Accessed via `sudo journalctl -u mysql` to check service-level start/stop issues.

* Memory Management: Although data is persistent on the disk, MySQL utilizes RAM via the InnoDB Buffer Pool to cache frequently accessed data, significantly increasing speed.

## System Impact

### What Changed:

✅ `/usr/sbin/mysqld` - MySQL server daemon (mysqld) installed

✅ `mysql.service` - Auto-started systemd service that launches MySQL server process

✅ `mysql` system user created - Dedicated user for running MySQL processes (UID/GID 113)

✅ `/etc/mysql/` - Configuration directory with my.cnf and conf.d/

✅ `/var/lib/mysql/` - Data directory created with system databases and secure initialization

✅ `/var/log/mysql/` - Log directory with error.log

✅ Root password secured - Random password generated, stored in `/etc/mysql/debian.cnf`

✅ `Port 3306` opened - MySQL listens on localhost only by default

✅ Socket `/var/run/mysqld/mysqld.sock` - Created for local connections

✅ MySQL client tools installed (`mysql`, `mysqladmin`, `mysqldump`, etc.)

### What Did NOT Change/Not Included:

❌ No remote access allowed (binds to 127.0.0.1 only by default)

❌ No firewall rules modified (UFW/iptables unchanged)

❌ No user databases created (only system databases: mysql, sys, information_schema performance_schema)

❌ No PHP/application connectivity configured (needs separate setup)

❌ No root password known to you initially (need to check debian.cnf or run sudo mysql with auth_socket)

❌ No replication or clustering configured (single standalone instance)