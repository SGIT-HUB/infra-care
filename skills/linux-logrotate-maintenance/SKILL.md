---
name: linux-logrotate-maintenance
description: Use when configuring or troubleshooting log rotation for services on Linux (OpenResty, Redis, custom apps). Covers logrotate config templates, postrotate signals, and cron environment pitfalls.
platforms: [linux]
---
# Linux Logrotate Maintenance

Configure and verify log rotation for services that lack built-in log management, ensuring logs are compressed, retained for a defined period, and safely reopened after rotation.

## When to Use
- Service writes to a single log file with no built-in rotation.
- Existing rotation uses wrong signal (e.g., SIGHUP instead of USR1) causing log loss.
- Postrotate commands fail silently under cron due to missing environment variables.

## Configuration Template

### OpenResty / Nginx
```
/usr/local/openresty/nginx/logs/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        [ -f /usr/local/openresty/nginx/logs/nginx.pid ] && kill -USR1 $(cat /usr/local/openresty/nginx/logs/nginx.pid)
    endscript
}
```

### Redis
```
/var/log/redis/redis.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    create 0640 redis redis
    postrotate
        redis-cli -a "$REDIS_PASSWORD" CONFIG SET logfile /var/log/redis/redis.log > /dev/null 2>&1 || true
    endscript
}
```

> **Note:** Replace `$REDIS_PASSWORD` with the actual authentication method appropriate for your environment. See "Cron Environment Pitfalls" below.

## Verification
Always dry-run before deploying:
```bash
logrotate -d /etc/logrotate.d/<config-file>
```
Check for syntax errors and confirm the correct log files are matched.

Force a test rotation (optional):
```bash
logrotate -f /etc/logrotate.d/<config-file>
```

## Cron Environment Pitfalls

Logrotate is typically executed by `/etc/cron.daily/logrotate` as root. The cron environment is **minimal** and does NOT inherit user shell environment variables. Common failures:

| Symptom | Cause | Fix |
|---|---|---|
| `redis-cli CONFIG SET` fails with NOAUTH | `REDISCLI_AUTH` env var not available in cron | Use `-a 'password'` flag explicitly, or read from a restricted file |
| Postrotate script exits non-zero, rotation aborts | Command not found in cron PATH | Use absolute paths (e.g., `/usr/bin/redis-cli`) |
| Log file not reopened after rotation | Wrong signal or PID file path | Verify signal (USR1 for nginx/openresty) and PID file location |

### Recommended Pattern for Authenticated Services
Store credentials in a root-only readable file rather than relying on environment variables:
```bash
# Create restricted credential file
echo "Redis@password" > /root/.rediscli_auth
chmod 600 /root/.rediscli_auth

# In logrotate postrotate
postrotate
    /usr/bin/redis-cli -a "$(cat /root/.rediscli_auth)" CONFIG SET logfile /var/log/redis/redis.log > /dev/null 2>&1 || true
endscript
```

## Retention Policy Guidelines
| Scenario | Recommended Retention |
|---|---|
| Development / debugging | 7–30 days |
| Production application logs | 30–90 days |
| Audit / compliance logs | Per regulatory requirement |
| Access logs with high volume | 7–14 days with compression |

Use `compress` + `delaycompress` to avoid compressing the most recent rotated file (useful if processes still write to it briefly after rotation).
