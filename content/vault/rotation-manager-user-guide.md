# Vault Rotation Manager User Guide

## Overview

The Rotation Manager is Vault's centralized credential rotation system that automatically rotates credentials on a scheduled basis. It provides a robust, fault-tolerant mechanism for managing the lifecycle of static credentials across various secret engines and authentication methods.

**Note:** The Rotation Manager is an Enterprise feature and is not available in Vault Community Edition.

## Key Features

- **Automated Credential Rotation**: Schedule automatic rotation of credentials using CRON-style schedules or simple time-based periods
- **Flexible Scheduling**: Support for both CRON expressions and TTL-based rotation periods
- **Rotation Windows**: Define time windows during which rotations are allowed to occur
- **Retry Policies**: Configurable retry behavior with exponential backoff for failed rotations
- **High Availability**: Works seamlessly in clustered environments with proper failover
- **Namespace Support**: Full support for Vault Enterprise namespaces
- **Orphan Management**: Tracks and manages credentials that fail to rotate after exhausting retries

## How It Works

The Rotation Manager operates as a background service within Vault that:

1. **Maintains a Priority Queue**: Credentials are queued based on their next rotation time
2. **Checks the Queue Periodically**: Every 5 seconds, the manager checks for credentials due for rotation
3. **Dispatches Rotation Jobs**: When a credential is due, a rotation request is sent to the appropriate plugin
4. **Handles Failures**: Failed rotations are retried with exponential backoff according to the configured policy
5. **Persists State**: All rotation state is persisted to storage for recovery after restarts or failovers

## Supported Plugins

The Rotation Manager works with plugins that implement the `RotateCredential` callback. Currently supported:

- **Database Secrets Engine**: Root credential rotation
- **AWS Secrets Engine**: Root credential rotation
- **AWS Auth Method**: Client credential rotation
- **LDAP Auth Method**: Bind credential rotation

## Configuration Parameters

### Rotation Timing

You must specify **either** `rotation_schedule` **or** `rotation_period`, but not both.

#### rotation_schedule

A CRON-style expression defining when rotations should occur.

**Format**: Standard CRON format with 5 or 6 fields
- `* * * * *` (minute, hour, day of month, month, day of week)
- `* * * * * *` (second, minute, hour, day of month, month, day of week)

**Examples**:
```
"0 0 * * *"        # Daily at midnight
"0 2 * * 0"        # Weekly on Sunday at 2 AM
"0 0 1 * *"        # Monthly on the 1st at midnight
"*/30 * * * *"     # Every 30 minutes
"0 0,12 * * *"     # Twice daily at midnight and noon
```

#### rotation_period

A simple duration-based rotation interval (in seconds).

**Examples**:
```
86400              # Rotate every 24 hours
3600               # Rotate every hour
604800             # Rotate every week
```

#### rotation_window

Optional. Specifies the time window (in seconds) after the scheduled time during which rotation is allowed to occur. Only applicable when using `rotation_schedule`.

**Example**:
```
rotation_schedule = "0 2 * * *"
rotation_window = 3600
```
This allows rotation to occur between 2:00 AM and 3:00 AM.

**Default**: If not specified, rotations can occur at any time after the scheduled time.

### Rotation Policies

#### rotation_policy

Optional. The name of a rotation policy that defines retry behavior. If not specified, the default policy is used.

**Default Policy Values**:
- `max_retries_per_cycle`: 6 retries per cycle
- `max_retry_cycles`: 3 cycles
- Total maximum retries: 18 (6 × 3)

#### disable_automated_rotation

Set to `true` to disable automated rotation and deregister the credential from the Rotation Manager.

**Default**: `false`

## Usage Examples

### Example 1: Database Root Credential Rotation with Schedule

Configure a database connection with daily rotation at 2 AM:

```bash
vault write database/config/my-database \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/mydb" \
    allowed_roles="my-role" \
    username="vault_admin" \
    password="initial_password" \
    rotation_schedule="0 2 * * *" \
    rotation_window=3600
```

### Example 2: AWS Root Credential Rotation with Period

Configure AWS root credentials to rotate every 7 days:

```bash
vault write aws/config/root \
    access_key=AKIAIOSFODNN7EXAMPLE \
    secret_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
    region=us-east-1 \
    rotation_period=604800
```

### Example 3: Using a Custom Rotation Policy

First, create a custom rotation policy (requires appropriate permissions):

```bash
vault write sys/rotation-policy/aggressive \
    max_retries_per_cycle=10 \
    max_retry_cycles=5
```

Then reference it in your configuration:

```bash
vault write database/config/critical-db \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/criticaldb" \
    allowed_roles="critical-role" \
    username="vault_admin" \
    password="initial_password" \
    rotation_schedule="0 */6 * * *" \
    rotation_policy="aggressive"
```

### Example 4: Disabling Automated Rotation

To disable rotation for a previously configured credential:

```bash
vault write database/config/my-database \
    disable_automated_rotation=true
```

Or simply remove the rotation parameters:

```bash
vault write database/config/my-database \
    rotation_schedule="" \
    rotation_period=0
```

## Checking Rotation Status

### Get Next Rotation Time

To check when a credential is scheduled for rotation:

```bash
vault read database/config/my-database
```

The response includes:
- `next_vault_rotation`: Timestamp of the next scheduled rotation
- `last_vault_rotation`: Timestamp of the last successful rotation
- `rotation_id`: Unique identifier for this rotation job

**Example Response**:
```json
{
  "data": {
    "connection_url": "postgresql://{{username}}:{{password}}@localhost:5432/mydb",
    "next_vault_rotation": "2026-03-26T02:00:00Z",
    "last_vault_rotation": "2026-03-25T02:00:00Z",
    "rotation_id": "database/config/my-database",
    "rotation_schedule": "0 2 * * *",
    "rotation_window": 3600
  }
}
```

### Check for Orphaned Credentials

Orphaned credentials are those that have failed rotation after exhausting all retry attempts. To list orphaned credentials:

```bash
vault read sys/rotation-manager/orphans
```

**Note**: This endpoint requires appropriate permissions and is typically restricted to administrators.

## Retry Behavior

When a rotation fails, the Rotation Manager implements an intelligent retry strategy:

### Retry Cycles

1. **Per-Cycle Retries**: The system attempts rotation up to `max_retries_per_cycle` times with exponential backoff
2. **Backoff Duration**: Starts at 10 seconds, increases exponentially up to 5 minutes
3. **Cycle Reset**: After exhausting per-cycle retries, the system waits until the next scheduled rotation time
4. **Multiple Cycles**: The system can retry across multiple cycles up to `max_retry_cycles`

### Backoff Calculation

- **Minimum Backoff**: 10 seconds
- **Maximum Backoff**: 5 minutes
- **Growth**: Exponential with randomization to prevent thundering herd

### Example Retry Timeline

With default policy (`max_retries_per_cycle=6`, `max_retry_cycles=3`):

**Cycle 1** (scheduled for 2:00 AM):
- Attempt 1: 2:00:00 AM (fails)
- Attempt 2: 2:00:10 AM (fails, 10s backoff)
- Attempt 3: 2:00:30 AM (fails, 20s backoff)
- Attempt 4: 2:01:10 AM (fails, 40s backoff)
- Attempt 5: 2:02:30 AM (fails, 80s backoff)
- Attempt 6: 2:05:10 AM (fails, 160s backoff)
- Attempt 7: 2:10:10 AM (fails, 300s backoff)

**Cycle 2** (next scheduled time, e.g., 2:00 AM next day):
- Retry counter resets, attempts 1-7 again

**Cycle 3** (next scheduled time):
- Final retry cycle, attempts 1-7 again

**After Cycle 3**: Credential is orphaned and requires manual intervention

## Orphaned Credentials

When a credential exhausts all retry attempts across all cycles, it becomes "orphaned":

### What Happens

1. The credential is moved to the orphan storage view
2. The rotation job is deregistered from the active queue
3. Administrators are notified via logs
4. The credential remains in the orphaned state until manually resolved

### Resolving Orphaned Credentials

1. **Investigate the Root Cause**: Check Vault logs for rotation failure reasons
2. **Fix the Underlying Issue**: Resolve connectivity, permission, or configuration problems
3. **Re-register the Credential**: Update the configuration to re-enable rotation

```bash
# Fix the issue, then re-register
vault write database/config/my-database \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/mydb" \
    allowed_roles="my-role" \
    username="vault_admin" \
    password="new_password" \
    rotation_schedule="0 2 * * *"
```

## High Availability Considerations

### Cluster Behavior

- **Active Node Only**: The Rotation Manager runs only on the active node of a cluster
- **Automatic Failover**: When leadership changes, the new active node restores rotation state from storage
- **Replication**: 
  - **Shared Mounts**: Rotations occur on the Primary cluster's active node
  - **Local Mounts**: Rotations occur on each cluster's active node independently

### Best Practices

1. **Monitor Active Node**: Ensure the active node is healthy and can reach backend systems
2. **Test Failover**: Verify rotation continues after leadership changes
3. **Network Connectivity**: Ensure all cluster nodes can reach credential backends
4. **Storage Performance**: Adequate storage performance is critical for queue operations

## Troubleshooting

