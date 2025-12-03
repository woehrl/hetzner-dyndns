# Hetzner Dynamic DNS Updater

## Concept
`hetzner_dyndns.php` keeps Hetzner DNS records in sync with a live client IP. It detects when the IPv4/IPv6 values change, stores the latest record IDs in SQLite and retries failed updates via a cron-safe path so no change is lost.

## Tech stack
- PHP (with `curl` + SQLite3) for the HTTP/cron endpoint and persistence.
- SQLite database (`hetzner_dyndns.sqlite3`) stores the hostname-to-record metadata plus retry state.
- Hetzner DNS API (legacy `dns.hetzner.com/api/v1`) plus the new Hetzner Console API (`api.hetzner.cloud/v1`) are both supported.

## Usage
1. Place `hetzner_dyndns.php` and `hetzner_dyndns.config.php.dist` into the document root, then copy the `.dist` file to `hetzner_dyndns.config.php`.
2. Configure authentication credentials and realms in `hetzner_dyndns.config.php`.
3. Point your client at `hetzner_dyndns.php` with `?hostname=...&myip=...&realm=<name>` or supply the password via `HTTP Basic`, `X-Authentication`, or the `p` parameter.
   *Note: `.htaccess` rewrites `/nic/update` and `/v3/update` to `hetzner_dyndns.php`, so you can also keep using the original DynDNS-style endpoints.*
4. Schedule a cron job such as:
   ```
   */5 * * * * php /path/to/hetzner_dyndns.php --cron --realm=default
   ```
   so failed attempts are retried automatically; you can also hit `hetzner_dyndns.php?action=cron`.

## Command-line helper
- `hetzner_dyndns_listhosts.php` reads the same sqlite history database and prints each hostname alongside the current IPv4/IPv6 values, the configured realm, whether the row is pending, and when it was last updated.
- Run it only from the shell (`php cli_list_hosts.php`) so you can quickly audit what IPs are stored without invoking the HTTP endpoint.

## Configuration
- `hetzner_dyndns.config.php.dist` is the committed template; copy it to `hetzner_dyndns.config.php` and fill in the real values.
- Include shared metadata such as `script_password`, `history_db`, `default_realm`, each realm’s tokens/TTL, and `api_order`, plus the optional `auth_realm` label.
- The `api_order` array determines which API(s) to attempt and in which order; omit `dns` from that list if you only want the Console API to run.
- Optionally add `'zone_name' => 'your-zone.example.com'` per realm when the DNS zone sits under a subdomain; otherwise the updater infers the domain automatically.
- Add or adjust realms, tokens, and TTL values in that file, then rerun the cron to refresh pending rows.

## Notifications
- Enable `'notifications.enabled' => true` to send an email after each update attempt.
- Set `'notifications.method'` to `'php'` to use `mail()` or `'smtp'` to speak directly to an authenticated SMTP server.
- Provide at least one recipient via `'notifications.recipients'` or use `'success_recipients'`/`'failure_recipients'` when you need different audiences per outcome.
- Control which events trigger mail with `'send_on_success'` and `'send_on_failure'` (defaults are `false`/`true`).
- Include SMTP credentials under `'notifications.smtp'` when `'method' => 'smtp'` so the script can authenticate and relay updates.

## Debug logging
- Enable debug mode (`debug => true`) to emit zone-lookup steps, HTTP requests, and API responses to PHP’s error log (web server log or CLI stderr). This makes it easier to confirm which zone name is used, see HTTP status codes, and inspect any error payload the Hetzner APIs return.
- Optionally set `debug_log` to a writable file path so the script writes those diagnostics there instead of the default `error_log()` target; this is handy on shared hosting where you may not know where PHP’s error log lives.

## Notes
- The script protects against concurrent access with SQLite pragmas and retry counters, so repeated failures are surfaced through the cron summary output.
- Keep the `history_db` file writable by the PHP process (defaults to `hetzner_dyndns.sqlite3` next to the script). 
- `.htaccess` blocks direct access to `hetzner_dyndns.php`, any other php script and related secrets while allowing DynDNS-standard paths (`/nic/update`, `/v3/update`) to proxy into the script with query parameters intact.

[![Donate](https://img.shields.io/badge/Donate-PayPal-blue.svg)](https://www.paypal.me/FWoehrl)