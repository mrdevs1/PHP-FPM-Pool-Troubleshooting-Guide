# PHP-FPM Pool Troubleshooting Guide

A practical troubleshooting guide for diagnosing PHP-FPM pool exhaustion, slow PHP requests, high process usage, and HTTP 502/504 errors on Linux web servers.

This guide is intended for hosting support engineers, Linux system administrators, PHP developers, and web hosting providers operating PHP-FPM environments, including cPanel/WHM, CloudLinux, and LiteSpeed configurations.

## Table of Contents

- [Overview](#overview)
- [Common Symptoms](#common-symptoms)
- [1. Check PHP-FPM Service Status](#1-check-php-fpm-service-status)
- [2. Identify PHP-FPM Processes](#2-identify-php-fpm-processes)
- [3. Investigate pm.max_children Exhaustion](#3-investigate-pmmax_children-exhaustion)
- [4. Understand PHP-FPM Pool Settings](#4-understand-php-fpm-pool-settings)
- [5. Estimate PHP-FPM Memory Usage](#5-estimate-php-fpm-memory-usage)
- [6. Review PHP-FPM Logs](#6-review-php-fpm-logs)
- [7. Troubleshoot HTTP 502 and 504 Errors](#7-troubleshoot-http-502-and-504-errors)
- [8. Diagnose Slow PHP Requests](#8-diagnose-slow-php-requests)
- [9. Apply Safe Configuration Changes](#9-apply-safe-configuration-changes)
- [10. cPanel, CloudLinux, and LiteSpeed Considerations](#10-cpanel-cloudlinux-and-litespeed-considerations)
- [11. Prevention and Monitoring](#11-prevention-and-monitoring)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Security and Production Safety](#security-and-production-safety)
- [Useful References](#useful-references)
- [Contributing](#contributing)
- [License](#license)

## Overview

PHP-FPM (FastCGI Process Manager) manages PHP worker processes that execute PHP applications. Each pool can have its own process-management configuration and resource limits.

When all available workers are busy, additional PHP requests may wait in a queue. If the queue grows or requests exceed configured timeouts, visitors may experience slow responses, HTTP 502/504 errors, or application failures.

Increasing the worker limit without checking available memory and CPU can make the problem worse. The correct solution begins with identifying the actual bottleneck.

## Common Symptoms

| Symptom | Possible cause |
|---|---|
| `server reached pm.max_children setting` | All configured pool workers are occupied |
| HTTP 502 Bad Gateway | PHP-FPM unavailable, crashed, or unreachable |
| HTTP 504 Gateway Timeout | Upstream PHP request exceeded a timeout |
| Slow WordPress or WooCommerce pages | Long PHP execution, database delays, or worker saturation |
| High memory usage | Too many workers, memory-heavy applications, or leaks |
| Large number of queued requests | Insufficient worker capacity or slow requests |
| PHP-FPM repeatedly restarting | Configuration errors, crashes, or resource exhaustion |

These symptoms indicate possible causes, not definitive diagnoses. Check the relevant logs and resource metrics before changing settings.

## 1. Check PHP-FPM Service Status

First, identify the PHP-FPM service and verify whether it is running.

### Check common service names

```bash
systemctl --type=service --state=running | grep -Ei 'php.*fpm'
```

Check a known service:

```bash
systemctl status php-fpm
```

On version-specific installations, the service may be named differently, such as:

```bash
systemctl status php8.2-fpm
```

Replace the example service name with the actual service installed on the server.

### Inspect recent service logs

```bash
journalctl -u php-fpm --since "30 minutes ago" --no-pager
```

For a version-specific service:

```bash
journalctl -u php8.2-fpm --since "30 minutes ago" --no-pager
```

Look for failed starts, worker exits, configuration errors, and repeated restarts.

**Important:** Do not restart services repeatedly before checking the logs. A restart may temporarily hide the symptoms without resolving the underlying problem.

## 2. Identify PHP-FPM Processes

Inspect running PHP-FPM processes:

```bash
ps -eo pid,ppid,user,%cpu,%mem,rss,etime,args --sort=-rss | grep '[p]hp-fpm'
```

Review overall system resource usage:

```bash
free -h
uptime
vmstat 1 5
```

If available, use `top` or `htop` to inspect CPU and memory consumption.

### What to investigate

- Whether the number of PHP workers approaches the configured pool limit.
- Whether one account consumes substantially more resources than others.
- Whether individual workers use unusually high memory.
- Whether CPU usage remains high during slow requests.
- Whether the server is under memory pressure or swapping heavily.

A high worker count alone does not prove a problem. Compare it with pool limits, request latency, available memory, and server workload.

## 3. Investigate pm.max_children Exhaustion

A common PHP-FPM warning is:

```text
WARNING: [pool example] server reached pm.max_children setting (10), consider raising it
```

This indicates that the pool reached its configured maximum number of child processes at that moment.

It does not automatically mean that the limit should be increased.

### Investigation steps

1. Identify which pool generated the warning.
2. Check how frequently the warning occurs.
3. Determine whether requests are slow or blocked.
4. Review the pool's `pm.max_children` value.
5. Measure worker memory consumption.
6. Check database response times and external API calls.
7. Determine whether additional worker capacity can safely be supported.

### Check pool configuration

If you know the relevant configuration directory:

```bash
grep -RniE '^\s*(pm|pm\.max_children|pm\.start_servers|pm\.min_spare_servers|pm\.max_spare_servers|pm\.max_requests)\s*=' /etc/php-fpm.d/ /etc/php/*/fpm/pool.d/ 2>/dev/null
```

Configuration paths vary by distribution, PHP version, hosting panel, and installation method. Do not assume every listed directory exists.

## 4. Understand PHP-FPM Pool Settings

PHP-FPM provides several process-management settings.

| Setting | Purpose |
|---|---|
| `pm` | Selects the process-management strategy |
| `pm.max_children` | Maximum number of child processes in a pool |
| `pm.start_servers` | Initial number of workers in dynamic mode |
| `pm.min_spare_servers` | Minimum idle workers in dynamic mode |
| `pm.max_spare_servers` | Maximum idle workers in dynamic mode |
| `pm.max_requests` | Requests handled by a child before it is recycled |
| `request_terminate_timeout` | Maximum permitted execution time for a request, if configured |
| `pm.status_path` | Optional endpoint for pool status information |

### Process-management modes

**`static`**

Maintains a fixed number of child processes. The `pm.max_children` setting determines the number of children.

**`dynamic`**

Adjusts the worker count based on demand and spare-worker settings, up to `pm.max_children`.

**`ondemand`**

Starts workers when requests arrive and allows idle workers to exit after the configured idle timeout.

Choose a mode based on workload patterns and the hosting environment. There is no single optimal mode for every server.

## 5. Estimate PHP-FPM Memory Usage

Before increasing `pm.max_children`, estimate the memory available for PHP-FPM workers.

Start with:

```bash
free -h
```

Then inspect worker RSS values:

```bash
ps -eo rss,args | awk '/php-fpm: pool/ {sum += $1; count++} END {if (count > 0) printf "Workers: %d\nAverage RSS: %.1f MiB\nTotal worker RSS: %.1f MiB\n", count, sum/count/1024, sum/1024; else print "No matching PHP-FPM pool workers found"}'
```

This is a rough estimate. RSS can include shared memory pages, so summing RSS may overstate the memory uniquely consumed by workers. Worker memory also varies by application and request.

### Basic capacity planning

A rough planning formula is:

```text
Estimated worker capacity =
    Memory budget allocated to PHP-FPM
    / Representative memory per worker
```

For example, if the PHP-FPM memory budget is approximately 1,200 MiB and representative worker memory is 60 MiB:

```text
1,200 / 60 = 20 workers
```

This is only an illustrative estimate, not a recommendation to configure 20 workers.

Reserve sufficient memory for the operating system, database, web server, caches, control panel, security services, and other workloads. Use measured peak behavior and an appropriate safety margin when choosing the final limit.

On CloudLinux or other resource-controlled environments, check the account's actual resource limits as well as the server-wide memory.

## 6. Review PHP-FPM Logs

PHP-FPM logs may contain warnings, startup failures, slow-request reports, and worker errors.

Common locations include:

```text
/var/log/php-fpm/
/var/log/php-fpm/error.log
/var/log/php8.2-fpm.log
```

The actual paths depend on the installation and configuration.

### Search for relevant errors

For a known log file:

```bash
grep -Ei 'max_children|slow|warning|error|failed|memory' /path/to/php-fpm.log | tail -n 100
```

Follow new entries:

```bash
tail -f /path/to/php-fpm.log
```

Use the actual log path on your server.

### Check web-server error logs

Apache and LiteSpeed deployments may log upstream failures in their respective error logs. Nginx deployments may record FastCGI errors in the Nginx error log.

Look for messages involving:

- Connection refused.
- Connection reset.
- Upstream timed out.
- Socket not found.
- Permission denied.
- Resource temporarily unavailable.

Correlate the timestamps across PHP-FPM, web-server, and application logs.

## 7. Troubleshoot HTTP 502 and 504 Errors

HTTP 502 and 504 errors can originate from several layers. Do not assume every error is caused by PHP-FPM worker exhaustion.

### Step 1: Confirm the response

```bash
curl -sS -o /dev/null -w 'HTTP status: %{http_code}\nTotal time: %{time_total}s\n' https://example.com/
```

Replace `example.com` with a site you are authorized to test.

Repeat the test when the issue occurs and compare results with server logs.

### Step 2: Check PHP-FPM availability

```bash
systemctl status php-fpm
```

Use the correct service name for the installed environment.

### Step 3: Check upstream connectivity

Depending on the configuration, PHP-FPM may communicate through a Unix socket or a TCP port.

For a Unix socket, verify that the configured socket exists and that the web-server process has appropriate permissions.

For TCP, inspect the configured address and port and verify that the listener is available.

Do not expose PHP-FPM directly to the public internet.

### Step 4: Compare timeout settings

Check relevant timeouts in:

- PHP-FPM pool configuration.
- PHP's `max_execution_time`.
- Nginx FastCGI configuration, if used.
- Apache or LiteSpeed upstream configuration.
- Reverse proxies or CDNs.

Increasing a timeout may help a legitimate long-running operation, but it will not fix an overloaded server or a request blocked by a database query.

## 8. Diagnose Slow PHP Requests

Slow requests can occupy workers for long periods and cause pool exhaustion even when the total traffic is moderate.

### Enable PHP-FPM slow logging carefully

Pool configuration can support settings similar to:

```ini
request_slowlog_timeout = 5s
slowlog = /var/log/php-fpm/example-slow.log
```

These are examples. Confirm that the directives are supported by your installed PHP-FPM version and that the log directory exists with suitable permissions.

The timeout is illustrative; select a threshold appropriate to the application.

After a configuration change, validate the configuration and reload the correct service during an appropriate maintenance window.

Slow logs can contain file paths and application details. Restrict access and avoid publishing them publicly.

### Investigate likely bottlenecks

- Slow SQL queries or database locks.
- Excessive WordPress plugins or expensive hooks.
- WooCommerce checkout or product queries.
- External HTTP requests that wait for another service.
- Large imports, scheduled jobs, or background tasks.
- Inefficient PHP code or unexpectedly large data processing.

Fix the slow operation where possible rather than increasing worker counts indefinitely.

## 9. Apply Safe Configuration Changes

Before modifying PHP-FPM settings:

1. Identify the exact pool and configuration file.
2. Confirm that the file is managed by the hosting panel or automation system.
3. Back up the existing configuration.
4. Check available memory and account resource limits.
5. Make one controlled change at a time.
6. Validate the configuration.
7. Reload or restart the correct service only when appropriate.
8. Monitor errors, memory use, and response times afterward.

### Back up a configuration file

```bash
cp -a /path/to/pool.conf /path/to/pool.conf.bak
```

Replace both paths with the actual configuration file and backup destination. Ensure the destination does not overwrite an existing backup.

### Validate configuration

Depending on the installation, a validation command may look like:

```bash
php-fpm -t
```

or:

```bash
php-fpm8.2 -t
```

Use the binary and validation options supported by your installation. Some systems require an explicit configuration file or service-specific binary.

### Apply changes

Once validation succeeds, use the service-management method appropriate to the installed PHP-FPM service and hosting panel.

Do not blindly copy a configuration from another server. PHP versions, pool layouts, socket paths, user permissions, and control-panel integration can differ.

### Roll back if necessary

If a change causes new errors, restore the backup using the appropriate maintenance procedure, validate the restored configuration, and reload the service if required.

Keep the original configuration available until the change has been verified.

## 10. cPanel, CloudLinux, and LiteSpeed Considerations

Hosting environments often manage PHP-FPM differently from a standard Linux package installation.

### cPanel/WHM

- Identify the domain's assigned PHP version.
- Check whether PHP-FPM is enabled for that domain.
- Determine whether the pool configuration is generated or managed by WHM.
- Use the supported WHM interface or documented configuration tools for changes.
- Avoid editing generated files that the control panel may overwrite.

Official documentation: https://docs.cpanel.net/

### CloudLinux

- Check account-level resource usage and limits.
- Determine whether CPU, memory, entry-process, or process limits are being reached.
- Distinguish server-wide PHP-FPM exhaustion from per-account resource restrictions.
- Use CloudLinux's supported interfaces and tools for the installed configuration.

Official documentation: https://docs.cloudlinux.com/

### LiteSpeed

LiteSpeed deployments may use LiteSpeed's own PHP handler configuration rather than a conventional PHP-FPM pool for every workload.

Before applying PHP-FPM-specific advice, confirm which PHP handler the site actually uses. Check LiteSpeed's external application configuration, worker limits, and error logs where applicable.

Official documentation: https://docs.litespeedtech.com/

**Important:** Do not apply PHP-FPM commands or settings to a LiteSpeed PHP handler unless that deployment actually uses PHP-FPM.

## 11. Prevention and Monitoring

Prevent recurring pool exhaustion by monitoring both demand and request duration.

Recommended practices:

- Alert on repeated `pm.max_children` warnings.
- Monitor memory pressure, CPU usage, and swap activity.
- Track PHP request latency and HTTP 5xx rates.
- Review slow logs when available.
- Optimize expensive database queries and application code.
- Keep PHP and application dependencies supported and updated.
- Set reasonable limits for background jobs and long-running operations.
- Review resource usage during peak traffic.
- Document configuration changes and rollback procedures.

If you enable the PHP-FPM status endpoint, restrict access to trusted monitoring systems or local administrative access. Do not expose sensitive process information publicly.

## Troubleshooting Checklist

Use this checklist during incident response:

- [ ] Confirm the affected website, PHP version, and PHP handler.
- [ ] Record the time and frequency of the issue.
- [ ] Check the relevant service status.
- [ ] Review PHP-FPM and web-server logs.
- [ ] Identify the affected pool and its worker limit.
- [ ] Inspect worker memory, CPU usage, and overall system pressure.
- [ ] Check account-level resource limits where applicable.
- [ ] Investigate slow application or database operations.
- [ ] Estimate safe worker capacity before changing limits.
- [ ] Back up and validate configuration changes.
- [ ] Apply changes through the supported management method.
- [ ] Monitor the result and document the resolution.

## Security and Production Safety

- Run diagnostic commands with the minimum privileges necessary.
- Do not expose PHP-FPM TCP listeners or status endpoints publicly.
- Avoid publishing logs containing customer data, session information, tokens, or internal paths.
- Do not disable security controls to work around performance problems.
- Avoid unplanned service restarts on production servers.
- Obtain authorization before investigating customer accounts.
- Use backups and rollback procedures before changing service configuration.
- Redact sensitive information from issue reports.

## Useful References

- [PHP Manual: PHP-FPM Configuration](https://www.php.net/manual/en/install.fpm.configuration.php)
- [PHP Manual: PHP-FPM](https://www.php.net/manual/en/install.fpm.php)
- [cPanel Documentation](https://docs.cpanel.net/)
- [CloudLinux Documentation](https://docs.cloudlinux.com/)
- [LiteSpeed Documentation](https://docs.litespeedtech.com/)
- [Nginx Documentation](https://nginx.org/en/docs/)

## Contributing

Contributions are welcome, especially:

- Verified troubleshooting examples from different Linux distributions.
- Additional PHP-FPM log examples with sensitive information removed.
- Safe diagnostic scripts and monitoring recommendations.
- Documentation improvements for cPanel, CloudLinux, and LiteSpeed.
- Corrections for version-specific behavior.

Please explain the affected environment, the observed symptoms, the diagnostic evidence, and the proposed fix. Do not include credentials, customer data, or unredacted production logs.

## License

This project is available under the MIT License. Add a `LICENSE` file containing the standard MIT License text and update the copyright holder and year as appropriate.
