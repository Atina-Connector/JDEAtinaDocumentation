# JFrog Maven Monitor Guide

## Overview

This guide documents the full setup for `/usr/local/bin/jfrog_maven_monitor.sh`, including:

- external Maven synthetic monitoring from a second droplet
- Slack notifications
- SSH passwordless access to the JFrog server
- optional remote restart of JFrog when the check fails
- cron scheduling
- troubleshooting notes

---

## Goal

Run a real Maven download from another droplet against:

- repository: `https://jfrog.atina-connection.com/artifactory/libs-release`
- artifact: `com.atina:JDEAtinaServer:1.0.1`

and:

1. detect failures
2. notify Slack
3. optionally restart JFrog remotely over SSH
4. wait
5. retry the Maven check
6. notify Slack again if it recovers or if it still fails

---

## Architecture

### Monitor droplet
Runs:

- Maven
- the monitor script
- cron
- Slack webhook notifications
- SSH client

### JFrog droplet
Runs:

- Artifactory OSS 5.0.1
- restart commands through SSH

---

## Current script location

```bash
/usr/local/bin/jfrog_maven_monitor.sh
```

---

## Slack configuration

Create:

```bash
/etc/jfrog-monitor.conf
```

Contents:

```bash
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/XXXXX/XXXXX/XXXXX"
```

Protect it:

```bash
chmod 600 /etc/jfrog-monitor.conf
```

---

## SSH passwordless access

### 1. Create a dedicated key without passphrase on the monitor droplet

```bash
ssh-keygen -t ed25519 -f /root/.ssh/jfrog_monitor_nopass -C "jfrog-monitor-nopass"
```

When prompted for passphrase, press **Enter** twice so it stays empty.

### 2. Copy the public key to the JFrog server

```bash
ssh-copy-id -i /root/.ssh/jfrog_monitor_nopass.pub root@157.245.236.175
```

### 3. Test direct login

```bash
ssh -i /root/.ssh/jfrog_monitor_nopass root@157.245.236.175
```

It should connect without password and without passphrase.

### 4. Configure SSH alias

Create or edit:

```bash
/root/.ssh/config
```

Add:

```sshconfig
Host jfrog-server
    HostName 157.245.236.175
    User root
    IdentityFile /root/.ssh/jfrog_monitor_nopass
    IdentitiesOnly yes
```

Set permissions:

```bash
chmod 700 /root/.ssh
chmod 600 /root/.ssh/config
chmod 600 /root/.ssh/jfrog_monitor_nopass
chmod 644 /root/.ssh/jfrog_monitor_nopass.pub
```

### 5. Validate alias

```bash
ssh jfrog-server
```

---

## JFrog restart commands

Current desired restart flow on the JFrog server:

```bash
cd /home/jfrog/artifactory-oss-5.0.1/bin
sudo pkill -f java
sudo pkill -f artifactory
sudo ./artifactory.sh start
```

### Important note

`pkill -f java` is broad and will kill **all** Java processes on that host.  
Only use it if that droplet is dedicated to JFrog/Artifactory.

---

## Recommended remote restart test

From the monitor droplet, test manually:

```bash
ssh jfrog-server 'cd /home/jfrog/artifactory-oss-5.0.1/bin && pkill -f java || true && pkill -f artifactory || true && ./artifactory.sh start'
```

If this works manually, the monitor script should also be able to restart JFrog.

---

## Recommended monitor script

Save as:

```bash
/usr/local/bin/jfrog_maven_monitor.sh
```

Contents:

```bash
#!/usr/bin/env bash
set -u

CONFIG_FILE="/etc/jfrog-monitor.conf"
LOG_FILE="/var/log/jfrog-maven-monitor.log"
STATE_DIR="/var/lib/jfrog-monitor"
STATE_FILE="${STATE_DIR}/status.txt"
WORK_DIR="/tmp/jfrog-maven-monitor"

ARTIFACT_COORD="com.atina:JDEAtinaServer:1.0.1"
ARTIFACT_GROUP_PATH="com/atina/JDEAtinaServer/1.0.1"
ARTIFACT_JAR="JDEAtinaServer-1.0.1.jar"
ARTIFACT_POM="JDEAtinaServer-1.0.1.pom"
REPO_URL="https://jfrog.atina-connection.com/artifactory/libs-release"
DEST_FILE="${WORK_DIR}/${ARTIFACT_JAR}"
M2_LOCAL="${WORK_DIR}/m2"

JFROG_SSH_HOST="jfrog-server"
JFROG_BIN_DIR="/home/jfrog/artifactory-oss-5.0.1/bin"
RESTART_WAIT_SECONDS=45
SSH_OPTS="-o BatchMode=yes -o ConnectTimeout=20"

timestamp() { date '+%Y-%m-%d %H:%M:%S'; }
log() { echo "[$(timestamp)] $*" | tee -a "$LOG_FILE"; }

mkdir -p "$STATE_DIR" "$WORK_DIR"
touch "$LOG_FILE"

if [[ -f "$CONFIG_FILE" ]]; then
  # shellcheck disable=SC1090
  source "$CONFIG_FILE"
fi

notify_slack() {
  local msg="$1"

  if [[ -z "${SLACK_WEBHOOK_URL:-}" ]]; then
    log "WARN: SLACK_WEBHOOK_URL is not defined. Slack notification skipped."
    return 0
  fi

  local response
  response=$(curl -sS -w " HTTP_STATUS:%{http_code}" \
    -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"${msg//\"/\\\"}\"}" \
    "$SLACK_WEBHOOK_URL" 2>&1)

  local http_status
  http_status=$(echo "$response" | sed -n 's/.*HTTP_STATUS:\([0-9][0-9][0-9]\).*/\1/p')

  if [[ "$http_status" == "200" ]]; then
    log "Slack notification sent successfully."
  else
    log "ERROR sending Slack notification. Response: $response"
  fi
}

get_previous_status() {
  if [[ -f "$STATE_FILE" ]]; then
    cat "$STATE_FILE"
  else
    echo "UNKNOWN"
  fi
}

set_status() {
  echo "$1" > "$STATE_FILE"
}

restart_jfrog_remote() {
  log "Attempting remote restart on JFrog host ${JFROG_SSH_HOST}..."

  local restart_output
  restart_output=$(ssh ${SSH_OPTS} "${JFROG_SSH_HOST}" "
    cd ${JFROG_BIN_DIR} && \
    pkill -f java || true && \
    pkill -f artifactory || true && \
    ./artifactory.sh start
  " 2>&1)

  local rc=$?
  log "Remote restart output:"
  log "${restart_output}"

  if [[ $rc -ne 0 ]]; then
    log "ERROR: Remote restart command failed with rc=${rc}"
    return 1
  fi

  log "Remote restart command completed."
  return 0
}

run_maven_check() {
  rm -rf "$M2_LOCAL" "$DEST_FILE"
  mkdir -p "$M2_LOCAL" "$WORK_DIR"

  POM_URL="${REPO_URL}/${ARTIFACT_GROUP_PATH}/${ARTIFACT_POM}"
  HTTP_CHECK_OUTPUT=$(curl -k -I -sS --max-time 20 "$POM_URL" 2>&1)
  HTTP_CHECK_RC=$?

  MVN_OUTPUT_FILE="${WORK_DIR}/mvn-output.log"

  timeout 120 mvn -B -U org.apache.maven.plugins:maven-dependency-plugin:2.4:get \
    -DremoteRepositories=temp::default::${REPO_URL} \
    -Dartifact=${ARTIFACT_COORD} \
    -Ddest=${DEST_FILE} \
    -Dmaven.repo.local=${M2_LOCAL} \
    >"$MVN_OUTPUT_FILE" 2>&1

  MVN_RC=$?

  CURRENT_STATUS="OK"
  DETAIL="Artifact download succeeded."

  if [[ $HTTP_CHECK_RC -ne 0 ]]; then
    CURRENT_STATUS="FAIL"
    DETAIL="Initial HTTPS connectivity check failed for ${POM_URL}."
  elif [[ $MVN_RC -ne 0 ]]; then
    CURRENT_STATUS="FAIL"
    DETAIL="Maven dependency:get failed."
  elif [[ ! -f "$DEST_FILE" ]]; then
    CURRENT_STATUS="FAIL"
    DETAIL="Maven finished but destination JAR was not created."
  fi

  if [[ "$CURRENT_STATUS" == "FAIL" && -f "$MVN_OUTPUT_FILE" ]]; then
    if grep -qi "Connection refused" "$MVN_OUTPUT_FILE"; then
      DETAIL="Connection refused by JFrog endpoint."
    elif grep -qi "Read timed out" "$MVN_OUTPUT_FILE"; then
      DETAIL="Connection to JFrog timed out."
    elif grep -qi "status code: 401" "$MVN_OUTPUT_FILE"; then
      DETAIL="JFrog returned HTTP 401 Unauthorized."
    elif grep -qi "status code: 403" "$MVN_OUTPUT_FILE"; then
      DETAIL="JFrog returned HTTP 403 Forbidden."
    elif grep -qi "status code: 404" "$MVN_OUTPUT_FILE"; then
      DETAIL="JFrog returned HTTP 404 Not Found."
    elif grep -qi "status code: 500" "$MVN_OUTPUT_FILE"; then
      DETAIL="JFrog returned HTTP 500 Internal Server Error."
    elif grep -qi "status code: 502" "$MVN_OUTPUT_FILE"; then
      DETAIL="JFrog returned HTTP 502 Bad Gateway."
    elif grep -qi "status code: 503" "$MVN_OUTPUT_FILE"; then
      DETAIL="JFrog returned HTTP 503 Service Unavailable."
    fi
  fi
}

PREVIOUS_STATUS=$(get_previous_status)
HOSTNAME_VALUE=$(hostname)

log "Starting JFrog Maven synthetic check..."
run_maven_check

if [[ "$CURRENT_STATUS" == "OK" ]]; then
  log "SUCCESS: ${DETAIL}"

  if [[ "$PREVIOUS_STATUS" != "OK" ]]; then
    notify_slack ":large_green_circle: JFrog RECOVERED on ${HOSTNAME_VALUE}\nArtifact: ${ARTIFACT_COORD}\nRepository: ${REPO_URL}\nResult: ${DETAIL}"
  fi

  set_status "OK"
else
  log "FAILURE: ${DETAIL}"
  log "HTTP check output:"
  log "$HTTP_CHECK_OUTPUT"
  log "Maven output:"
  cat "$MVN_OUTPUT_FILE" | tee -a "$LOG_FILE"

  notify_slack ":red_circle: JFrog Maven check FAILED on ${HOSTNAME_VALUE}\nArtifact: ${ARTIFACT_COORD}\nRepository: ${REPO_URL}\nReason: ${DETAIL}\nAttempting remote restart on JFrog..."

  if restart_jfrog_remote; then
    log "Waiting ${RESTART_WAIT_SECONDS} seconds before retry..."
    sleep "${RESTART_WAIT_SECONDS}"

    log "Retrying Maven check after remote restart..."
    run_maven_check

    if [[ "$CURRENT_STATUS" == "OK" ]]; then
      log "SUCCESS AFTER RESTART: ${DETAIL}"
      notify_slack ":large_green_circle: JFrog RECOVERED after remote restart on ${HOSTNAME_VALUE}\nArtifact: ${ARTIFACT_COORD}\nRepository: ${REPO_URL}\nResult: ${DETAIL}"
      set_status "OK"
    else
      log "FAILURE AFTER RESTART: ${DETAIL}"
      log "HTTP check output after restart:"
      log "$HTTP_CHECK_OUTPUT"
      log "Maven output after restart:"
      cat "$MVN_OUTPUT_FILE" | tee -a "$LOG_FILE"

      notify_slack ":red_circle: JFrog is still failing after remote restart on ${HOSTNAME_VALUE}\nArtifact: ${ARTIFACT_COORD}\nRepository: ${REPO_URL}\nReason: ${DETAIL}\nLog file: ${LOG_FILE}"
      set_status "FAIL"
    fi
  else
    notify_slack ":red_circle: Remote restart command FAILED on ${HOSTNAME_VALUE}\nJFrog host: ${JFROG_SSH_HOST}\nPlease check SSH access / restart permissions.\nLog file: ${LOG_FILE}"
    set_status "FAIL"
  fi
fi

log "Check finished with status: ${CURRENT_STATUS}"
exit 0
```

