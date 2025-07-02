# SLiMS Pretty URL Implementation Guide

A comprehensive guide for implementing user-friendly URLs in SLiMS Bulian, supporting both Nginx and Apache web servers.

## 🎯 What are Pretty URLs?

Pretty URLs transform standard SLiMS query-based URLs into clean, user-friendly formats:

**Before (Standard URLs):**
```
https://domain.com/index.php?p=libinfo
https://domain.com/index.php?p=show_detail&id=12345
https://domain.com/index.php?p=member
```

**After (Pretty URLs):**
```
https://domain.com/libinfo
https://domain.com/sd=12345
https://domain.com/member
```

## 🚀 Features

- ✅ Clean, SEO-friendly URLs
- ✅ Support for both Nginx and Apache
- ✅ Backward compatibility with existing URLs
- ✅ Static asset handling (CSS, JS, images)
- ✅ Custom URL patterns for specific pages
- ✅ Zero modification to SLiMS core files

## 📋 URL Patterns Supported

| Pattern | Maps To | Description |
|---------|---------|-------------|
| `/page` | `index.php?p=page` | Basic page routing |
| `/sd=12345` | `index.php?p=show_detail&id=12345` | Short detail URL |

## 🔧 Nginx Configuration

### Step 1: Create Nginx Configuration File

Create a new file `/etc/nginx/sites-available/slims-pretty-url`:

```nginx
server {
    listen 80;
    server_name domain.com www.domain.com;
    root /var/www/html/slims;
    index index.php index.html;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
    
    # Static files with long cache
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
    
    # Handle PHP files directly
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    
    # SLiMS Pretty URL Rules
    location / {
        # Try direct file/directory first
        try_files $uri $uri/ @slims_rewrite;
    }
    
    # SLiMS rewrite rules
    location @slims_rewrite {
        # Custom alias: /sd=12345 -> show_detail&id=12345
        if ($uri ~ "^/sd=([0-9]+)/?$") {
            rewrite ^/sd=([0-9]+)/?$ /index.php?p=show_detail&id=$1 last;
        }
        
        # Detail pages: /detail/12345 -> show_detail&id=12345
        if ($uri ~ "^/detail/([0-9]+)/?$") {
            rewrite ^/detail/([0-9]+)/?$ /index.php?p=show_detail&id=$1 last;
        }
        
        # Three-part URLs: /page/action/id
        if ($uri ~ "^/([^/]+)/([^/]+)/([^/]+)/?$") {
            rewrite ^/([^/]+)/([^/]+)/([^/]+)/?$ /index.php?p=$1&action=$2&id=$3 last;
        }
        
        # Two-part URLs: /page/action
        if ($uri ~ "^/([^/]+)/([^/]+)/?$") {
            rewrite ^/([^/]+)/([^/]+)/?$ /index.php?p=$1&action=$2 last;
        }
        
        # Single page URLs: /page
        if ($uri ~ "^/([^/]+)/?$") {
            rewrite ^/([^/]+)/?$ /index.php?p=$1 last;
        }
        
        # Fallback to index.php
        rewrite ^(.*)$ /index.php last;
    }
    
    # Deny access to sensitive files
    location ~ /\. {
        deny all;
    }
    
    location ~ ~$ {
        deny all;
    }
    
    # Error pages
    error_page 404 /index.php;
    error_page 500 502 503 504 /50x.html;
}
```

### Step 2: Enable the Configuration

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/slims-pretty-url /etc/nginx/sites-enabled/

