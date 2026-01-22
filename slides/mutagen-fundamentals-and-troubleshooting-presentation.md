# **Mutagen Fundamentals and Troubleshooting**
### with

<img src="images/ddev-logo.svg" alt="DDEV Logo" class="ddev-logo">

---

## What is Mutagen?

Asynchronous file synchronization tool for DDEV

* Decouples in-container reads/writes from host machine
* Enabled by default on macOS and traditional Windows
* Provides dramatic performance improvements
* Maintains cached copy in Docker volume
* Syncs changes asynchronously

---

## The Problem Mutagen Solves

Traditional Docker bind-mounts are slow on macOS/Windows

* Every file access checked against host filesystem
* Docker's implementation not performant
* PHP projects with tens of thousands of files
* `php-fpm` opening many files simultaneously
* **Result**: Poor webserving performance

> "The first time I tried it I cried." — Developer feedback

---

## How Mutagen Works in DDEV

1. Mounts Docker volume onto `/var/www` in web container
2. Linux Mutagen daemon installed inside web container
3. Host-side Mutagen daemon started by DDEV
4. Two daemons keep each other synchronized

**Near-native filesystem speed on both sides**

---

## Lifecycle Events

* **`ddev start`**: Starts daemon, creates/resumes sync session
* **`ddev stop`**: Flushes sync, pauses session
* **`ddev composer`**: Triggers synchronous flush after completion
* **`ddev mutagen reset`**: Removes volume, recreates from host

---

## Upload Directories

DDEV excludes user-generated files from Mutagen using bind-mounts

* **Drupal**: `sites/default/files`
* **WordPress**: `wp-content/uploads`
* **TYPO3**: `fileadmin`, `uploads`

Override in `.ddev/config.yaml`:

```yaml
upload_dirs:
  - sites/default/files
  - ../node_modules
```

---

## Common Issue: Initial Sync Time

First-time sync: 5-60 seconds depending on project size

* Magento 2: ~48 seconds initially, ~12 seconds subsequent
* If sync >1 minute: likely syncing large files unnecessarily

**Solution**: Add large directories to `upload_dirs`

---

## Common Issue: `node_modules`

Frontend build tools create massive directories

**Problem**: Slows Mutagen sync significantly

**Solution**: Add to `upload_dirs`

```yaml
upload_dirs:
  - sites/default/files
  - ../node_modules
```

Then run `ddev restart`

---

## Common Issue: File Changes When Stopped

Changes made while DDEV stopped (git operations, etc.)

* Mutagen has no awareness of changes
* May restore old files from volume on restart

**Solution**: Run before restarting

```bash
ddev mutagen reset
```

---

## Common Issue: Simultaneous Changes

Same file changes on both sides while out of sync

**Rare but possible with:**
* Scripts running simultaneously
* Massive branch changes
* Large `npm install` or `yarn install` operations

---

## Best Practices

* Do Git operations on the host, not in container
* Use `ddev composer` for Composer operations
* Run `ddev mutagen sync` after major Git branch changes
* Run `ddev mutagen sync` after manual Composer operations in container

---

## Disk Space Considerations

Project code exists in two places:
* On your computer
* Inside Docker volume

**Watch for:**
* Volumes >5GB (warning)
* Volumes >10GB (critical)

```bash
ddev utility mutagen-diagnose --all
```

---

## New Diagnostic Tool

```bash
ddev utility mutagen-diagnose
```

**Analyzes:**
* Volume sizes (warns >5GB, critical >10GB)
* Upload directories configuration
* Sync session status
* Large directories being synced
* Ignore patterns validation

---

## Diagnostic Output Example

```
⚠ 3 node_modules directories exist but are not
  excluded from sync (can cause slow sync)
  → Add to .ddev/config.yaml:
    upload_dirs:
      - sites/default/files
      - web/themes/custom/mytheme/node_modules
      - web/themes/contrib/bootstrap/node_modules
      - app/node_modules
  → Then run: ddev restart
```

---

## Debugging Long Startup Times

Watch what Mutagen is syncing:

```bash
ddev mutagen reset  # Start from scratch
ddev start
```

In another terminal:

```bash
while true; do ddev mutagen st -l | grep "^Current"; sleep 1; done
```

Shows output like:

```
Current file: vendor/bin/large-binary (306 MB/5.2 GB)
Current file: vendor/bin/large-binary (687 MB/5.2 GB)
```

---

## Monitoring Commands

Watch real-time sync activity:

```bash
ddev mutagen monitor
```

Force explicit sync:

```bash
ddev mutagen sync
```

Check sync status:

```bash
ddev mutagen status
ddev mutagen status -l  # Detailed
```

---

## Troubleshooting Steps

1. **Verify project works without Mutagen**

```bash
ddev config --performance-mode=none && ddev restart
```

2. **Run diagnostics**

```bash
ddev utility mutagen-diagnose
```

---

## Troubleshooting Steps (cont.)

3. **Reset to clean `.ddev/mutagen/mutagen.yml`**

```bash
mv .ddev/mutagen/mutagen.yml .ddev/mutagen/mutagen.yml.bak
ddev restart
```

4. **Reset Mutagen volume**

```bash
ddev mutagen reset
ddev restart
```

---

## Troubleshooting Steps (cont.)

5. **Enable debug output**

```bash
DDEV_DEBUG=true ddev start
```

6. **View Mutagen logs**

```bash
ddev mutagen logs
```

7. **Restart Mutagen daemon**

```bash
ddev utility mutagen daemon stop
ddev utility mutagen daemon start
```

---

## Advanced: Excluding Directories

**Recommended**: Use `upload_dirs` in `.ddev/config.yaml`

```yaml
upload_dirs:
  - sites/default/files
  - ../node_modules
  - ../vendor/bin
```

**Advanced**: Edit `.ddev/mutagen/mutagen.yml`

```yaml
ignore:
  paths:
    - "/web/themes/custom/mytheme/node_modules"
```

Always run `ddev mutagen reset` after changing `mutagen.yml`

---

## Git Hooks for Automatic Sync

Add `.git/hooks/post-checkout`:

```bash
#!/usr/bin/env bash
ddev mutagen sync || true
```

Make it executable:

```bash
chmod +x .git/hooks/post-checkout
```

---

## When to Disable Mutagen

**Disable if:**
* On Linux or WSL2 (native performance already)
* Filesystem consistency more critical than webserving performance
* Troubleshooting other DDEV issues

**Disable per-project:**

```bash
ddev mutagen reset && ddev config --performance-mode=none && ddev restart
```

**Disable globally:**

```bash
ddev config global --performance-mode=none
```

---

## Key Takeaways

* Use `ddev utility mutagen-diagnose` as first debugging step
* Configure `upload_dirs` to exclude large directories
* Run `ddev mutagen reset` after file changes when DDEV stopped
* Do Git operations on host, not in container
* Monitor sync activity with `ddev mutagen monitor`

**Benefits far outweigh caveats for most projects**

---

## Resources

* [DDEV Performance Documentation](https://docs.ddev.com/en/stable/users/install/performance/)
* [Mutagen Documentation](https://mutagen.io/documentation/)
* Blog: "Mutagen in DDEV: Functionality, Issues, and Debugging"

---

## Questions?
