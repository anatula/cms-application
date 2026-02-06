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