Make it executable:

```bash
chmod +x /usr/local/bin/jfrog_maven_monitor.sh
```

---

## Cron configuration

Recommended cron entry:

```cron
0 8,16 * * * /usr/local/bin/jfrog_maven_monitor.sh >> /var/log/jfrog-maven-monitor-cron.log 2>&1
```

Edit root crontab:

```bash
crontab -e
```

Verify:

```bash
crontab -l
```

---

## Logs and state

### Main log

```bash
/var/log/jfrog-maven-monitor.log
```

### Cron wrapper log

```bash
/var/log/jfrog-maven-monitor-cron.log
```

### Current status file

```bash
/var/lib/jfrog-monitor/status.txt
```

Possible values:

- `OK`
- `FAIL`

---

## How the script works

### If Maven succeeds
- logs success
- sets state to `OK`
- sends recovery Slack only if previous state was not `OK`

### If Maven fails
- logs the failure
- sends Slack alert
- attempts remote restart over SSH
- waits
- retries Maven download

### If retry succeeds
- sends recovery Slack
- stores `OK`

### If retry still fails
- sends final failure Slack
- stores `FAIL`

---

## Current observed failure

From the terminal output, the Maven check correctly detected:

```text
Connect to jfrog.atina-connection.com:443 ... failed: Connection refused
```

