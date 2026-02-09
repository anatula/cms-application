# Configure WordPress and Plugins/Themes

1. Open your browser to: http://localhost/ or the server's IP
2. Select language and click "Continue"
3. Enter site information:
 -  Site Title: My WordPress Site
 - Username: Choose an admin username (not "admin")
 - Password: Click "Generate" for a strong password
 - Your Email: Enter a valid email address
4. Click "Install WordPress"
5. Click "Log In" and enter your credentials

Then, we are done! we'll get the message:
```text
Success!
WordPress has been installed. Thank you, and enjoy!
```
## Plugins

A plugin is PHP code that adds functionality to WordPress.

Examples of Plugins:
- reSmush.it = Compresses images automatically
- WooCommerce = Adds e-commerce features
- Yoast SEO = Search engine optimization tools
- Contact Form 7 = Creates contact forms

Plugins live:

```bash
# Default WordPress structure:
/usr/share/nginx/html/
├── wp-content/
│   └── plugins/           # ← PLUGINS GO HERE
│       ├── akismet/       # Example plugin 1
│       ├── hello.php      # Example plugin 2
│       └── resmushit/     # Your plugin
```

### Install a plugin 

#### Via WordPress Admin

1. Go to http://[your-server-ip]/wp-admin/
2. Login with your credentials
3. Left menu: Plugins → Add Plugin
4. Search: "reSmush.it"
Installed:
Version: 1.0.4
Author: ShortPixel
Last Updated: 2 months ago
Requires WordPress Version: 4.0.0 or higher
Compatible up to: 6.9.1
Requires PHP Version: 7.4 or higher
5. Click "Install" then "Activate"

#### Manually via SSH

```bash
# Navigate to plugins directory
cd /usr/share/nginx/html/wp-content/plugins/

# Download plugin
sudo wget https://downloads.wordpress.org/plugin/resmushit-image-optimizer.latest-stable.zip

# Extract
sudo unzip resmushit-image-optimizer.latest-stable.zip

# Set permissions
sudo chown -R www-data:nginx resmushit-image-optimizer/
sudo find resmushit-image-optimizer/ -type d -exec chmod 755 {} \;
sudo find resmushit-image-optimizer/ -type f -exec chmod 644 {} \;
```

These commands standardize permissions for WordPress plugin directories and files to ensure proper functionality and security:

#### Command Breakdown:
```bash
# Set all directories to 755 (read/write/execute for owner, read/execute for others)
sudo find resmushit-image-optimizer/ -type d -exec chmod 755 {} \;
```

- `find resmushit-image-optimizer/` = Look inside the resmushit-image-optimizer/ folder
- `-type d` = Find only directories (not files)
- `-exec chmod 755 {} \;` = For each directory found, run chmod 755 on it
- `{}` = Placeholder that gets replaced with each directory path found
- `\;` = End of the -exec command

Result: All directories inside resmushit-image-optimizer/ get permission 755

```bash
# Set all files to 644 (read/write for owner, read-only for others)  
sudo find resmushit-image-optimizer/ -type f -exec chmod 644 {} \;
```

Same logic but:
- `-type f` = Find only files (not directories)
- `chmod 644` = Set file permissions to 644

Result: All files inside resmushit-image-optimizer/ get permission 644

#### Why Use find Instead of Simple chmod -R?
Problem with chmod -R:

```bash
# WRONG approach:
sudo chmod -R 755 resmushit-image-optimizer/
This would set BOTH directories AND files to 755 which is:
```
✅ Correct for directories (need 755 to be enterable)

❌ WRONG for files (PHP files shouldn't be executable!)

#### Visual Example of Plugin Structure:
```bash
resmushit-image-optimizer/
├── resmushit.php                 # Main plugin file (should be 644)
├── uninstall.php                 # Uninstall script (644)
├── js/                           # Directory (755)
│   ├── resmushit.js              # JavaScript file (644)
│   └── admin.js                  # JavaScript file (644)
├── css/                          # Directory (755)
│   └── style.css                 # CSS file (644)
├── includes/                     # Directory (755)
│   ├── class-resmushit.php       # PHP class (644)
│   └── helpers.php               # Helper functions (644)
└── languages/                    # Directory (755)
    └── resmushit.pot             # Translation file (644)
```

#### Check WordPress database 

For plugin activation:

```bash
sudo mysql wordpress_db -e "SELECT LENGTH(option_value) as size, option_value LIKE '%resmushit%' as has_resmushit FROM wp_options WHERE option_name = 'active_plugins';"

a:1:{i:0;s:39:"resmushit-image-optimizer/resmushit.php";}
a:1 = Array with 1 element
i:0 = Index 0 (first plugin)
s:39 = String of 39 characters
"resmushit-image-optimizer/resmushit.php" = The plugin's main file
```

For plugin setting:

```bash
sudo mysql wordpress_db -e "SELECT option_name, LENGTH(option_value) as size FROM wp_options WHERE option_name LIKE '%resmushit%';"
+---------------------------------+------+
| option_name                     | size |
+---------------------------------+------+
| resmushit_qlty                  |    2 |
| resmushit_on_upload             |    1 |
| resmushit_statistics            |    1 |
| resmushit_total_optimized       |    1 |
| resmushit_cron                  |    1 |
| resmushit_cron_lastaction       |    1 |
| resmushit_cron_lastrun          |    1 |
| resmushit_cron_firstactivation  |    1 |
| resmushit_preserve_exif         |    1 |
| resmushit_remove_unsmushed      |    1 |
| resmushit_has_no_backup_files   |    1 |
| resmushit_notice_close_eoldec23 |    1 |
+---------------------------------+------+
```

What This Means:

✅ Plugin files installed in /wp-content/plugins/resmushit-image-optimizer/

✅ Plugin activated in WordPress database

✅ WordPress loads resmushit.php on every page request

✅ Plugin functionality should be available

### Problem: DHCP changed the server's IP
The server's IP changed, but WordPress still thinks it's at the old one.

#### Quick Fix:
1. Update WordPress Database
```bash
# Update to the new IP
sudo mysql wordpress_db << 'EOF'
UPDATE wp_options 
SET option_value = 'http://192.168.1.XXX' 
WHERE option_name IN ('siteurl', 'home');
FLUSH PRIVILEGES;
EOF
```