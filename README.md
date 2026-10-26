# Hetzner Dynamic DNS Updater

## Concept
`hetzner_dyndns.php` keeps Hetzner DNS records in sync with a live client IP. It detects when the IPv4/IPv6 values change, stores the latest record IDs in SQLite and retries failed updates via a cron-safe path so no change is lost.

## Tech stack
- PHP (with `curl` + SQLite3) for the HTTP/cron endpoint and persistence.
- SQLite database (`hetzner_dyndns.sqlite3`) stores the hostname-to-record metadata plus retry state.
- Hetzner Console API (`api.hetzner.cloud/v1`). The legacy DNS API (`dns.hetzner.com/api/v1`) was shut down by Hetzner in May 2026 and is no longer supported by this script.
- Works on IPv4, dual-stack, and IPv6-only (e.g. DS-Lite) connections; A and AAAA records are updated for whichever address families the client supplies.

## Requirements
- PHP 7.1 or newer with the `curl` and `sqlite3` extensions enabled. Use a release that contains the October 2026 PHP2Shell fixes (8.2.35, 8.3.36, 8.4.27, 8.5.12 or later); on shared hosting select that version in the hoster's control panel.
- Apache with `mod_rewrite` and `.htaccess` overrides enabled — the shipped `.htaccess` provides the DynDNS endpoints and blocks direct script access.
- A Hetzner Console project API token with DNS read/write permissions (one per realm).
- The A/AAAA records must already exist in the zone; the updater only changes their values, it never creates records.

## Usage
1. Place `hetzner_dyndns.php`, `.htaccess`, `.user.ini` and `hetzner_dyndns.config.php.dist` into the document root, then copy the `.dist` file to `hetzner_dyndns.config.php`.
2. Configure authentication credentials and realms in `hetzner_dyndns.config.php` (`auth_user` and `auth_password`; user defaults to `update` if omitted).
3. Point your client at `/nic/update` or `/v3/update` and authenticate via HTTP Basic (or the legacy `X-Authentication` header / `p` parameter if your client only supports a shared password):
   ```
   https://ddns.example.com/nic/update?hostname=home.example.com&myip=203.0.113.7
   ```
   *Note: `.htaccess` rewrites `/nic/update` and `/v3/update` to `hetzner_dyndns.php` and blocks direct requests to the `.php` file, so always use the DynDNS-style paths.*
4. Schedule a cron job such as:
   ```
   */5 * * * * php /path/to/hetzner_dyndns.php --cron --realm=default
   ```
   so failed attempts are retried automatically; you can also hit `/nic/update?action=cron`. Omit `--realm` to process pending rows across all realms.

## Request parameters
| Parameter | Description |
| --- | --- |
| `hostname` | Required. FQDN to update; at least two labels so the zone can be derived. Multi-level subdomains are supported. |
| `myip` | IP address to set. Accepts a comma-separated list, so one IPv4 and one IPv6 address can be sent together. Falls back to the client's remote address when omitted. |
| `myip6` / `myipv6` | Dedicated IPv6 address, as sent by FRITZ!Box (`myip6`) and other clients (`myipv6`). When one of these is sent *without* `myip`, the remote-address fallback for IPv4 is skipped on purpose — behind DS-Lite/CGNAT it would write the carrier's IPv4 into the A record. |
| `realm` | Realm to use; falls back to `default_realm` from the configuration. |
| `p` | Legacy shared-password authentication for clients that cannot do HTTP Basic. |
| `action=cron` | Runs the retry pass over pending rows instead of a normal update. |

The remote address is taken from `X-Forwarded-For` when present, otherwise from `REMOTE_ADDR`. Only run this behind a reverse proxy you control, or always send `myip` explicitly.

### FRITZ!Box
Use a custom DynDNS provider with this update URL (the box substitutes the placeholders itself):
```
https://ddns.example.com/nic/update?hostname=<domain>&myip=<ipaddr>&myip6=<ip6addr>
```
Set username and password to the configured `auth_user` / `auth_password`.

## Responses
Responses follow the dyndns2 convention:

| Response | Meaning |
| --- | --- |
| `good <ip>` | Record was updated to the supplied address. |
| `nochg <ip>` | Stored addresses already matched, no API call was made. Some clients flag repeated `good` answers for the same IP as abusive, hence the distinction. |
| `nohost` | No `hostname` parameter was supplied (HTTP 400). |
| `Invalid domain name` | `hostname` failed validation. |
| `Unknown realm` | The `realm` parameter does not match a configured realm (HTTP 400). |
| `No valid IP address provided` | Neither a usable `myip` nor an IPv6 parameter nor a remote address was available. |
| HTTP 401 | Authentication failed. |
| HTTP 405 | Request method other than GET. |
| HTTP 503 | The Hetzner API call failed; the row is flagged for the cron retry and the error message is returned. |