# Test configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
```

## 🔧 Apache Configuration

### Option 1: Virtual Host Configuration

Add to your Apache virtual host configuration:

```apache
<VirtualHost *:80>
    ServerName domain.com
    ServerAlias www.domain.com
    DocumentRoot /var/www/html/slims
    
    # Enable mod_rewrite
    RewriteEngine On
    
    # Security headers
    Header always set X-Frame-Options SAMEORIGIN
    Header always set X-Content-Type-Options nosniff
    Header always set X-XSS-Protection "1; mode=block"
    
    # Static file caching
    <FilesMatch "\.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$">
        ExpiresActive On
        ExpiresDefault "access plus 1 year"
        Header append Cache-Control "public"
    </FilesMatch>
    
    # SLiMS Pretty URL Rules
    
    # Skip rewrite for existing files and directories
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    
    # Custom alias: /sd=12345 -> show_detail&id=12345
    RewriteRule ^sd=([0-9]+)/?$ index.php?p=show_detail&id=$1 [L,QSA]
      
    # Single page URLs: /page
    RewriteRule ^([^/]+)/?$ index.php?p=$1 [L,QSA]
    
    # Protect sensitive files
    <Files ~ "^\.(htaccess|htpasswd|ini|log|sh|sql)">
        Require all denied
    </Files>
    
    ErrorLog ${APACHE_LOG_DIR}/slims_error.log
    CustomLog ${APACHE_LOG_DIR}/slims_access.log combined
</VirtualHost>
```

### Option 2: .htaccess File

Create `.htaccess` in your SLiMS root directory:

```apache
# SLiMS Pretty URL Configuration
RewriteEngine On

# Security headers
<IfModule mod_headers.c>
    Header always set X-Frame-Options SAMEORIGIN
    Header always set X-Content-Type-Options nosniff
    Header always set X-XSS-Protection "1; mode=block"
</IfModule>

# Static file caching
<IfModule mod_expires.c>
    ExpiresActive On
    ExpiresByType text/css "access plus 1 year"
    ExpiresByType application/javascript "access plus 1 year"
    ExpiresByType image/png "access plus 1 year"
    ExpiresByType image/jpg "access plus 1 year"
    ExpiresByType image/jpeg "access plus 1 year"
    ExpiresByType image/gif "access plus 1 year"
    ExpiresByType image/ico "access plus 1 year"
    ExpiresByType image/svg+xml "access plus 1 year"
</IfModule>

# Skip rewrite for existing files and directories
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d

# Custom alias: /sd=12345 -> show_detail&id=12345
RewriteRule ^sd=([0-9]+)/?$ index.php?p=show_detail&id=$1 [L,QSA]

# Detail pages: /detail/12345 -> show_detail&id=12345
RewriteRule ^detail/([0-9]+)/?$ index.php?p=show_detail&id=$1 [L,QSA]

# Three-part URLs: /page/action/id
RewriteRule ^([^/]+)/([^/]+)/([^/]+)/?$ index.php?p=$1&action=$2&id=$3 [L,QSA]

# Two-part URLs: /page/action
RewriteRule ^([^/]+)/([^/]+)/?$ index.php?p=$1&action=$2 [L,QSA]

# Single page URLs: /page
RewriteRule ^([^/]+)/?$ index.php?p=$1 [L,QSA]

# Protect sensitive files
<Files ~ "^\.(htaccess|htpasswd|ini|log|sh|sql)">
    Require all denied
