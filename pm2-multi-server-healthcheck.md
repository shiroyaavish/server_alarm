# PM2 Multi-Server Health Check + Email Alerts

A lightweight Bash-based monitoring system for checking multiple PM2
applications, automatically restarting failed applications, and sending
email alerts when a service goes down or recovers.

The system is designed so that **one configuration file can contain
multiple servers/applications**.

------------------------------------------------------------------------

## 1. What This System Does

The monitoring system checks multiple applications every minute.

For each configured server:

1.  Calls its health API.
2.  Expects HTTP `200`.
3.  Retries failed health checks.
4.  Counts consecutive failures.
5.  Restarts the PM2 application after the configured failure threshold.
6.  Checks the application again after restart.
7.  Sends an email when:
    -   the application remains down,
    -   the application is automatically recovered,
    -   the application recovers after being down,
    -   the restart limit is reached.
8.  Prevents restart loops.
9.  Keeps per-server logs and state.
10. Runs automatically through cron.

The monitoring script itself runs on the same VPS where the PM2
applications are running.

> Important: This version supports **multiple PM2 applications/servers
> on one VPS** from one configuration file.
>
> If the applications are running on different VPS machines, see
> **Section 18 --- Monitoring Multiple VPS Machines**.

------------------------------------------------------------------------

# 2. Folder Structure

Use one folder for the monitoring system:

``` text
/opt/pm2-healthcheck/
├── pm2-healthcheck.sh       # Main monitoring script
├── servers.conf             # Multiple server/app configuration
├── crontab.txt              # Cron configuration
├── logs/
│   ├── cron.log
│   ├── server-1.log
│   ├── server-2.log
│   └── ...
└── state/
    ├── server-1/
    ├── server-2/
    └── ...
```

Create it:

``` bash
mkdir -p /opt/pm2-healthcheck/logs
mkdir -p /opt/pm2-healthcheck/state
```

Run the commands as the same Linux user that owns the PM2 processes.

For example:

``` bash
whoami
pm2 list
```

------------------------------------------------------------------------

# 3. Install Required Packages

Install the required tools:

``` bash
apt update
apt install -y curl msmtp msmtp-mta mailutils
```

The script also uses:

-   Bash
-   PM2
-   Node.js
-   `curl`
-   `flock`
-   `mail` / `sendmail`

Check them:

``` bash
bash --version
curl --version
pm2 --version
node --version
flock --version
```

------------------------------------------------------------------------

# 4. Configure Gmail SMTP

The system can use Gmail SMTP through an App Password.

Create a Gmail App Password from:

``` text
https://myaccount.google.com/apppasswords
```

Two-Step Verification must be enabled.

Do not use your normal Gmail password.

Remove spaces from the 16-character App Password.

Create:

``` bash
nano ~/.msmtprc
```

Example:

``` text
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /opt/pm2-healthcheck/logs/msmtp.log

account        gmail
host           smtp.gmail.com
port           587
from           yourname@gmail.com
user           yourname@gmail.com
password       YOUR_16_CHAR_APP_PASSWORD

account default : gmail
```

Set permissions:

``` bash
chmod 600 ~/.msmtprc
```

The password should exist only in this file.

------------------------------------------------------------------------

# 5. Test Email

Run:

``` bash
echo "PM2 health check email test" | mail -s "PM2 Health Check Test" you@example.com
```

Check the SMTP log:

``` bash
tail -F /opt/pm2-healthcheck/logs/msmtp.log
```

If the email arrives successfully, continue.

------------------------------------------------------------------------

# 6. Find the PM2 Environment

Check PM2:

``` bash
pm2 list
```

Example:

``` text
┌────┬───────────────┬─────────┬────────┬─────────┐
│ id │ name          │ mode    │ status │ restart │
├────┼───────────────┼─────────┼────────┼─────────┤
│ 0  │ api-server    │ fork    │ online │ 0       │
│ 1  │ admin-server  │ fork    │ online │ 0       │
│ 2  │ worker-server │ fork    │ online │ 0       │
└────┴───────────────┴─────────┴────────┴─────────┘
```

Find the PM2 executable:

``` bash
dirname "$(which pm2)"
```

For an NVM installation, it may look similar to:

``` text
/root/.nvm/versions/node/v24.21.0/bin
```

This path will be placed into `EXTRA_PATH`.

------------------------------------------------------------------------

# 7. Health API Requirements

Every monitored application should expose a health endpoint.

Example:

``` text
http://127.0.0.1:3000/health
```

The endpoint should return:

``` text
HTTP 200
```

Example response:

``` json
{
  "status": "ok"
}
```

Test manually:

``` bash
curl -i --max-time 10 http://127.0.0.1:3000/health
```

For another application:

``` bash
curl -i --max-time 10 http://127.0.0.1:4000/health
```

------------------------------------------------------------------------

# 8. Multi-Server Configuration

Instead of creating a separate configuration file for every application,
use one:

``` bash
nano /opt/pm2-healthcheck/servers.conf
```

Example:

``` bash
# ============================================================
# Global configuration
# ============================================================

ALERT_TO="first@example.com,second@example.com"

EXTRA_PATH="/root/.nvm/versions/node/v24.21.0/bin"

STATE_DIR="/opt/pm2-healthcheck/state"
LOG_DIR="/opt/pm2-healthcheck/logs"

EXPECT_CODE=200
CURL_TIMEOUT=10

RETRIES=3
RETRY_DELAY=3

FAIL_THRESHOLD=2

AUTO_RESTART=true
MAX_RESTARTS_PER_HOUR=3
RESTART_GRACE=15

REMINDER_MINUTES=30


# ============================================================
# Server configuration
#
# FORMAT:
#
# SERVER_NAME|PM2_APP|HEALTH_URL
#
# ============================================================

SERVERS=(
  "server-1|api-server|http://127.0.0.1:3000/health"
  "server-2|admin-server|http://127.0.0.1:4000/health"
  "server-3|worker-server|http://127.0.0.1:5000/health"
)
```

The important part is:

``` bash
SERVERS=(
  "server-1|api-server|http://127.0.0.1:3000/health"
  "server-2|admin-server|http://127.0.0.1:4000/health"
  "server-3|worker-server|http://127.0.0.1:5000/health"
)
```

Each line contains:

``` text
SERVER_NAME | PM2_APP | HEALTH_URL
```

------------------------------------------------------------------------

# 9. Adding More Servers

To add another application, simply add another line.

Example:

``` bash
SERVERS=(
  "server-1|api-server|http://127.0.0.1:3000/health"
  "server-2|admin-server|http://127.0.0.1:4000/health"
  "server-3|worker-server|http://127.0.0.1:5000/health"
  "server-4|payment-server|http://127.0.0.1:6000/health"
  "server-5|notification-server|http://127.0.0.1:7000/health"
)
```

No change to the monitoring script is required.

------------------------------------------------------------------------

# 10. Main Monitoring Script

Create:

``` bash
nano /opt/pm2-healthcheck/pm2-healthcheck.sh
```

Paste:

