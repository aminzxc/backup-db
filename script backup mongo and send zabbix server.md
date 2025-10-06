### script
```
#!/usr/bin/env bash
set -euo pipefail

############################
# setting
############################

MONGO_CONTAINER="db1"
BACKUP_DIR="/var/backups/mongodb"
RETENTION_DAYS=3

REMOTE_USER="root"
REMOTE_HOST="IP"
REMOTE_DIR="/home/backups/mongo"

SSH_KEY="${HOME}/.ssh/backup_rsync_ed25519"

LOG_FILE="/var/log/backup_mongo.log"

ZBX_HOST="HOST NAME"
ZBX_SERVER="IP"
ZBX_PORT="10051"
ZBX_KEY="backup.report[mongodb]"

JOB_NAME="mongodb"
START_TS="$(date +%s)"
STATUS=1
MESSAGE="ok"

send_zbx() {
  local now dur size sha ts payload
  now=$(date +%s)
  dur=$(( now - START_TS ))
  ts=$(date -Is)

  if [[ -f "${ARCHIVE_PATH:-/nonexistent}" ]]; then
    size=$(stat -c%s "$ARCHIVE_PATH" 2>/dev/null || echo 0)
    sha=$(sha256sum "$ARCHIVE_PATH" 2>/dev/null | awk '{print $1}')
  else
    size=0
    sha=""
  fi

  payload=$(printf '{"job":"%s","status":%s,"size":%s,"duration":%s,"sha256":"%s","message":"%s","timestamp":"%s"}' \
                  "$JOB_NAME" "$STATUS" "$size" "$dur" "$sha" "$MESSAGE" "$ts")

  zabbix_sender -z "$ZBX_SERVER" -p "$ZBX_PORT" \
    -s "$ZBX_HOST" -k "$ZBX_KEY" -o "$payload" >/dev/null 2>&1 || true
}

############################
# start
############################
mkdir -p "$BACKUP_DIR"
touch "$LOG_FILE"

TS="$(date +'%Y-%m-%d_%H-%M-%S')"
ARCHIVE_NAME="mongo-backup_${TS}.archive.gz"
ARCHIVE_PATH="${BACKUP_DIR}/${ARCHIVE_NAME}"

echo "[$(date '+%F %T')] Backup started..." | tee -a "$LOG_FILE"

docker exec "$MONGO_CONTAINER" \
  mongodump --archive --gzip --oplog > "$ARCHIVE_PATH"

if [ ! -s "$ARCHIVE_PATH" ]; then
  echo "[$(date '+%F %T')] ERROR: Backup file is empty or missing!" | tee -a "$LOG_FILE"
  STATUS=0; MESSAGE="empty backup file"
  send_zbx
  exit 1
fi

MD5_LOCAL="$(md5sum "$ARCHIVE_PATH" | awk '{print $1}')"
echo "[$(date '+%F %T')] Local backup created: ${ARCHIVE_PATH} (md5: ${MD5_LOCAL})" | tee -a "$LOG_FILE"

rsync -az --progress -e "ssh -i ${SSH_KEY} -o StrictHostKeyChecking=accept-new" \
  "$ARCHIVE_PATH" "${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_DIR}/"

if ssh -i "$SSH_KEY" -o StrictHostKeyChecking=accept-new "${REMOTE_USER}@${REMOTE_HOST}" "command -v md5sum >/dev/null 2>&1"; then
  MD5_REMOTE="$(ssh -i "$SSH_KEY" -o StrictHostKeyChecking=accept-new \
    "${REMOTE_USER}@${REMOTE_HOST}" \
    "md5sum '${REMOTE_DIR}/${ARCHIVE_NAME}' | awk '{print \$1}'" )"

  if [ -z "${MD5_REMOTE:-}" ]; then
    echo "[$(date '+%F %T')] ERROR: Remote MD5 is empty (file missing or md5sum failed)." | tee -a "$LOG_FILE"
    STATUS=0; MESSAGE="remote md5 empty"
    send_zbx
    exit 1
  elif [ "$MD5_LOCAL" = "$MD5_REMOTE" ]; then
    echo "[$(date '+%F %T')] OK: MD5 verified: $MD5_LOCAL" | tee -a "$LOG_FILE"
  else
    echo "[$(date '+%F %T')] ERROR: MD5 mismatch! local=$MD5_LOCAL remote=$MD5_REMOTE" | tee -a "$LOG_FILE"
    STATUS=0; MESSAGE="md5 mismatch"
    send_zbx
    exit 1
  fi
else
  echo "[$(date '+%F %T')] WARN: md5sum not found on remote; skipping hash verification." | tee -a "$LOG_FILE"
fi

echo "[$(date '+%F %T')] Remote transfer finished to ${REMOTE_HOST}:${REMOTE_DIR}/${ARCHIVE_NAME}" | tee -a "$LOG_FILE"

find "$BACKUP_DIR" -type f -name 'mongo-backup_*.archive.gz' -mtime +"$RETENTION_DAYS" -print -delete | tee -a "$LOG_FILE" || true

echo "[$(date '+%F %T')] Backup completed successfully." | tee -a "$LOG_FILE"

STATUS=1; MESSAGE="ok"
send_zbx

```
