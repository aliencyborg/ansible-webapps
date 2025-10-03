# cPanel WordPress Backup Restore Guide

This guide explains how to restore your cPanel WordPress backup into Docker containers using Ansible.

## Overview

Your cPanel backup contains:

- **WordPress Files**: Located in `homedir/public_html/`
- **Database**: Located in `mysql.sql` and `mysql/overvie2_overview1.sql`

**IMPORTANT**: Many cPanel backups contain a "broken" mix of modern WordPress files (6.x) with legacy PHP 5.x plugin configurations. The restoration process now automatically handles these compatibility issues.

## Prerequisites

1. Your WordPress containers should already be deployed using your `ovc-wp.yml` task
2. The backup should be extracted to `~/Downloads/backup-6.13.2025_07-04-38_overvie2`
3. Your `play_vars/ovc-wp.yml` file should contain the correct database credentials

## Quick Start

### Option 1: Modern Backup Restore (Recommended)

For most cPanel backups, including those with mixed WordPress versions, use the **legacy restore method** which automatically handles compatibility issues:

```bash
# This method uses WordPress 6.1 with PHP 8.0 - compatible with both old and new WordPress files
ansible-playbook playbooks/restore-ovc-cpanel-legacy.yml

# Test your site, then optionally upgrade to latest WordPress:
ansible-playbook playbooks/upgrade-ovc-php.yml
```

**What this method automatically fixes:**

- ✅ **Database connection settings** - Updates wp-config.php for Docker containers
- ✅ **Plugin compatibility** - Disables plugins with PHP 8.0 type errors
- ✅ **Cache file issues** - Removes advanced-cache.php with old cPanel paths
- ✅ **PHP version compatibility** - Uses WordPress 6.1 + PHP 8.0 bridge version

### Option 2: Legacy PHP Method (Only for very old backups)

Only use this if your backup contains WordPress 4.x or older:

```bash
# Edit tasks/ovc-wp-legacy.yml to use wordpress:4.9-php5.6-apache
# Then run:
ansible-playbook playbooks/restore-ovc-cpanel-legacy.yml
```

## Common "Broken Backup" Issues Automatically Fixed

### Issue 1: PHP Parse Errors

**Problem**: Backup contains WordPress 6.x files with modern PHP syntax but old plugins
**Solution**: WordPress 6.1 + PHP 8.0 bridge handles most compatibility issues
**Auto-fix**: Legacy plugins with type errors are automatically disabled

### Issue 2: Database Connection Errors  

**Problem**: wp-config.php contains old cPanel database settings
**Solution**: Automatically updates database credentials for Docker containers:

```php
// Old cPanel settings (automatically replaced):
define('DB_HOST', 'localhost');
define('DB_NAME', 'cpanel_username_dbname');
define('DB_USER', 'cpanel_username_user');

// New Docker settings (automatically applied):
define('DB_HOST', 'mysql');
define('DB_NAME', 'ovcwpdb');
define('DB_USER', 'ovcwpadmin');
```

### Issue 3: Cache File Path Errors

**Problem**: advanced-cache.php references old cPanel paths like `/home3/username/public_html/`
**Solution**: Automatically removes problematic cache files during restoration

### Issue 4: Plugin Compatibility Errors

**Problem**: Plugins with PHP 8.0 type checking issues (e.g., `round('6.8.1')`)
**Solution**: Automatically disables known problematic plugins:

- `force-strong-passwords` (PHP type errors)
- `wp-super-cache` (path issues)

## Bridging Strategy Explained

The **WordPress 6.1 + PHP 8.0** combination works as a "bridge" because:

1. **WordPress 6.1**: Close enough to 6.8.x to handle modern core files
2. **PHP 8.0**: Supports modern syntax but with better legacy plugin tolerance
3. **Automatic fixes**: Handles database, cache, and plugin compatibility issues

This approach works for:

- ✅ WordPress 4.x backups (legacy compatibility)
- ✅ WordPress 5.x backups (direct compatibility)  
- ✅ WordPress 6.x backups (modern compatibility)
- ✅ "Broken" mixed-version backups (automatic fixes)

## Upgrading to Modern WordPress + PHP (After Restoration)

### Overview

After successful restoration with the bridge version (WordPress 6.1 + PHP 8.0), you may want to upgrade to the latest versions:

