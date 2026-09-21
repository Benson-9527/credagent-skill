# CredAgent Troubleshooting Guide

## Permission Denied Errors

When a legitimate application receives "Permission denied" errors when accessing protected credential files, use the following diagnostic steps:

### Check CredAgent Logs

#### Linux (systemd)

View recent warning and error logs from the CredAgent service. Note that systemd treats all program output as info level, so use grep to filter WARNING/ERROR messages:

```bash
# View last 30 log entries and filter for warnings/errors
journalctl -u credagent -n 30 | grep -E "\[WARNING\]|\[ERROR\]"

# View real-time logs for active debugging
journalctl -u credagent -f

# Find denied access for a specific protected file (recommended)
journalctl -u credagent -n 50 | grep "denied\]" | grep "Object:a23457c0659d"
```

**Note (v0.2.0+)**: Only credagent service exists — no credmon.

#### macOS (launchd)

On macOS, CredAgent writes logs to files instead of systemd journal:

```bash
# View last 30 lines of credagent log
tail -n 30 /usr/local/var/log/credagent/credagent.log

# View real-time logs for active debugging
tail -f /usr/local/var/log/credagent/credagent.log

# Find denied access for a specific protected file (recommended)
grep "denied\]" /usr/local/var/log/credagent/credagent.log | grep "Object:a23457c0659d" | tail -n 10

# Search with case-insensitive match
grep -i "denied" /usr/local/var/log/credagent/credagent.log | tail -n 20
```

**macOS Log File Locations:**
- CredAgent daemon: `/usr/local/var/log/credagent/credagent.log`
- Error output: `/usr/local/var/log/credagent/credagent.err`

**Note (v0.2.0+)**: No credmon log files — FUSE is integrated.

### Common Permission Denied Causes

1. **Binary Integrity Mismatch**
   - The allowed binary has been modified since protection was applied
   - Check if the program was updated, patched, or corrupted
   - Solution: Re-protect the credential with the current binary version

2. **Wrong Binary Path**
   - The application is being executed from a different path than specified during protection
   - Example: Protected `/usr/bin/ssh` but application uses `/bin/ssh`
   - Solution: Verify the exact binary path with `which <command>` and re-protect if needed

3. **Wrapper Script Issues**
   - For interpreted languages (Python, Bash), ensure you protected the actual script file, not just the interpreter
   - Example: Protect `script.py` not `/usr/bin/python3`
   - Solution: Use the full path to your script as the `--allowed-binary`

4. **Process Context Changes**
   - Some applications spawn child processes that access credentials
   - The child process may have different characteristics than the parent
   - Solution: Identify the actual process accessing the file using logs

5. **FUSE Mount Issues**
   - The CredAgent FUSE filesystem may not be properly mounted
   - Check mount status: `mount | grep credagent`
   - Solution: Restart credagent service if mount is missing

### Diagnostic Commands

#### Linux
```bash
# Verify FUSE mount point
mount | grep credagent

# Check service status (v0.2.0+: only credagent)
systemctl status credagent

# List protected files to verify configuration
credctl list

# Check if the protected file symlink exists
ls -la /path/to/original/credential/file

# Verify the target file in FUSE mount
ls -la /mnt/credagent/shielded/<file-id>
```

#### macOS
```bash
# Verify FUSE mount point
mount | grep credagent
# or
mount | grep osxfuse

# Check service status
sudo launchctl list | grep credagent

# List protected files to verify configuration
credctl list

# Check if the protected file symlink exists
ls -la /path/to/original/credential/file

# Verify the target file in FUSE mount
ls -la /usr/local/var/credagent/shielded/<file-id>

# Check socket files
ls -la /var/run/credagent/
```

### Log Analysis

When examining CredAgent logs, look for these key indicators:

- **"Binary hash mismatch"** - Indicates the allowed binary has changed
- **"Unauthorized process"** - Process doesn't match allowed binary path
- **"Access count exceeded"** - Max open count limit reached
- **"FUSE operation failed"** - Underlying filesystem issues

### Understanding CredAgent Log Format

CredAgent logs contain detailed information about access attempts. Here's how to interpret them:

**Example log entries:**
```
Apr 14 17:42:25 credagent[111915]: [WARNING] [engine.go:318] [Alarm not denied] Object:a23457c0659d, Program: cat /home/user/.aliyun/config.json, Reason: Process executable /usr/bin/cat not in allowed list: [/usr/local/bin/aliyun]
Apr 14 17:42:25 credagent[111915]: [WARNING] [engine.go:331] [denied] Object:a23457c0659d, Program: cat /home/user/.aliyun/config.json, Reason: Binary checksum mismatch. Expected: [a5f9e35...], Actual: 90c9437a...
```

**Log field explanations:**

| Field | Description |
|-------|-------------|
| `[Alarm not denied]` | Warning indicator, but NOT the reason for denial (informational) |
| `[denied]` | **This is the actual denial reason** - the access was blocked for this reason |
| `Object:a23457c0659d` | Protected credential file ID (matches the FUSE file name) |
| `Program: cat ...` | The command/program that attempted access |
| `Reason: ...` | Detailed explanation of why access was denied |

**Finding the Object ID:**

To find the Object ID for a protected file, check the symlink:
```bash
$ ls -l ~/.aliyun/config.json
lrwxrwxrwx 1 user user 36 Apr 14 16:47 /home/user/.aliyun/config.json -> /mnt/credagent/shielded/a23457c0659d
```
> `a23457c0659d` is the Object ID
                                                                                           
**Recommended diagnostic command:**
```bash
# Replace a23457c0659d with your actual Object ID
journalctl -u credagent -n 50 | grep "denied\]" | grep "Object:a23457c0659d"
```

This will show you the exact reason why access was denied for that specific protected file.

### Admin Password Issues

If you used `--auto-init` during installation, CredAgent generates a random, unrecoverable admin password. Management commands such as `release`, `remove`, `update`, `attach`, `detach`, and `refresh` require the admin password and will fail with `Incorrect password` until you reset it.

**Symptoms:**
```
Error: failed to release credential: rpc error: code = PermissionDenied desc = Incorrect password
```

**Solution:**

1. Reset the admin password. Choose **N** when asked whether to overwrite the master key, then choose **Y** to set a new admin password:
   ```bash
   sudo /usr/local/bin/credagent init --config /usr/local/etc/credagent/engine/config.json
   ```

2. Restart the service so the running daemon loads the new password hash.

   **Linux:**
   ```bash
   sudo systemctl restart credagent
   ```

   **macOS:**
   ```bash
   sudo launchctl unload /Library/LaunchDaemons/com.credagent.enclave.plist
   sudo launchctl load /Library/LaunchDaemons/com.credagent.enclave.plist
   ```

3. Retry the management command and enter the new password.

### Recovery Steps

1. **Temporary Access**: If urgent access is needed, release the credential temporarily:
   ```bash
   rm /path/to/original/file
   credctl release /path/to/original/file
   ```

2. **Re-protect with Correct Settings**: After identifying the issue, re-protect with corrected parameters

3. **Service Restart**: If services appear unresponsive:

   **Linux (systemd):**
   ```bash
   sudo systemctl restart credagent
   ```

   **macOS (launchd):**
   ```bash
   # Unload service
   sudo launchctl unload /Library/LaunchDaemons/com.credagent.enclave.plist
   
   # Load service
   sudo launchctl load /Library/LaunchDaemons/com.credagent.enclave.plist
   ```

Remember: Permission denied errors are a security feature, not a bug. They indicate that CredAgent is working correctly to prevent unauthorized access.