``` bash
#!/usr/bin/env bash

# ============================================================
# PM2 Multi-Server Health Check
#
# Usage:
#   pm2-healthcheck.sh /opt/pm2-healthcheck/servers.conf
# ============================================================

set -uo pipefail


# ------------------------------------------------------------
# Load configuration
# ------------------------------------------------------------

CONF="${1:-}"

if [[ -z "$CONF" || ! -r "$CONF" ]]; then
    echo "Usage: $0 /path/to/servers.conf" >&2
    exit 2
fi

# shellcheck disable=SC1090
source "$CONF"


# ------------------------------------------------------------
# Required configuration
# ------------------------------------------------------------

: "${ALERT_TO:?ALERT_TO missing in $CONF}"
: "${SERVERS:?SERVERS missing in $CONF}"


# ------------------------------------------------------------
# Defaults
# ------------------------------------------------------------

STATE_DIR="${STATE_DIR:-/opt/pm2-healthcheck/state}"
LOG_DIR="${LOG_DIR:-/opt/pm2-healthcheck/logs}"

EXPECT_CODE="${EXPECT_CODE:-200}"
CURL_TIMEOUT="${CURL_TIMEOUT:-10}"

RETRIES="${RETRIES:-3}"
RETRY_DELAY="${RETRY_DELAY:-3}"

FAIL_THRESHOLD="${FAIL_THRESHOLD:-2}"

AUTO_RESTART="${AUTO_RESTART:-true}"
MAX_RESTARTS_PER_HOUR="${MAX_RESTARTS_PER_HOUR:-3}"
RESTART_GRACE="${RESTART_GRACE:-15}"

REMINDER_MINUTES="${REMINDER_MINUTES:-30}"

EXTRA_PATH="${EXTRA_PATH:-}"

export PATH="${EXTRA_PATH:+$EXTRA_PATH:}/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin"


# ------------------------------------------------------------
# Create directories
# ------------------------------------------------------------

mkdir -p "$STATE_DIR"
mkdir -p "$LOG_DIR"


# ------------------------------------------------------------
# Check required commands
# ------------------------------------------------------------

for cmd in curl pm2 flock node mail; do

    if ! command -v "$cmd" >/dev/null 2>&1; then
        echo "ERROR: '$cmd' not found in PATH=$PATH" >&2
        exit 3
    fi

done


# ------------------------------------------------------------
# Email helper
# ------------------------------------------------------------

send_mail() {

    local subject="$1"
    local body="$2"

    local sm

    sm="$(command -v sendmail || true)"

    if [[ -x "$sm" ]]; then

        printf \
'To: %s\nFrom: pm2-healthcheck\nSubject: %s\nContent-Type: text/plain; charset=UTF-8\n\n%s\n' \
            "$ALERT_TO" \
            "$subject" \
            "$body" \
        | "$sm" -t

        return $?

    fi


    if command -v mail >/dev/null 2>&1; then

        printf '%s\n' "$body" \
        | mail -s "$subject" "$ALERT_TO"

        return $?

    fi


    echo "ERROR: no mail/sendmail command available" >&2

    return 1
}


# ------------------------------------------------------------
# Logging
# ------------------------------------------------------------

log() {

    local server_name="$1"
    shift

    local message="$*"

    local file="$LOG_DIR/${server_name}.log"

    local line

    line="$(date '+%F %T') [$server_name] $message"

    echo "$line" >> "$file"

}


# ------------------------------------------------------------
# PM2 status
# ------------------------------------------------------------

pm2_status() {

    local app="$1"

    pm2 jlist 2>/dev/null \
    | node -e '

        let d = "";

        process.stdin
            .on("data", c => d += c)
            .on("end", () => {

                try {

                    const a = JSON.parse(
                        d.slice(d.indexOf("["))
                    );

                    const app = process.argv[1];

                    const p = a.filter(
                        x => x.name === app
                    );

                    if (!p.length) {

                        console.log("not_found");

                    } else {

                        console.log(
                            p.map(
                                x =>
                                x.pm2_env.status +
                                "(restarts=" +
                                x.pm2_env.restart_time +
                                ")"
                            ).join(", ")
                        );

                    }

                } catch (e) {

                    console.log("unknown");

                }

            });

    ' "$app" 2>/dev/null || echo "unknown"
}


# ------------------------------------------------------------
# Restart count
# ------------------------------------------------------------

restarts_last_hour() {

    local restart_log="$1"

    local now
    local cutoff
    local tmp

    now="$(date +%s)"

    cutoff=$((now - 3600))

    touch "$restart_log"

    tmp="$(
        awk -v c="$cutoff" '$1 >= c' "$restart_log"
    )"

    printf '%s\n' "$tmp" \
    | sed '/^$/d' \
    > "$restart_log"

    wc -l < "$restart_log" \
    | tr -d ' '
}


# ------------------------------------------------------------
# Duration
# ------------------------------------------------------------

human_duration() {

    local seconds="$1"

    printf '%dh %dm' \
        $((seconds / 3600)) \
        $(((seconds % 3600) / 60))
}


# ------------------------------------------------------------
# Health check
# ------------------------------------------------------------

check_health() {

    local health_url="$1"

    local body_tmp="$2"

    local last_code="000"

    local i

    for ((i = 1; i <= RETRIES; i++)); do

        last_code="$(
            curl \
                -s \
                -m "$CURL_TIMEOUT" \
                -o "$body_tmp" \
                -w '%{http_code}' \
                "$health_url" \
                2>/dev/null
        )" || true


        if [[ "$last_code" == "$EXPECT_CODE" ]]; then

            echo "$last_code"

            return 0

        fi


        if (( i < RETRIES )); then

            sleep "$RETRY_DELAY"

        fi

    done


    echo "$last_code"

    return 1
}


# ------------------------------------------------------------
# Diagnostics
# ------------------------------------------------------------

diagnostics() {

    local server_name="$1"
    local pm2_app="$2"
    local health_url="$3"
    local last_code="$4"

    echo "Server       : $server_name"
    echo "PM2 App      : $pm2_app"
    echo "PM2 Status   : $(pm2_status "$pm2_app")"
    echo "HTTP Code    : $last_code (expected $EXPECT_CODE)"
    echo "Health URL   : $health_url"
    echo "Time         : $(date)"
    echo "Uptime       : $(uptime -p 2>/dev/null || echo unknown)"
    echo "Disk /       : $(df -h / | awk 'NR==2{print $5" used, "$4" free"}')"
    echo "Memory       : $(free -m | awk 'NR==2{print $3"MB used / "$2"MB total"}')"

    echo

    echo "--- Last PM2 log lines ---"

    pm2 logs "$pm2_app" \
        --lines 25 \
        --nostream \
        --raw \
        2>&1 \
    | tail -n 40
}


# ------------------------------------------------------------
# Process one server
# ------------------------------------------------------------

check_server() {

    local server_name="$1"
    local pm2_app="$2"
    local health_url="$3"


    # --------------------------------------------------------
    # Per-server state
    # --------------------------------------------------------

    local server_state_dir="$STATE_DIR/$server_name"

    mkdir -p "$server_state_dir"


    local fail_file="$server_state_dir/fails"
    local alert_file="$server_state_dir/alert"
    local restart_log="$server_state_dir/restarts"
    local lock_file="$server_state_dir/lock"


    local body_tmp

    body_tmp="$(mktemp)"


    trap 'rm -f "$body_tmp"' RETURN


    # --------------------------------------------------------
    # Prevent overlapping checks
    # --------------------------------------------------------

    exec 9>"$lock_file"

    if ! flock -n 9; then

        log "$server_name" \
            "Another check is already running. Skipping."

        return 0

    fi


    # --------------------------------------------------------
    # Health check
    # --------------------------------------------------------

    local last_code

    if last_code="$(check_health "$health_url" "$body_tmp")"; then

        # ----------------------------------------------------
        # HEALTHY
        # ----------------------------------------------------

        echo 0 > "$fail_file"


        if [[ -f "$alert_file" ]]; then

            local down_since
            local down_for

            down_since="$(
                awk -F= '/^DOWN_SINCE=/{print $2}' "$alert_file"
            )"

            down_since="${down_since:-$(date +%s)}"

            down_for=$(
                $(date +%s) - down_since
            )


            rm -f "$alert_file"


            log "$server_name" \
                "RECOVERED after $(human_duration "$down_for")"


            send_mail \
                "[RECOVERED] $server_name" \
                "$server_name is healthy again.

PM2 App: $pm2_app
Total downtime: $(human_duration "$down_for")

$(diagnostics \
    "$server_name" \
    "$pm2_app" \
    "$health_url" \
    "$last_code")"


        else

            log "$server_name" \
                "healthy (HTTP $last_code)"

        fi


        return 0

    fi


    # --------------------------------------------------------
    # UNHEALTHY
    # --------------------------------------------------------

    local fails

    fails="$(
        cat "$fail_file" 2>/dev/null || echo 0
    )"

    fails=$((fails + 1))

    echo "$fails" > "$fail_file"


    log "$server_name" \
        "health check failed (HTTP=$last_code, consecutive=$fails/$FAIL_THRESHOLD)"


    if (( fails < FAIL_THRESHOLD )); then

        return 0

    fi


    # --------------------------------------------------------
    # Confirmed DOWN
    # --------------------------------------------------------

    local restart_note="Auto-restart disabled."


    # --------------------------------------------------------
    # Auto restart
    # --------------------------------------------------------

    if [[ "$AUTO_RESTART" == "true" ]]; then

        local count

        count="$(
            restarts_last_hour "$restart_log"
        )"


        if (( count >= MAX_RESTARTS_PER_HOUR )); then

            restart_note="Restart limit reached ($count in last hour). NOT restarting. Manual action needed."

            log "$server_name" "$restart_note"


        else

            log "$server_name" \
                "Restarting PM2 app $pm2_app (attempt $((count + 1))/$MAX_RESTARTS_PER_HOUR this hour)"


            date +%s >> "$restart_log"


            if pm2 describe "$pm2_app" >/dev/null 2>&1; then

                pm2 restart "$pm2_app" \
                    >> "$LOG_DIR/${server_name}.log" \
                    2>&1

            else

                pm2 start "$pm2_app" \
                    >> "$LOG_DIR/${server_name}.log" \
                    2>&1

            fi


            sleep "$RESTART_GRACE"


            if last_code="$(check_health "$health_url" "$body_tmp")"; then

                echo 0 > "$fail_file"


                log "$server_name" \
                    "auto-restart succeeded"


                send_mail \
                    "[AUTO-RECOVERED] $server_name" \
                    "$server_name failed its health check and was automatically restarted.

PM2 App: $pm2_app

The service is healthy again.

$(diagnostics \
    "$server_name" \
    "$pm2_app" \
    "$health_url" \
    "$last_code")"


                return 0

            fi


            restart_note="PM2 restart did not fix the problem. Service is still down."

            log "$server_name" "$restart_note"

        fi

    fi


    # --------------------------------------------------------
    # Still DOWN
    # --------------------------------------------------------

    local now

    now="$(date +%s)"


    local down_since=0
    local last_alert=0


    if [[ -f "$alert_file" ]]; then

        down_since="$(
            awk -F= '/^DOWN_SINCE=/{print $2}' "$alert_file"
        )"

        last_alert="$(
            awk -F= '/^LAST_ALERT=/{print $2}' "$alert_file"
        )"

    fi


    down_since="${down_since:-0}"
    last_alert="${last_alert:-0}"


    # --------------------------------------------------------
    # First DOWN alert
    # --------------------------------------------------------

    if [[ ! -f "$alert_file" ]]; then

        printf \
            'DOWN_SINCE=%s\nLAST_ALERT=%s\n' \
            "$now" \
            "$now" \
            > "$alert_file"


        log "$server_name" \
            "sending DOWN alert"


        send_mail \
            "[ALERT] $server_name is DOWN" \
            "$server_name is not responding on its health API.

PM2 App: $pm2_app
Health URL: $health_url
HTTP Code: $last_code

Action taken:
$restart_note

$(diagnostics \
    "$server_name" \
    "$pm2_app" \
    "$health_url" \
    "$last_code")"


        return 0

    fi


    # --------------------------------------------------------
    # Reminder
    # --------------------------------------------------------

    if (( now - last_alert >= REMINDER_MINUTES * 60 )); then

        printf \
            'DOWN_SINCE=%s\nLAST_ALERT=%s\n' \
            "$down_since" \
            "$now" \
            > "$alert_file"


        log "$server_name" \
            "sending STILL DOWN reminder"


        send_mail \
            "[STILL DOWN] $server_name ($(human_duration $((now - down_since))))" \
            "$server_name has been down for $(human_duration $((now - down_since))).

PM2 App: $pm2_app
Health URL: $health_url
HTTP Code: $last_code

Action taken:
$restart_note

$(diagnostics \
    "$server_name" \
    "$pm2_app" \
    "$health_url" \
    "$last_code")"

    fi

}


# ============================================================
# MAIN
# ============================================================

for SERVER in "${SERVERS[@]}"; do

    IFS='|' read -r SERVER_NAME PM2_APP HEALTH_URL <<< "$SERVER"


    if [[ -z "$SERVER_NAME" ||
          -z "$PM2_APP" ||
          -z "$HEALTH_URL" ]]; then

        echo "Invalid server configuration:"
        echo "$SERVER"

        continue

    fi


    check_server \
        "$SERVER_NAME" \
        "$PM2_APP" \
        "$HEALTH_URL"

done

exit 0
```