</Files>
```

## 🧪 Testing Pretty URLs

### 1. Test Basic Pages

```bash
# Test these URLs in your browser:
https://domain.com/libinfo
https://domain.com/visitor
https://domain.com/member
https://domain.com/bibliography
```

### 2. Test Detail Pages

```bash
# Replace 12345 with actual biblio_id from your database
https://domain.com/detail/12345
https://domain.com/sd=12345
```

### 3. Test Static Assets

```bash
# Verify CSS and JS files load correctly
https://domain.com/libinfo  # Check if styles are applied
https://domain.com/bibliography  # Check if search forms work
```

### 4. Test Complex URLs

```bash
# Test multi-parameter URLs
https://domain.com/bibliography/search/advanced
https://domain.com/member/register/form
```

## 🔍 Troubleshooting

### Common Issues and Solutions

#### 1. 404 Errors on Pretty URLs

**Problem:** Pretty URLs return 404 Not Found

**Solutions:**
- **Nginx:** Check if `try_files` directive is correctly configured
- **Apache:** Ensure `mod_rewrite` is enabled and `.htaccess` is readable
- Verify web server has read permissions on configuration files

#### 2. Static Assets Not Loading

**Problem:** CSS/JS files return 404 when accessing pretty URLs

**Solutions:**
- Check static file location patterns in configuration
- Ensure static files are excluded from rewrite rules
- Verify file permissions and paths

#### 3. Infinite Redirect Loops

**Problem:** Pages keep redirecting endlessly

**Solutions:**
- Check for conflicting rewrite rules
- Ensure `RewriteCond` excludes existing files
- Review order of rewrite rules

#### 4. Original URLs Still Working

**Problem:** Both pretty and original URLs work (not an issue, but for SEO)

**Solutions:**
- Implement canonical URLs in HTML `<head>`
- Add 301 redirects from old to new URLs if needed

### Debug Tools

Create a debug file to test rewrite behavior:

```php
<?php
// File: debug_rewrite.php
echo "<h3>URL Rewrite Debug</h3>";
echo "<strong>REQUEST_URI:</strong> " . $_SERVER['REQUEST_URI'] . "<br>";
echo "<strong>QUERY_STRING:</strong> " . $_SERVER['QUERY_STRING'] . "<br>";
echo "<strong>GET Parameters:</strong><br>";
print_r($_GET);
echo "<strong>SERVER_NAME:</strong> " . $_SERVER['SERVER_NAME'] . "<br>";
echo "<strong>DOCUMENT_ROOT:</strong> " . $_SERVER['DOCUMENT_ROOT'] . "<br>";
?>
```

## 📚 Additional Configuration

### Custom URL Patterns

Add more specific patterns for your needs:

**Nginx:**
```nginx
# Member profile pages: /member/profile/123
if ($uri ~ "^/member/profile/([0-9]+)/?$") {
    rewrite ^/member/profile/([0-9]+)/?$ /index.php?p=member&action=profile&id=$1 last;
}

# Search results: /search/books
if ($uri ~ "^/search/([^/]+)/?$") {
    rewrite ^/search/([^/]+)/?$ /index.php?p=search&type=$1 last;
}
```

**Apache:**
```apache
# Member profile pages: /member/profile/123
RewriteRule ^member/profile/([0-9]+)/?$ index.php?p=member&action=profile&id=$1 [L,QSA]

# Search results: /search/books
RewriteRule ^search/([^/]+)/?$ index.php?p=search&type=$1 [L,QSA]
```

### SSL/HTTPS Configuration

For HTTPS sites, update your configuration:

**Nginx:**
```nginx
server {
    listen 443 ssl http2;
    ssl_certificate /path/to/certificate.crt;
    ssl_certificate_key /path/to/private.key;
    
    # Include all the rules above
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name domain.com www.domain.com;
    return 301 https://$server_name$request_uri;
}
```

## 🛡️ Security Considerations

1. **Input Validation:** URL parameters are automatically validated by SLiMS
2. **File Access:** Sensitive files are protected in configuration
3. **SQL Injection:** SLiMS handles database queries securely
4. **XSS Protection:** Security headers are included in configuration

## 📈 SEO Benefits

- **Clean URLs:** Better for search engine indexing
- **User-Friendly:** Easier to share and remember
- **Canonical URLs:** Prevents duplicate content issues
- **Better Analytics:** Cleaner tracking in web analytics tools

## 🔄 Migration from Standard URLs

1. **Gradual Migration:** Both URL formats work simultaneously
2. **301 Redirects:** Implement if needed for SEO
3. **Update Internal Links:** Gradually update internal links to use pretty URLs
4. **Sitemap Updates:** Update your sitemap.xml if you have one

## 📞 Support

This pretty URL implementation is designed to be:
- ✅ Zero-modification to SLiMS core
- ✅ Backward compatible
- ✅ Easy to reverse if needed
- ✅ Performance optimized

For issues specific to your SLiMS installation, consult the SLiMS community forums or documentation.


