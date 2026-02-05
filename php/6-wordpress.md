# WordPress 

### Architecture & Database Essentials

 The "Brain" (Database): Acts as the central memory. It stores all text content (posts, pages, comments), user metadata, and site settings (timezones, site titles). This architecture allows for rapid searching, filtering, and multi-user collaboration.
 The "Process": WordPress is a collection of PHP files. When a link is clicked, these files execute a query to the database, pull the specific data needed, and build the HTML page on-the-fly (dynamically) for the visitor.
 
 Security: Passwords are encrypted. WordPress uses one-way hashing (salting) so that passwords never appear as plain text in the database. Even a database administrator cannot see your actual password.
 
 Themes: These control the visual design. While they rely heavily on .css for styling, they are packages of PHP, CSS, and JS files that define the 
 site's layout and appearance.
 
 Plugins: These add functionality. They are primarily .php scripts that inject new features into the core software, such as contact forms, SEO tools, or e-commerce capabilities.
 
 Data Types: Information is stored in specialized MySQL fields. It uses Longtext for blog bodies, Varchar for short strings like titles, and DateTime for timestamps. Complex configurations are often stored as "serialized" strings.
 
 Media & Content: Media files (images, videos, PDFs) live only in the physical /wp-content/uploads/ folder on the server. The database does not store the actual image; it only stores the URL path pointing to where the file is located.
 
 Portability: To migrate or back up a site, you must move two components: the Database (text, logic, and settings) and the wp-content folder (which contains your uploaded media, themes, and plugins).
 
 User Hierarchy: The system manages permissions for various roles, including Admins (full system control), Authors/Editors (content creation), and Developers (managing the underlying code and server-side database structure).

 ## Configure the MySQL database for Wordpress

 1. Create the WordPress Database & User 
Log into your MySQL shell (using the password you created during the security setup):

sudo mysql -u root -p

Once inside, run these SQL commands one by one:

 -- 1. Create a database for your site
 CREATE DATABASE wordpress_db DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
 
 -- 2. Create a dedicated user (replace 'your_password' with a strong one)
 CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'your_password';
 
 -- 3. Grant the user full permissions for this specific database
 GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
 
 -- 4. Reload privileges to apply changes and exit
 FLUSH PRIVILEGES;
 EXIT;
Note: Using a dedicated user (rather than 'root') is a critical security step for any production site.

## Install required modules
WordPress is written in PHP, so your Ubuntu server needs the PHP interpreter and a few specific "helper" modules to talk to MySQL and handle images.

`sudo apt install php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip -y`


## Install Wordpress

Once your database and PHP are ready, you can pull the latest WordPress files into your web directory:
```bash
 cd /tmp
 wget https://wordpress.org/latest.tar.gz
 tar -xvf latest.tar.gz
 sudo cp -a wordpress/. /var/www/html/
```

## Configure Nginx for Wordpress

To run WordPress on Nginx, you must configure a Server Block (similar to an Apache Virtual Host). Since WordPress is written in PHP, Nginx acts as the "front door" and passes the PHP code to a processor called PHP-FPM (FastCGI Process Manager).

1. Create the Nginx Configuration

Create a new configuration file for your site (replace example.com with your IP or domain):

```bash
 sudo nano /etc/nginx/sites-available/wordpress
```

Paste the following configuration into the file:

```
nginx
 server {
     listen 80;
     server_name your_server_ip; # Replace with your IP or domain
     root /var/www/html;
     index index.php index.html index.htm;
 
     location / {
         try_files $uri $uri/ /index.php?$args;
     }
 
     location ~ \.php$ {
         include snippets/fastcgi-php.conf;
         fastcgi_pass unix:/var/run/php/php8.1-fpm.sock; # Check your PHP version (8.1, 8.2, etc.)
     }
 
     location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
         expires max;
         log_not_found off;
     }
 }
```

Note: Run 
`ls /var/run/php/`
 to verify your exact php-fpm.sock version (e.g., php8.1-fpm.sock).

2. Enable the Configuration

Link the file to the sites-enabled directory and test for errors:

```bash
 # Enable the site
 sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
 
 # Remove the default Nginx "Welcome" page
 sudo rm /etc/nginx/sites-enabled/default
 
 # Test configuration for syntax errors
 sudo nginx -t
 
 # Reload Nginx to apply changes
 sudo systemctl reload nginx
```

3. Final Permissions Check

Nginx needs to "own" the files to upload images and run updates.

```bash
 sudo chown -R www-data:www-data /var/www/html
 sudo find /var/www/html/ -type d -exec chmod 755 {} \;
 sudo find /var/www/html/ -type f -exec chmod 644 {} \;
```

Open your browser and go to http://your_server_ip. You will see the WordPress 5-minute installation wizard.