### Common Issues

#### Rotation Not Occurring

**Symptoms**: Credentials are not rotating at the scheduled time

**Possible Causes**:
1. Rotation Manager not running (check if node is active)
2. Credential not registered (verify configuration includes rotation parameters)
3. Outside rotation window (check `rotation_window` setting)
4. Backend connectivity issues

**Resolution**:
```bash
# Check rotation status
vault read database/config/my-database

# Verify Vault is active
vault status

# Check logs for errors
journalctl -u vault -f
```

#### Repeated Rotation Failures

**Symptoms**: Logs show continuous rotation failures

**Possible Causes**:
1. Invalid credentials
2. Network connectivity issues
3. Backend service unavailable
4. Insufficient permissions

**Resolution**:
1. Test backend connectivity manually
2. Verify credential permissions
3. Check backend service health
4. Review rotation policy settings

#### Credentials Becoming Orphaned

**Symptoms**: Credentials appear in orphan list

**Possible Causes**:
1. Persistent backend failures
2. Configuration errors
3. Network issues
4. Insufficient retry policy

**Resolution**:
1. List orphaned credentials: `vault read sys/rotation-manager/orphans`
2. Investigate root cause in logs
3. Fix underlying issue
4. Re-register credential with corrected configuration

### Log Messages

Key log messages to monitor:

```
# Successful rotation
[INFO] vault.rotation-job-manager: successfully rotated job: rotationID=database/config/my-database

# Rotation failure
[ERROR] vault.rotation-job-manager: rotation failed, attempting to re-queue: rotationID=database/config/my-database error="connection refused"

# Orphaning
[ERROR] vault.rotation-job-manager: orphaning item, please re-register after resolving the error: rotationID=database/config/my-database

# Deregistration
[DEBUG] vault.rotation-job-manager: deregistering rotation job; mount has been disabled: mount=database
```

## Security Considerations

### Credential Exposure

- Rotations occur automatically without user intervention
- New credentials are generated by the backend system
- Old credentials are invalidated immediately after successful rotation
- Failed rotations do not invalidate existing credentials

### Audit Logging

All rotation operations are logged in Vault's audit log:
- Rotation attempts
- Successes and failures
- Configuration changes
- Deregistration events

### Permissions

Rotation operations require:
- Plugin-specific permissions for the rotation endpoint
- System-level permissions for rotation policy management (administrators only)

## Migration from Legacy Rotation

If migrating from a plugin's built-in rotation mechanism:

1. **Disable Legacy Rotation**: Follow plugin-specific instructions
2. **Configure Rotation Manager**: Set `rotation_schedule` or `rotation_period`
3. **Verify Migration**: Check `next_vault_rotation` matches expected schedule
4. **Monitor First Rotation**: Ensure successful transition

The Rotation Manager will honor the legacy rotation time for the first cycle, then follow the new schedule.

## API Reference

### Configuration Endpoints

Each plugin that supports rotation exposes configuration endpoints. Common pattern:

```
POST /auth/<mount>/config/root
POST /<mount>/config/root
POST /database/config/<name>
```

### System Endpoints

```
GET /sys/rotation-manager/orphans
```
Lists all orphaned rotation entries (requires admin permissions).

## Limitations

1. **Enterprise Only**: Not available in Vault Community Edition
2. **Plugin Support Required**: Only works with plugins implementing `RotateCredential`
3. **Active Node Only**: Rotations only occur on the active cluster node
4. **Schedule Precision**: Queue checked every 5 seconds (rotations may occur up to 5 seconds after scheduled time)
5. **No Manual Trigger**: Cannot manually trigger a scheduled rotation (use plugin-specific manual rotation endpoints instead)

## Best Practices

1. **Use Rotation Windows**: For schedule-based rotations, define appropriate windows to handle timing variations
2. **Monitor Orphans**: Regularly check for orphaned credentials and resolve issues promptly
3. **Test Configurations**: Test rotation in non-production environments first
4. **Set Appropriate Policies**: Balance retry aggressiveness with system load
5. **Plan Maintenance Windows**: Consider rotation schedules when planning maintenance
6. **Document Custom Policies**: Maintain documentation for any custom rotation policies
7. **Monitor Logs**: Set up alerting for rotation failures
8. **Regular Audits**: Periodically review rotation configurations and schedules

## Support

For issues or questions:
- Check Vault logs for detailed error messages
- Review plugin-specific documentation
- Consult HashiCorp support for Enterprise customers
- Report bugs through official HashiCorp channels

## Version History

- **Vault 1.16+**: Rotation Manager introduced as Enterprise feature
- **Vault 1.17+**: Enhanced retry policies and orphan management