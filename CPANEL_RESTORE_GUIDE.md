# cPanel WordPress Backup Restore Guide

This guide explains how to restore your cPanel WordPress backup into Docker containers using Ansible.

## Overview

Your cPanel backup contains:

- **WordPress Files**: Located in `homedir/public_html/`
- **Database**: Located in `mysql.sql` and `mysql/overvie2_overview1.sql`

## Prerequisites

1. Your WordPress containers should already be deployed using your `ovc-wp.yml` task
2. The backup should be extracted to `~/Downloads/backup-6.13.2025_07-04-38_overvie2`
3. Your `play_vars/ovc-wp.yml` file should contain the correct database credentials

## Quick Start

### Option 1: Use the Ansible Playbook (Recommended)

```bash
# Run the restore playbook
ansible-playbook playbooks/restore-ovc-cpanel.yml -i inventory.example

# Or if you need to specify the target host
ansible-playbook playbooks/restore-ovc-cpanel.yml -i inventory.example --limit your_host
```

### Option 2: Manual Steps

If you prefer to do it manually or need to troubleshoot:

#### 1. Copy WordPress Files

```bash
# Stop the WordPress container
docker stop ovc-wp-web

# Clear existing content
docker run --rm -v ovc-wp-content:/var/www/html alpine:latest sh -c "rm -rf /var/www/html/* /var/www/html/.*" || true

# Copy WordPress files
docker cp ~/Downloads/backup-6.13.2025_07-04-38_overvie2/homedir/public_html/. ovc-wp-web:/var/www/html/

# Fix permissions
docker run --rm -v ovc-wp-content:/var/www/html alpine:latest sh -c "chown -R 33:33 /var/www/html"

# Start the container
docker start ovc-wp-web
```

#### 2. Restore Database

Choose one of these database files (try them in order):

```bash
# Option 1: Main database file
cat ~/Downloads/backup-6.13.2025_07-04-38_overvie2/mysql.sql | \
  docker exec -i ovc-wp-db mysql -uovcwpadmin -psGo9gwLrTtxG ovcwpdb

# Option 2: Specific database file (if above fails)
cat ~/Downloads/backup-6.13.2025_07-04-38_overvie2/mysql/overvie2_overview1.sql | \
  docker exec -i ovc-wp-db mysql -uovcwpadmin -psGo9gwLrTtxG ovcwpdb
```

#### 3. Update WordPress URLs

```bash
# Update site URLs to match your new domain
docker exec ovc-wp-web wp search-replace "http://overvie2.siteground.biz" "https://ovc.aliencyb.org" --allow-root

# Flush rewrite rules
docker exec ovc-wp-web wp rewrite flush --allow-root

# Clear cache
docker exec ovc-wp-web wp cache flush --allow-root
```

#### 4. Fix File Permissions

```bash
docker exec ovc-wp-web find /var/www/html -type f -exec chmod 644 {} \;
docker exec ovc-wp-web find /var/www/html -type d -exec chmod 755 {} \;
docker exec ovc-wp-web chown -R www-data:www-data /var/www/html
```

## Troubleshooting

### Database Issues

If you get database errors:

1. **Check if database is empty:**

   ```bash
   docker exec ovc-wp-db mysql -uovcwpadmin -psGo9gwLrTtxG ovcwpdb -e "SHOW TABLES;"
   ```

2. **Drop and recreate database if needed:**

   ```bash
   docker exec ovc-wp-db mysql -uovcwpadmin -psGo9gwLrTtxG -e "DROP DATABASE ovcwpdb; CREATE DATABASE ovcwpdb;"
   ```

3. **Try disabling foreign key checks:**

   ```bash
   echo "SET FOREIGN_KEY_CHECKS=0;" > temp_restore.sql
   cat ~/Downloads/backup-6.13.2025_07-04-38_overvie2/mysql.sql >> temp_restore.sql
   echo "SET FOREIGN_KEY_CHECKS=1;" >> temp_restore.sql
   cat temp_restore.sql | docker exec -i ovc-wp-db mysql -uovcwpadmin -psGo9gwLrTtxG ovcwpdb
   rm temp_restore.sql
   ```

### WordPress Issues

1. **Check if WordPress files are properly copied:**

   ```bash
   docker exec ovc-wp-web ls -la /var/www/html/
   ```

2. **Check wp-config.php database settings:**

   ```bash
   docker exec ovc-wp-web cat /var/www/html/wp-config.php | grep DB_
   ```

3. **Update wp-config.php if needed:**

   ```bash
   docker exec ovc-wp-web wp config set DB_HOST mysql --allow-root
   docker exec ovc-wp-web wp config set DB_NAME ovcwpdb --allow-root
   docker exec ovc-wp-web wp config set DB_USER ovcwpadmin --allow-root
   docker exec ovc-wp-web wp config set DB_PASSWORD [your_password] --allow-root
   ```

### Container Issues

1. **Check container status:**

   ```bash
   docker ps -a | grep ovc-wp
   ```

2. **Check container logs:**

   ```bash
   docker logs ovc-wp-web
   docker logs ovc-wp-db
   ```

3. **Restart containers:**

   ```bash
   docker restart ovc-wp-db ovc-wp-web
   ```

## Configuration Files

### Current Setup

- **WordPress Container**: `ovc-wp-web`
- **Database Container**: `ovc-wp-db`
- **Database Name**: `ovcwpdb`
- **Database User**: `ovcwpadmin`
- **Database Password**: `[your_password]`

### Files Created

- `tasks/restore-cpanel-backup.yml` - Main restore task
- `playbooks/restore-ovc-cpanel.yml` - Restoration playbook

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