Make it executable:

``` bash
chmod 755 /opt/pm2-healthcheck/pm2-healthcheck.sh
```

------------------------------------------------------------------------

# 11. Important Fix to the Duration Calculation

If you copy the script above, make sure the `down_for` calculation uses
Bash arithmetic exactly like this:

``` bash
down_for=$(( $(date +%s) - down_since ))
```

So this section should be:

``` bash
down_since="$(
    awk -F= '/^DOWN_SINCE=/{print $2}' "$alert_file"
)"

down_since="${down_since:-$(date +%s)}"

down_for=$(( $(date +%s) - down_since ))
```

This produces values such as:

``` text
0h 2m
0h 30m
2h 15m
```

------------------------------------------------------------------------

# 12. Example Complete Configuration

For three PM2 applications:

``` bash
nano /opt/pm2-healthcheck/servers.conf
```

Use:

``` bash
ALERT_TO="admin@example.com"

EXTRA_PATH="/root/.nvm/versions/node/v24.21.0/bin"

STATE_DIR="/opt/pm2-healthcheck/state"
LOG_DIR="/opt/pm2-healthcheck/logs"

EXPECT_CODE=200

CURL_TIMEOUT=10

RETRIES=3
RETRY_DELAY=3

FAIL_THRESHOLD=2

AUTO_RESTART=true

MAX_RESTARTS_PER_HOUR=3

RESTART_GRACE=15

REMINDER_MINUTES=30


SERVERS=(
    "API Server|api-server|http://127.0.0.1:3000/health"
    "Admin Server|admin-server|http://127.0.0.1:4000/health"
    "Worker Server|worker-server|http://127.0.0.1:5000/health"
)
```

