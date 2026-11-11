---
name: openresty-stream-maintenance
description: Use when debugging frequent disconnections through OpenResty stream proxy for Redis, PostgreSQL, or other TCP long-connection services. Covers proxy_timeout tuning and root cause analysis.
platforms: [linux]
---
# OpenResty Stream Proxy Timeout Maintenance

Diagnose and resolve frequent disconnections when using OpenResty as a TCP/UDP stream proxy for backend services such as Redis or PostgreSQL.

## Symptom
- Client connects through OpenResty stream proxy to Redis/PG successfully at first.
- Connection drops after a short idle period (typically 60s for Redis, 180s for PG).
- Backend service logs show no errors; security group rules are correct.
- Direct connection to backend (bypassing OpenResty) remains stable.

## Root Cause
OpenResty `stream` module has a default `proxy_timeout` that is often too short for development or long-lived connections:
- Redis default: 60 seconds
- PostgreSQL default: 180 seconds

When the client is idle beyond this threshold, OpenResty silently closes the upstream connection, causing the client to see a broken pipe or EOF.

## Diagnosis Steps
1. Confirm backend services are healthy and listening on expected ports.
2. Verify security group rules allow the proxy ports from the client IP.
3. Check current `proxy_timeout` in the relevant `server {}` block inside `stream {}`:
   ```bash
   grep -A5 'proxy_timeout' /usr/local/openresty/nginx/conf/nginx.conf
   ```
4. Compare the timeout value with observed disconnection intervals. If they match, `proxy_timeout` is the cause.

## Resolution
Adjust `proxy_timeout` to a value suitable for your use case. For local development environments, 3600s (1 hour) is typically sufficient:

```nginx
stream {
    # Redis proxy
    server {
        listen 27742;
        proxy_pass 127.0.0.1:6379;
        proxy_timeout 3600s;
        proxy_connect_timeout 10s;
    }

    # PostgreSQL proxy
    server {
        listen 27741;
        proxy_pass 127.0.0.1:5432;
        proxy_timeout 3600s;
        proxy_connect_timeout 10s;
    }
}
```

Validate and reload:
```bash
/usr/local/openresty/nginx/sbin/nginx -t
/usr/local/openresty/nginx/sbin/nginx -s reload
```

## Notes
- `proxy_timeout` controls **idle** read/write timeout, not total connection lifetime. Active connections are not affected.
- For production, set this based on actual application keepalive behavior rather than using arbitrarily large values.
- `proxy_connect_timeout` should remain short (e.g., 10s) to fail fast on unreachable backends.
- This issue is specific to the `stream` module; HTTP `proxy_read_timeout` / `proxy_send_timeout` are separate directives in the `http` context.
