# WordPress 

It's a collection of PHP scripts and files that sit in a web directory. Files you download:

```bash
/usr/share/nginx/html/
├── index.php          ← The main controller
├── wp-config.php      ← Configuration
├── wp-content/        │
├── wp-includes/       ├─ WordPress core
└── wp-admin/          │
```
### Dependencies (Already Installed)
https://wordpress.org/about/requirements/

- ✅ NGINX: Web server (listens on port 80) 
- ✅ PHP-FPM: PHP processor (via Unix socket)
- ✅ MySQL: Database server (port 3306)

### Runtime Behavior
- No new processes - Runs inside existing PHP-FPM workers
- No new services - Uses existing NGINX + PHP-FPM + MySQL
- No system users - Runs as www-data (existing PHP user)

### WordPress vs System Services
```bash
✅ INSTALLED & RUNNING:
├── nginx.service        (systemd service)
├── php8.5-fpm.service   (systemd service) 
└── mysql.service        (systemd service)

📁 TO BE ADDED:
└── /usr/share/nginx/html/wordpress/  (just files)
```

### Architecture & Database Essentials

 - The "Brain" (Database): Acts as the central memory. It stores all text content (posts, pages, comments), user metadata, and site settings (timezones, site titles). This architecture allows for rapid searching, filtering, and multi-user collaboration.

 - The "Process": WordPress is a collection of PHP files. When a link is clicked, these files execute a query to the database, pull the specific data needed, and build the HTML page on-the-fly (dynamically) for the visitor.
 
 - Security: Passwords are encrypted. WordPress uses one-way hashing (salting) so that passwords never appear as plain text in the database. Even a database administrator cannot see your actual password.
 
 - Themes: These control the visual design. While they rely heavily on .css for styling, they are packages of PHP, CSS, and JS files that define the site's layout and appearance.
 
 - Plugins: These add functionality. They are primarily .php scripts that inject new features into the core software, such as contact forms, SEO tools, or e-commerce capabilities.
 
 - Data Types: Information is stored in specialized MySQL fields. It uses Longtext for blog bodies, Varchar for short strings like titles, and DateTime for timestamps. Complex configurations are often stored as "serialized" strings.
 
 - Media & Content: Media files (images, videos, PDFs) live only in the physical `/wp-content/uploads/` folder on the server. The database does not store the actual image; it only stores the *URL path pointing to where the file is located.*
 
 - Portability: To migrate or backup a site, you must move two components: 
    - the Database (text, logic, and settings) 
    - wp-content folder (which contains your uploaded media, themes, and plugins)
 
 - User Hierarchy: The system manages permissions for various roles, including Admins (full system control), Authors/Editors (content creation), and Developers (managing the underlying code and server-side database structure).

## Installation Steps 

## 1) Verify current setup
```bash
1. Verify Current Setup
bash
# Check services are running
sudo systemctl status nginx php8.5-fpm mysql

# Check web root configuration
sudo grep "root" /etc/nginx/conf.d/default.conf
# Should show: root /usr/share/nginx/html;

# Test PHP is working
echo '<?php echo "PHP OK at " . date("H:i:s"); ?>' | sudo tee /usr/share/nginx/html/test.php
curl http://localhost/test.php
# Should output: PHP OK at [current time]
```

## 2) Install Required PHP Modules

```bash
# Update package list
sudo apt update

# Install WordPress required PHP modules
sudo apt install \
  php8.5-mysql \
  php8.5-curl \
  php8.5-gd \
  php8.5-intl \
  php8.5-mbstring \
  php8.5-soap \
  php8.5-xml \
  php8.5-xmlrpc \
  php8.5-zip -y

# Restart PHP-FPM to load new modules

# Verify modules are loaded
php-fpm8.5 -m | grep -E "(mysql|curl|gd|xml|zip|mbstring)"
```

### 3) Configure MySQL Database
MySQL root uses auth_socket so no password (check: `sudo mysql -u root -p -e "SELECT user, host, plugin FROM mysql.user WHERE user = 'root';"`) `sudo mysql` works without password. This is secure for local development
Its like a system-level authentication (must have sudo access!)