------------------------------------------------------------------------

# 13. How the Failure Logic Works

Suppose:

``` text
API Server
PM2 app: api-server
Health API: http://127.0.0.1:3000/health
```

The cron runs every minute.

### First failed check

``` text
health check failed
consecutive=1/2
```

No restart yet.

### Second failed check

``` text
health check failed
consecutive=2/2
```

The system considers the service down.

It then runs:

``` bash
pm2 restart api-server
```

### Restart successful

After the configured grace period:

``` text
HTTP 200
```

The system sends:

``` text
[AUTO-RECOVERED] API Server
```

### Restart unsuccessful

If the health API still fails:

``` text
PM2 restart did not fix the problem.
```

The system sends:

``` text
[ALERT] API Server is DOWN
```

------------------------------------------------------------------------

# 14. Restart Loop Protection

The script should never endlessly restart an application.

The default is:

``` bash
MAX_RESTARTS_PER_HOUR=3
```

That means the monitoring system allows a maximum of three automatic
restart attempts within one hour.

For example:

``` text
10:05 → restart
10:15 → restart
10:25 → restart
10:30 → NO restart
```

The fourth attempt produces:

``` text
Restart limit reached (3 in last hour).
NOT restarting.
Manual action needed.
```

This prevents a broken application from continuously restarting.

------------------------------------------------------------------------

# 15. Cron Configuration

Create:

``` bash
nano /opt/pm2-healthcheck/crontab.txt
```

Add:

``` text
MAILTO=""
* * * * * /opt/pm2-healthcheck/pm2-healthcheck.sh /opt/pm2-healthcheck/servers.conf >> /opt/pm2-healthcheck/logs/cron.log 2>&1
```

Install:

``` bash
crontab -l > /root/crontab.backup 2>/dev/null
crontab /opt/pm2-healthcheck/crontab.txt
```

Check:

``` bash
crontab -l
```

You should see:

``` text
MAILTO=""
* * * * * /opt/pm2-healthcheck/pm2-healthcheck.sh /opt/pm2-healthcheck/servers.conf >> /opt/pm2-healthcheck/logs/cron.log 2>&1
```

> `crontab <file>` replaces the existing user's crontab. Always keep the
> backup.

------------------------------------------------------------------------

# 16. Test Manually

Before relying on cron, run:

``` bash
/opt/pm2-healthcheck/pm2-healthcheck.sh \
    /opt/pm2-healthcheck/servers.conf
```

Check exit status:

``` bash
echo $?
```

Normally:

``` text
0
```

Check logs:

``` bash
ls -lah /opt/pm2-healthcheck/logs/
```

For a particular server:

``` bash
tail -F /opt/pm2-healthcheck/logs/API\ Server.log
```

It is usually easier to avoid spaces in server names.

Recommended:

``` bash
"api-server|api-server|http://127.0.0.1:3000/health"
```

instead of:

``` bash
"API Server|api-server|http://127.0.0.1:3000/health"
```

------------------------------------------------------------------------

# 17. Recommended Server Naming

Use simple names:

``` bash
SERVERS=(
    "api-server|api-server|http://127.0.0.1:3000/health"
    "admin-server|admin-server|http://127.0.0.1:4000/health"
    "worker-server|worker-server|http://127.0.0.1:5000/health"
)
```

This creates:

``` text
logs/
├── api-server.log
├── admin-server.log
└── worker-server.log
```

And:

``` text
state/
├── api-server/
├── admin-server/
└── worker-server/
```

------------------------------------------------------------------------

# 18. Monitoring Multiple VPS Machines

The configuration above monitors multiple PM2 applications on **one
VPS**.

If your requirement is instead:

``` text
VPS 1
├── api-server
└── admin-server

VPS 2
├── payment-server
└── worker-server

VPS 3
└── notification-server
```

then use a different architecture.

A central monitoring machine can maintain:

``` bash
SERVERS=(
    "vps-1|123.123.123.10|api-server|http://127.0.0.1:3000/health"
    "vps-2|123.123.123.11|payment-server|http://127.0.0.1:4000/health"
    "vps-3|123.123.123.12|notification-server|http://127.0.0.1:5000/health"
)
```

The central monitor would then connect to each VPS using SSH or check
publicly reachable health URLs.

This is different from the local multi-PM2 configuration in this
document.

For security, SSH keys should be used instead of storing passwords in
the script.

------------------------------------------------------------------------

# 19. Recovery Email

When an application becomes healthy after being down, the system sends:

``` text
[RECOVERED] api-server
```

Example:

``` text
api-server is healthy again.

PM2 App: api-server
Total downtime: 0h 12m

Server       : api-server
PM2 App      : api-server
PM2 Status   : online(restarts=2)
HTTP Code    : 200
Health URL   : http://127.0.0.1:3000/health
```

------------------------------------------------------------------------

# 20. Automatic Recovery Email

If PM2 restart immediately fixes the service:

``` text
[AUTO-RECOVERED] api-server
```

Example:

``` text
api-server failed its health check and was automatically restarted.

PM2 App: api-server

The service is healthy again.
```

This is useful because you know the system fixed the issue without
manual intervention.

------------------------------------------------------------------------

# 21. DOWN Email

If the service remains unavailable:

``` text
[ALERT] api-server is DOWN
```

The email contains:

``` text
Server
PM2 application
Health URL
HTTP status
Action taken
PM2 status
Uptime
Disk usage
Memory usage
Recent PM2 logs
```

------------------------------------------------------------------------

# 22. STILL DOWN Reminders

The first alert is sent immediately after the confirmed failure.

After that, the system does not send an email every minute.

Default:

``` bash
REMINDER_MINUTES=30
```

So:

``` text
10:02 → ALERT
10:32 → STILL DOWN
11:02 → STILL DOWN
11:32 → STILL DOWN
```

This prevents email spam.

Change it if required:

``` bash
REMINDER_MINUTES=60
```

------------------------------------------------------------------------

# 23. Logs

Main cron log:

``` bash
tail -F /opt/pm2-healthcheck/logs/cron.log
```

Individual application:

``` bash
tail -F /opt/pm2-healthcheck/logs/api-server.log
```

SMTP log:

``` bash
tail -F /opt/pm2-healthcheck/logs/msmtp.log
```

Check whether cron executed:

``` bash
grep CRON /var/log/syslog | tail
```

------------------------------------------------------------------------

# 24. Log Rotation

Create:

``` bash
nano /etc/logrotate.d/pm2-healthcheck
```

Add:

``` text
/opt/pm2-healthcheck/logs/*.log {
    weekly
    rotate 4
    compress
    missingok
    notifempty
    copytruncate
}
```

This keeps approximately four weeks of rotated logs.

------------------------------------------------------------------------

# 25. PM2 Startup After Reboot

Make sure PM2 itself starts after a VPS reboot:

``` bash
pm2 startup
```