- **Target**: WordPress 6.8 + PHP 8.2
- **Benefits**: Latest security patches, performance improvements, modern features
- **Support**: WordPress 6.1 has [beta support for PHP 8.2](https://make.wordpress.org/core/handbook/references/php-compatibility-and-wordpress-versions/)

### Step 1: Pre-Upgrade Analysis

Run the comprehensive compatibility check:

```bash
# This analyzes your current setup for PHP 8.2 and WordPress 6.8 compatibility
ansible-playbook playbooks/upgrade-ovc-php.yml --limit beast
```

**What this checks:**

- ✅ **PHP 8.2 Compatibility**: Scans for deprecated functions, dynamic properties, type issues
- ✅ **Plugin Analysis**: Identifies plugins with known PHP 8.2 issues
- ✅ **Theme Compatibility**: Checks for legacy PHP patterns
- ✅ **WordPress Core**: Verifies update path and database integrity
- ✅ **Site Functionality**: Tests current performance as baseline

### Step 2: Fix Compatibility Issues

Based on the analysis, address any issues found:

#### Common PHP 8.2 Issues to Fix

**1. Dynamic Properties (will cause errors in PHP 8.2)**

```php
// Old (problematic):
class MyClass {
    // Dynamic property access without declaration
}
$obj = new MyClass();
$obj->dynamic_property = 'value'; // Will cause error in PHP 8.2

// Fixed:
class MyClass {
    public $dynamic_property; // Explicitly declare properties
}
```

**2. Deprecated Functions**

```php
// Replace deprecated functions:
utf8_encode() → mb_convert_encoding()
utf8_decode() → mb_convert_encoding()
```

**3. Plugin Updates**

- Update plugins before upgrading PHP
- Disable plugins that aren't PHP 8.2 compatible
- Find modern alternatives for outdated plugins

### Step 3: Execute the Upgrade

Once compatibility issues are resolved:

```bash
# Execute the upgrade (creates automatic backups)
ansible-playbook playbooks/upgrade-ovc-php-phase2.yml --limit beast
```

**What this does:**

- ✅ **Creates comprehensive backups** (database + container snapshots)
- ✅ **Deploys WordPress 6.8 + PHP 8.2** container
- ✅ **Tests compatibility** before switching
- ✅ **Runs database upgrades** if needed
- ✅ **Validates site functionality**
- ✅ **Switches to new version** if tests pass
- ✅ **Automatic rollback** if issues occur

### Step 4: Post-Upgrade Monitoring

After successful upgrade:

1. **Monitor for 24-48 hours**
2. **Test all critical functionality**
3. **Check WordPress admin for plugin warnings**
4. **Update any remaining plugins**
5. **Performance testing**

### Alternative: Incremental Upgrade Path

If the direct upgrade to WordPress 6.8 + PHP 8.2 has issues, consider incremental steps:

```bash
# Option 1: Upgrade WordPress first, then PHP
# Edit playbooks/upgrade-ovc-php-phase2.yml target_wp_image: "wordpress:6.3"
ansible-playbook playbooks/upgrade-ovc-php-phase2.yml --limit beast

# Test, then continue to 6.5, 6.8, etc.

# Option 2: Upgrade PHP first, then WordPress  
# Edit tasks/ovc-wp-legacy.yml to use wordpress:6.1 (keeps same WP version)
# with PHP 8.1 instead of 8.0
```

### Rollback if Needed

If the upgrade causes issues:

```bash
# Find your backup timestamp from the upgrade output
ansible-playbook playbooks/rollback-upgrade.yml --limit beast -e backup_timestamp=TIMESTAMP
```

### Upgrade Benefits

**WordPress 6.8 Benefits:**

- Latest security patches
- Performance improvements
- Block editor enhancements
- PHP 8.2 full compatibility

**PHP 8.2 Benefits:**

- 10-15% performance improvement over PHP 8.0
- Better security features
- Modern syntax and type checking
- Active support until December 2025

### Troubleshooting Upgrade Issues

**Common Issues:**

1. **Plugin Compatibility**
   - Solution: Update plugins before upgrading
   - Alternative: Find modern plugin replacements
   - Temporary: Disable problematic plugins

2. **Theme Issues**
   - Solution: Update theme or switch to modern theme
   - Check: Custom theme code for PHP 8.2 compatibility

3. **Performance Degradation**
   - Check: Plugin conflicts with PHP 8.2
   - Solution: Enable WordPress caching
   - Monitor: Database performance after upgrade

**Emergency Contacts:**

- WordPress VIP PHP 8 Guide: <https://wpvip.com/how-to-prepare-your-wordpress-site-for-php-8/>
- WordPress PHP Compatibility: <https://make.wordpress.org/core/handbook/references/php-compatibility-and-wordpress-versions/>

## Manual Troubleshooting (If Needed)

### If Automatic Restoration Fails

If you need to troubleshoot or perform manual steps:

#### 1. Check Container Status

```bash
# On your remote server
ssh -p 43722 -i ~/.ssh/id_ed25519 hank@67.220.24.251 "docker ps -a | grep ovc"
```

#### 2. Check WordPress Logs

```bash
ssh -p 43722 -i ~/.ssh/id_ed25519 hank@67.220.24.251 "docker logs ovc-wp-web | tail -20"
```

#### 3. Manually Fix Database Connection (if needed)

```bash
# Update wp-config.php manually
ssh -p 43722 -i ~/.ssh/id_ed25519 hank@67.220.24.251 '
docker exec ovc-wp-web sed -i "s/define(\"DB_HOST\", \"[^\"]*\")/define(\"DB_HOST\", \"mysql\")/g" /var/www/html/wp-config.php
'
```

#### 4. Manually Disable Problematic Plugins

```bash
# Disable plugins causing PHP errors
ssh -p 43722 -i ~/.ssh/id_ed25519 hank@67.220.24.251 '
docker exec ovc-wp-web mv /var/www/html/wp-content/mu-plugins/force-strong-passwords /var/www/html/wp-content/mu-plugins/force-strong-passwords.disabled 2>/dev/null || true
'
```

#### 5. Test Database Connection

```bash
# Check if database has tables
ssh -p 43722 -i ~/.ssh/id_ed25519 hank@67.220.24.251 "docker exec ovc-wp-db mysql -uovcwpadmin -psGo9gwLrTtxG ovcwpdb -e 'SHOW TABLES;' | wc -l"
```

### Common Error Messages

**"Parse error: syntax error, unexpected ':'"**

- ✅ Fixed automatically by using WordPress 6.1 + PHP 8.0

**"Error establishing a database connection"**

- ✅ Fixed automatically by updating wp-config.php database settings

**"There has been a critical error on this website"**

- ✅ Fixed automatically by disabling problematic plugins

**HTTP 301 redirect to old domain**

- ✅ This is normal - indicates successful restoration!
- Update URLs through WordPress admin or database queries

## Configuration Files

### Current Setup

- **WordPress Container**: `ovc-wp-web`
- **Database Container**: `ovc-wp-db`
- **Database Name**: `ovcwpdb`
- **Database User**: `ovcwpadmin`
- **Database Password**: `[your_password]`

### Files Created

- `tasks/restore-cpanel-backup.yml` - Main restore task
- `tasks/ovc-wp-legacy.yml` - Legacy PHP 5.6 deployment
- `playbooks/restore-ovc-cpanel.yml` - Modern restoration playbook
- `playbooks/restore-ovc-cpanel-legacy.yml` - Legacy restoration playbook
- `playbooks/upgrade-ovc-php.yml` - PHP upgrade preparation playbook

## Alternative: WordPress Admin UI Restore

If you prefer using WordPress plugins:

1. Install a backup/restore plugin like:
   - **UpdraftPlus**
   - **BackWPup**
   - **Duplicator**

2. Convert your cPanel backup to the plugin format
3. Upload and restore through the WordPress admin interface

However, the Ansible approach is more reliable for large sites and ensures proper file permissions.

## Next Steps

After restoration:

1. **Test your site**: Visit `https://ovc.aliencyb.org`
2. **Update any hardcoded URLs** in content
3. **Check and update plugins/themes**
4. **Set up regular backups** using your existing backup tasks
5. **Update any third-party integrations** that use the old domain

## Security Notes

- The database credentials are visible in this guide - consider changing them after restoration
- Review any security plugins and their configurations
- Update any API keys that reference the old domain
- Check SSL certificates and redirects