## Command-line helper
- `hetzner_dyndns_listhosts.php` reads the same sqlite history database and prints each hostname alongside the current IPv4/IPv6 values, the configured realm, whether the row is pending, and when it was last updated.
- Run it only from the shell (`php hetzner_dyndns_listhosts.php`) so you can quickly audit what IPs are stored without invoking the HTTP endpoint. It refuses to run under a web SAPI.

## Configuration
- `hetzner_dyndns.config.php.dist` is the committed template; copy it to `hetzner_dyndns.config.php` and fill in the real values.
- Include shared metadata such as `auth_user` / `auth_password` (or just `auth_password` if you stick with the default `update` username), `history_db`, `default_realm`, each realm's `console_token`/TTL, plus the optional `auth_realm` label.
- The `console_token` is a Hetzner Console project API token with DNS read/write permissions; the legacy `dns_token`/`dns_endpoint`/`api_order` settings are ignored since the legacy API shutdown (a debug notice is logged if they are still present).
- `console_endpoint` defaults to `https://api.hetzner.cloud/v1` and normally needs no change.
- Optionally add `'zone_name' => 'your-zone.example.com'` per realm when the DNS zone sits under a subdomain; otherwise the updater infers the domain automatically.
- `lock_wait_seconds` (default `5`) controls how long the cron handler waits for the database lock, `max_retry_attempts` (default `5`) how often a failed row is retried before it is left for manual intervention.
- Add or adjust realms, tokens, and TTL values in that file, then rerun the cron to refresh pending rows.
- Only the exact rrset matching the hostname is updated. If the record does not exist, the updater reports an error instead of walking up the name hierarchy and rewriting the zone apex.

## Notifications
- Enable `'notifications.enabled' => true` to send an email after each update attempt.
- Set `'notifications.method'` to `'php'` to use `mail()` or `'smtp'` to speak directly to an authenticated SMTP server.
- Provide at least one recipient via `'notifications.recipients'` or use `'success_recipients'`/`'failure_recipients'` when you need different audiences per outcome.
- Control which events trigger mail with `'send_on_success'` and `'send_on_failure'` (defaults are `false`/`true`).
- Include SMTP credentials under `'notifications.smtp'` when `'method' => 'smtp'` so the script can authenticate and relay updates.

## Debug logging
- Enable debug mode (`debug => true`) to emit zone-lookup steps, HTTP requests, and API responses to PHP's error log (web server log or CLI stderr). This makes it easier to confirm which zone name is used, see HTTP status codes, and inspect any error payload the Hetzner APIs return.
- Optionally set `debug_log` to a writable file path so the script writes those diagnostics there instead of the default `error_log()` target; this is handy on shared hosting where you may not know where PHP's error log lives.

## Notes
- CLI invocations skip the HTTP authentication check by design — shell access to the host already implies authorization, and cron cannot carry HTTP credentials.
- Password comparisons use `hash_equals()`. The broken HTTP Digest branch was removed; use HTTP Basic over TLS.
- The script protects against concurrent access with SQLite pragmas and retry counters, so repeated failures are surfaced through the cron summary output.
- Keep the `history_db` file writable by the PHP process (defaults to `hetzner_dyndns.sqlite3` next to the script).
- `.htaccess` is deny-by-default: only `/nic/update` and `/v3/update` (rewritten into `hetzner_dyndns.php` with the query string intact) and non-PHP files below `/.well-known/` (ACME challenges) are served. Everything else - the PHP scripts, `hetzner_dyndns.sqlite3`, `hetzner_dyndns.debug.log`, the config template, dotfiles, backups - answers 403, whatever the file is called. Requests with a method other than GET/HEAD are refused before PHP sees them. The endpoint check is anchored on the request path: earlier versions matched `/nic/update` anywhere in the request line, so `/hetzner_dyndns_listhosts.php?/nic/update` slipped past the PHP block, and an anchored `^\.sqlite3$` pattern left the database downloadable. This only applies to Apache - on nginx or another server you have to deny those paths yourself.
- `.user.ini` (read by PHP-FPM/CGI, ignored by mod_php) turns `display_errors` off, disables session upload progress and caps POST sizes. Together with the GET-only rule this keeps the request paths used by PHP2Shell (CVE-2026-109618/-109619) and CVE-2026-115560 away from this endpoint. It is a mitigation, not a fix - keep PHP patched.
- The scripts never show PHP errors to HTTP clients; uncaught errors go to the server's error log instead of leaking filesystem paths.

[![Donate](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://www.paypal.me/FWoehrl)