Run the command PM2 gives you.

Then:

``` bash
pm2 save
```

Verify:

``` bash
pm2 list
```

------------------------------------------------------------------------

# 26. Full Test

## Terminal 1

Watch the monitoring log:

``` bash
tail -F /opt/pm2-healthcheck/logs/api-server.log
```

## Terminal 2

Stop the application:

``` bash
pm2 stop api-server
```

The next health checks should detect:

``` text
health check failed
```

After the configured threshold:

``` text
restarting via pm2
```

Then:

``` text
auto-restart succeeded
```

You should receive:

``` text
[AUTO-RECOVERED] api-server
```

------------------------------------------------------------------------

# 27. Test a Real Health Failure

To test the health endpoint independently:

``` bash
curl -i --max-time 10 http://127.0.0.1:3000/health
```

Expected:

``` text
HTTP/1.1 200 OK
```

If the service is down:

``` text
HTTP code: 000
```

The script treats that as unhealthy.

------------------------------------------------------------------------

# 28. Test All Configured Servers

Run:

``` bash
/opt/pm2-healthcheck/pm2-healthcheck.sh \
    /opt/pm2-healthcheck/servers.conf
```

The script will process:

``` text
server-1
server-2
server-3
server-4
...
```

one by one.

A healthy run may produce:

``` text
server-1 → healthy
server-2 → healthy
server-3 → healthy
```

Failures are written to the corresponding log files.

------------------------------------------------------------------------

# 29. Debug Mode

If something is not working, run:

``` bash
bash -x \
/opt/pm2-healthcheck/pm2-healthcheck.sh \
/opt/pm2-healthcheck/servers.conf \
2>&1 | tail -100
```

This shows the commands being executed.

Also check:

``` bash
echo "$PATH"
which pm2
which node
which curl
which mail
```

------------------------------------------------------------------------

# 30. Common Problems

## PM2 command not found

Find PM2:

``` bash
which pm2
```

Then:

``` bash
dirname "$(which pm2)"
```

Set:

``` bash
EXTRA_PATH="/root/.nvm/versions/node/v24.21.0/bin"
```

------------------------------------------------------------------------

## Health URL returns 000

Test:

``` bash
curl -i --max-time 10 http://127.0.0.1:3000/health
```

Possible causes:

-   Application is stopped.
-   Wrong port.
-   Wrong health endpoint.
-   Application is listening on another interface.
-   Firewall/network issue.
-   Application is still starting.

------------------------------------------------------------------------

## Health URL returns 404

The endpoint exists incorrectly.

Verify your application route.

Example expected:

``` text
GET /health
```

------------------------------------------------------------------------

## Email is not sent

Check:

``` bash
tail -100 /opt/pm2-healthcheck/logs/msmtp.log
```

Also test:

``` bash
echo "test" | mail -s "Test" you@example.com
```

Check:

``` bash
ls -l ~/.msmtprc
```

It should normally be readable only by the owner:

``` text
-rw------- 
```

------------------------------------------------------------------------

## Cron works manually but not automatically

Check:

``` bash
crontab -l
```

Then:

``` bash
tail -100 /opt/pm2-healthcheck/logs/cron.log
```

Cron has a limited environment, so `EXTRA_PATH` is especially important
when PM2 is installed using NVM.

------------------------------------------------------------------------

# 31. Recommended Production Configuration

For a normal production API:

``` bash
EXPECT_CODE=200

CURL_TIMEOUT=10

RETRIES=3

RETRY_DELAY=3

FAIL_THRESHOLD=2

AUTO_RESTART=true

MAX_RESTARTS_PER_HOUR=3

RESTART_GRACE=15

REMINDER_MINUTES=30
```

This means:

``` text
Health check
     ↓
3 curl attempts
     ↓
Still failed?
     ↓
2 consecutive failed cron checks
     ↓
Consider application DOWN
     ↓
PM2 restart
     ↓
Wait 15 seconds
     ↓
Health check
     ↓
 ┌───────────────┐
 │               │
Healthy        Failed
 │               │
 ↓               ↓
Recovery       DOWN alert
email          + reminders
```

------------------------------------------------------------------------

# 32. Example Production Configuration

``` bash
ALERT_TO="devops@example.com,admin@example.com"

EXTRA_PATH="/root/.nvm/versions/node/v24.21.0/bin"

STATE_DIR="/opt/pm2-healthcheck/state"

LOG_DIR="/opt/pm2-healthcheck/logs"

EXPECT_CODE=200

CURL_TIMEOUT=10

RETRIES=3

RETRY_DELAY=3

FAIL_THRESHOLD=2

AUTO_RESTART=true

MAX_RESTARTS_PER_HOUR=3

RESTART_GRACE=15

REMINDER_MINUTES=30


SERVERS=(
    "api-server|api-server|http://127.0.0.1:3000/health"
    "admin-server|admin-server|http://127.0.0.1:4000/health"
    "worker-server|worker-server|http://127.0.0.1:5000/health"
    "payment-server|payment-server|http://127.0.0.1:6000/health"
)
```