Then the script tried the remote restart and got:

```text
ERROR: Remote restart command failed with rc=255
```

### Meaning of `rc=255`
For `ssh`, exit code `255` usually means:

- SSH connectivity problem
- alias/config not available to the process
- host key issue
- authentication problem
- remote command execution problem
- remote shell startup issue

---

## Troubleshooting remote restart failure

### 1. Test non-interactive SSH exactly like the script

Run this from the monitor droplet:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=20 jfrog-server 'hostname'
```

If this fails, the script will also fail.

### 2. Test the remote restart command directly

```bash
ssh -o BatchMode=yes -o ConnectTimeout=20 jfrog-server 'cd /home/jfrog/artifactory-oss-5.0.1/bin && pkill -f java || true && pkill -f artifactory || true && ./artifactory.sh start'
```

### 3. If it still fails, run verbose SSH

```bash
ssh -vvv -o BatchMode=yes -o ConnectTimeout=20 jfrog-server 'hostname'
```

### 4. Confirm root can log in by SSH using the alias

```bash
ssh jfrog-server
```

### 5. Confirm the script runs as the same user that owns `/root/.ssh/config`

Check:

```bash
whoami
```

If the script runs from root's cron, it should use `/root/.ssh/config`.

---

## Strong recommendation: use a remote restart script on the JFrog server

Instead of executing long inline restart commands via SSH, create a local script on the JFrog server.

### On the JFrog server create:

```bash
/usr/local/bin/restart_jfrog_artifactory.sh
```

Contents:

```bash
#!/usr/bin/env bash
cd /home/jfrog/artifactory-oss-5.0.1/bin || exit 1
pkill -f java || true
pkill -f artifactory || true
./artifactory.sh start
```

Make it executable:

```bash
chmod +x /usr/local/bin/restart_jfrog_artifactory.sh
```

### Then change the monitor script function to:

```bash
restart_jfrog_remote() {
  log "Attempting remote restart on JFrog host ${JFROG_SSH_HOST}..."

  local restart_output
  restart_output=$(ssh ${SSH_OPTS} "${JFROG_SSH_HOST}" "/usr/local/bin/restart_jfrog_artifactory.sh" 2>&1)

  local rc=$?
  log "Remote restart output:"
  log "${restart_output}"

  if [[ $rc -ne 0 ]]; then
    log "ERROR: Remote restart command failed with rc=${rc}"
    return 1
  fi

  log "Remote restart command completed."
  return 0
}
```

This is cleaner, easier to test, and easier to maintain.

---

## Manual validation checklist

### Validate Maven path
```bash
/usr/local/bin/jfrog_maven_monitor.sh
```

### Validate logs
```bash
tail -100 /var/log/jfrog-maven-monitor.log
```

### Validate SSH alias
```bash
ssh jfrog-server
```

### Validate non-interactive SSH
```bash
ssh -o BatchMode=yes jfrog-server 'hostname'
```

### Validate remote restart
```bash
ssh -o BatchMode=yes jfrog-server '/usr/local/bin/restart_jfrog_artifactory.sh'
```

---

## Related JFrog server context

The underlying JFrog server issue previously diagnosed was memory exhaustion:

- Linux OOM killer terminated the Java process
- this caused Maven downloads to fail
- Artifactory OSS 5.0.1 is old
- server resources are constrained
- Derby is not ideal for production use

Because of that, this monitor is useful for:

- early detection
- remote automatic recovery attempt
- Slack alerting

but it does **not** replace the root fixes:

- increase RAM
- add swap
- tune Java heap
- move away from Derby
- upgrade Artifactory

---

## Immediate root cause statement in English

You can use this short diagnosis in incident reports:

> The issue was caused by the Linux Out Of Memory (OOM) killer terminating the Artifactory Java process.  
> When memory was exhausted on the JFrog droplet, the kernel killed the `java` process hosting Artifactory, which caused Maven artifact downloads to fail until the service was restarted.

---

## Immediate corrective actions

### Stage 1
- add swap
- tune JVM heap (`-Xmx`)
- keep the external synthetic monitor active
- enable Slack alerts
- enable optional remote restart

### Stage 2
- migrate from Derby to PostgreSQL
- increase droplet resources
- plan upgrade from Artifactory OSS 5.0.1

---

## Current JFrog cron jobs

Current entries reported:

```cron
0 18 * * * /usr/local/bin/jfrog_watchdog.sh >> /var/log/jfrog-watchdog.log 2>&1
0 18 * * * /usr/local/bin/jfrog-downloads-watch.sh >> /var/log/jfrog-cron.log 2>&1
```

The second one should be corrected to:

```cron
1 18 * * * /usr/local/bin/jfrog_downloads_notifier.sh >> /var/log/jfrog-cron.log 2>&1
```

Recommended final version:

```cron
0 18 * * * /usr/local/bin/jfrog_watchdog.sh >> /var/log/jfrog-watchdog.log 2>&1
1 18 * * * /usr/local/bin/jfrog_downloads_notifier.sh >> /var/log/jfrog-cron.log 2>&1
```

---

## Final recommendation

For the monitor script, the best next step is:

1. keep the current Maven synthetic check
2. switch remote restart to a dedicated script on the JFrog server
3. test non-interactive SSH using `BatchMode=yes`
4. keep Slack alerts only on state change and restart attempts

That gives you a simple but effective external watchdog for JFrog availability.