```bash
# This is CORRECT for your setup:
sudo mysql
# (enter MySQL shell, no password prompt)

# Run WordPress database commands:
CREATE DATABASE wordpress_db;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'WordPressDB123!';
GRANT ALL ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 4) Download WordPress Files
```bash
# Go to /tmp and download
cd /tmp
wget -q https://wordpress.org/latest.tar.gz

# Extract
tar -xzf latest.tar.gz

# Copy to your web root
sudo cp -a wordpress/. /usr/share/nginx/html/
```

### 5) Set Correct Permissions

The permission configuration ensures WordPress can function properly by aligning file ownership and access rights between the web server components. 

We change ownership to `www-data:nginx` so PHP-FPM (running as `www-data`) can execute WordPress scripts and write to directories like `wp-content/uploads`, while NGINX (in the `nginx group`) can read and serve all files. 

Directory permissions are set to 755 (read/write/execute for owner, read/execute for group and others) and file permissions to 644 (read/write for owner, read-only for group and others), with `wp-content/` specifically set to 775 to allow both PHP-FPM and NGINX to write files for uploads, plugins, and updates. 

This creates a secure, functional environment where WordPress has necessary write access without compromising security through overly permissive settings like 777.

```bash
# Set ownership (www-data:nginx for socket access)
sudo chown -R www-data:nginx /usr/share/nginx/html/

# Set directory permissions to 755, files to 644
sudo find /usr/share/nginx/html/ -type d -exec chmod 755 {} \;
sudo find /usr/share/nginx/html/ -type f -exec chmod 644 {} \;

# WordPress needs write access to wp-content
sudo chmod 775 /usr/share/nginx/html/wp-content/
```

### 6) Wordpress Configuration Step

#### 1) wp-config.php

The commands complete WordPress's essential configuration file in two critical parts. 

First, they generate and add eight unique cryptographic keys from WordPress.org's servers, which serve to encrypt session cookies, protect passwords in the database, and sign forms against malicious attacks, ensuring each installation has its own set of security secrets. 

Second, they add the remaining operational configuration: set the database table prefix (default 'wp_'), disable debug mode to avoid publicly exposing errors in production, define the absolute path where WordPress files reside using ABSPATH, and finally load the system core through the wp-settings.php file that initializes all functionality. Without these sections, WordPress would either fail to start correctly or be extremely vulnerable to security attacks.

```bash
# Navigate to web root
cd /usr/share/nginx/html/

# Create the main configuration with database settings
sudo tee wp-config.php << 'EOF'
<?php
define( 'DB_NAME', 'wordpress_db' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'WordPressDB123!' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8mb4' );
define( 'DB_COLLATE', '' );

// Security keys - unique for your site
EOF

# Add automatically generated security keys from WordPress.org
curl -s https://api.wordpress.org/secret-key/1.1/salt/ | sudo tee -a wp-config.php

# Add the remaining WordPress configuration
sudo cat >> wp-config.php << 'EOF'

$table_prefix = 'wp_';

define( 'WP_DEBUG', false );

if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

require_once ABSPATH . 'wp-settings.php';
EOF
```
#### 2) Fix file ownership
```bash
# Ensure the file is owned by PHP-FPM user and NGINX group
sudo chown www-data:nginx wp-config.php

# Set correct permissions (readable by PHP and NGINX, not writable by web)
sudo chmod 644 wp-config.php
```

#### 3) Test Database Connection

```bash
# Create a test script to verify database connectivity
echo '<?php
$conn = mysqli_connect("localhost", "wp_user", "WordPressDB123!", "wordpress_db");
if ($conn) {
    echo "✅ Database connection successful!";
    mysqli_close($conn);
} else {
    echo "❌ Error: " . mysqli_connect_error();
}
?>' | sudo tee dbtest.php

# Run the test
curl http://localhost/dbtest.php
# Expected output: ✅ Database connection successful!
```

### 7) Restart Services

```bash
# Reload PHP-FPM and NGINX to apply changes
sudo systemctl restart php8.5-fpm nginx
```

## Verification Commands
```bash
# Check wp-config.php is complete
ls -la wp-config.php
# Should show: -rw-r--r-- 1 www-data nginx

# Check file has security keys
grep "AUTH_KEY" wp-config.php
# Should show 8 security key lines

# Test WordPress is accessible
curl -I http://SERVER-IP-OR-LOCALHOST/
# Should return: HTTP/1.1 200 OK
```