------------------------------------------------------------------------

# 33. Security Recommendations

Do not put Gmail credentials inside:

``` text
servers.conf
```

Keep SMTP credentials in:

``` text
~/.msmtprc
```

with:

``` bash
chmod 600 ~/.msmtprc
```

Do not expose internal health endpoints publicly unless required.

Prefer:

``` text
http://127.0.0.1:3000/health
```

over exposing:

``` text
http://SERVER_IP:3000/health
```

when the monitoring script runs on the same VPS.

For multiple VPS monitoring, use SSH keys instead of passwords.

------------------------------------------------------------------------

# 34. Final File Layout

After installation:

``` text
/opt/pm2-healthcheck/
│
├── pm2-healthcheck.sh
├── servers.conf
├── crontab.txt
│
├── logs/
│   ├── cron.log
│   ├── msmtp.log
│   ├── api-server.log
│   ├── admin-server.log
│   ├── worker-server.log
│   └── payment-server.log
│
└── state/
    ├── api-server/
    │   ├── fails
    │   ├── alert
    │   ├── restarts
    │   └── lock
    │
    ├── admin-server/
    │
    ├── worker-server/
    │
    └── payment-server/
```

------------------------------------------------------------------------

# 35. Quick Installation

For a fresh server:

``` bash
mkdir -p /opt/pm2-healthcheck/logs
mkdir -p /opt/pm2-healthcheck/state

apt update

apt install -y curl msmtp msmtp-mta mailutils

chmod 600 ~/.msmtprc

chmod 755 /opt/pm2-healthcheck/pm2-healthcheck.sh

/opt/pm2-healthcheck/pm2-healthcheck.sh \
    /opt/pm2-healthcheck/servers.conf

echo $?

crontab /opt/pm2-healthcheck/crontab.txt

crontab -l

pm2 save
```

------------------------------------------------------------------------

# 36. Adding a New Application

The main benefit of this architecture is that adding a new application
is simple.

Add one line:

``` bash
SERVERS=(
    "api-server|api-server|http://127.0.0.1:3000/health"
    "admin-server|admin-server|http://127.0.0.1:4000/health"
    "new-server|new-server|http://127.0.0.1:8000/health"
)
```

Then test:

``` bash
/opt/pm2-healthcheck/pm2-healthcheck.sh \
    /opt/pm2-healthcheck/servers.conf
```

No cron change is required.

No new monitoring script is required.

No new state configuration is required.

------------------------------------------------------------------------

# 37. Final Architecture

``` text
                         CRON
                          │
                          │ Every minute
                          ▼
              pm2-healthcheck.sh
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
          Server 1     Server 2     Server 3
             │            │            │
             ▼            ▼            ▼
          Health API   Health API   Health API
             │            │            │
             ▼            ▼            ▼
          HTTP 200?    HTTP 200?    HTTP 200?
             │            │            │
       ┌─────┴─────┐ ┌────┴────┐ ┌────┴────┐
       │           │ │         │ │         │
     Healthy     Failed     Healthy     Failed
       │           │          │           │
       ▼           ▼          ▼           ▼
      Done      Retry       Done       Retry
                   │                      │
                   ▼                      ▼
             Failure threshold      Failure threshold
                   │                      │
                   ▼                      ▼
              PM2 restart            PM2 restart
                   │                      │
                   ▼                      ▼
             Health check            Health check
                │     │                 │    │
                ▼     ▼                 ▼    ▼
             Healthy Down            Healthy Down
                │     │                 │    │
                ▼     ▼                 ▼    ▼
            Recovery Alert          Recovery Alert
                    │
                    ▼
                  EMAIL
```

------------------------------------------------------------------------

# 38. Summary

The final system provides:

-   Multiple PM2 applications from one configuration file.
-   One monitoring script.
-   One cron job.
-   Individual health URLs.
-   Individual PM2 application names.
-   Individual server logs.
-   Individual state directories.
-   HTTP health checking.
-   Configurable retries.
-   Configurable failure threshold.
-   Automatic PM2 restart.
-   Restart-loop protection.
-   Recovery detection.
-   Immediate DOWN alerts.
-   Periodic STILL DOWN reminders.
-   AUTO-RECOVERED notifications.
-   RECOVERED notifications.
-   PM2 diagnostics.
-   Disk diagnostics.
-   Memory diagnostics.
-   Recent PM2 logs in alert emails.
-   Log rotation.
-   PM2 startup persistence.

The key configuration is:

``` bash
SERVERS=(
    "server-1|pm2-app-1|http://127.0.0.1:3000/health"
    "server-2|pm2-app-2|http://127.0.0.1:4000/health"
    "server-3|pm2-app-3|http://127.0.0.1:5000/health"
)
```

Add or remove applications from this array without changing the
monitoring script.
