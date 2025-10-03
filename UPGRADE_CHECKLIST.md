# WordPress & PHP Upgrade Checklist

## 🎯 Current State

- **Current**: WordPress 6.1 + PHP 8.0 (bridge version)
- **Target**: WordPress 6.8 + PHP 8.2 (modern)
- **Compatibility**: WordPress 6.1 has [beta support for PHP 8.2](https://make.wordpress.org/core/handbook/references/php-compatibility-and-wordpress-versions/)

## ✅ Pre-Upgrade Checklist

### 1. Compatibility Analysis

```bash
ansible-playbook playbooks/upgrade-ovc-php.yml --limit beast
```

**Review Results For:**

- [ ] PHP 8.2 compatibility issues
- [ ] Plugin compatibility warnings
- [ ] Theme compatibility issues
- [ ] Site functionality baseline

### 2. Fix Issues Before Upgrading

**Common PHP 8.2 Issues:**

- [ ] **Dynamic Properties**: Declare all class properties explicitly
- [ ] **Deprecated Functions**: Replace `utf8_encode()`, `utf8_decode()`, etc.
- [ ] **Type Declarations**: Fix parameter type mismatches
- [ ] **Legacy Plugins**: Update or disable incompatible plugins

**Known Problematic Plugins:**

- [ ] WP Super Cache (check version)
- [ ] W3 Total Cache (update required)
- [ ] Old contact forms (replace with modern alternatives)
- [ ] Force Strong Passwords (already disabled in restoration)

### 3. Backup Verification

- [ ] Current site is fully functional
- [ ] Database contains all content
- [ ] File permissions are correct
- [ ] HTTPS redirects working

## 🚀 Upgrade Execution

### Run the Upgrade

```bash
ansible-playbook playbooks/upgrade-ovc-php-phase2.yml --limit beast
```

**What It Does:**

- [ ] Creates automatic backups (database + containers)
- [ ] Deploys WordPress 6.8 + PHP 8.2
- [ ] Tests compatibility before switching
- [ ] Validates site functionality
- [ ] Switches to new version if tests pass

### Monitor Progress

- [ ] Watch for error messages during upgrade
- [ ] Verify backup creation
- [ ] Check container deployment success
- [ ] Confirm site accessibility test

## 🧪 Post-Upgrade Validation

### Immediate Tests

- [ ] Site loads correctly (HTTP 200/301/302)
- [ ] WordPress admin accessible
- [ ] No PHP fatal errors in logs
- [ ] Database connection working

### Extended Testing (24-48 hours)

- [ ] All plugins functioning correctly
- [ ] Contact forms working
- [ ] E-commerce functionality (if applicable)
- [ ] User registration/login
- [ ] Search functionality
- [ ] Performance monitoring

## 🚨 Rollback Plan

If issues occur:

```bash
# Use backup timestamp from upgrade output
ansible-playbook playbooks/rollback-upgrade.yml --limit beast -e backup_timestamp=TIMESTAMP
```

**Rollback Triggers:**

- [ ] Site returns 500/502 errors
- [ ] Critical functionality broken
- [ ] Performance significantly degraded
- [ ] PHP fatal errors in logs

## 📊 Alternative Upgrade Paths

### Option 1: Incremental WordPress Upgrade

```bash
# Upgrade WordPress gradually: 6.1 → 6.3 → 6.5 → 6.8
# Edit target_wp_image in playbooks/upgrade-ovc-php-phase2.yml
```

### Option 2: PHP First, WordPress Second

```bash
# Keep WordPress 6.1, upgrade to PHP 8.1, then PHP 8.2, then WordPress
# Edit image in tasks/ovc-wp-legacy.yml
```

### Option 3: Staging Environment Test

```bash
# Clone production, test upgrade on staging first
# Requires additional container/server setup
```

## 🔧 Troubleshooting Common Issues

### Plugin Compatibility

- **Issue**: Plugin causes PHP 8.2 errors
- **Solution**: Update plugin or find alternative
- **Temporary**: Disable problematic plugin

### Theme Compatibility  

- **Issue**: Theme uses deprecated PHP functions
- **Solution**: Update theme or hire developer to fix
- **Alternative**: Switch to modern WordPress theme

### Performance Issues

- **Issue**: Site slower after upgrade
- **Check**: Plugin conflicts, database queries
- **Solution**: Enable caching, optimize database

### Database Issues

- **Issue**: Database connection errors
- **Check**: Container networking, credentials
- **Solution**: Restart containers, verify settings

## 📋 Success Criteria

Upgrade is successful when:

- [ ] Site responds correctly (HTTP 200/301)
- [ ] WordPress admin accessible
- [ ] No PHP errors in logs
- [ ] All critical functionality works
- [ ] Performance maintained or improved
- [ ] Backup strategy updated for new version

## 🎉 Completion Tasks

After successful upgrade:

- [ ] Update documentation with new versions
- [ ] Clean up old backup files (after 48-hour monitoring)
- [ ] Update monitoring scripts if needed
- [ ] Plan regular WordPress/plugin updates
- [ ] Document any custom fixes made during upgrade

## 📞 Emergency Contacts

- **WordPress VIP PHP 8 Guide**: <https://wpvip.com/how-to-prepare-your-wordpress-site-for-php-8/>
- **WordPress Core PHP Compatibility**: <https://make.wordpress.org/core/handbook/references/php-compatibility-and-wordpress-versions/>
- **PHP 8.2 Migration Guide**: <https://www.php.net/manual/en/migration82.php>
