#!/usr/bin/env bash
#
# Splunk Replay Assist
# --------------------
# Interactively builds a batch script that replays (backfills) a saved search
# into its summary index using Splunk's fill_summary_index.py, split into
# fixed-size et/lt windows so each invocation stays within buffer limits.
#
# Requirements: bash 4+, GNU coreutils `date` (standard on any Linux Splunk host).
# No other dependencies.

set -euo pipefail

SPLUNK_HOME="${SPLUNK_HOME:-/opt/splunk}"
FSI="${SPLUNK_HOME}/bin/fill_summary_index.py"

# ---------------------------------------------------------------- helpers ---

die() { printf 'ERROR: %s\n' "$*" >&2; exit 1; }

# epoch -> "YYYY-MM-DD HH:MM:SS UTC  (local: YYYY-MM-DD HH:MM:SS TZ)"
human() {
  local u l
  u=$(date -u -d "@$1" '+%Y-%m-%d %H:%M:%S UTC')
  l=$(date    -d "@$1" '+%Y-%m-%d %H:%M:%S %Z')
  printf '%s  (local: %s)' "$u" "$l"
}

# Accepts a raw epoch, or anything GNU `date -d` understands
# (e.g. "2026-04-08 00:00 UTC", "2026-10-01", "last monday").
to_epoch() {
  local in="$1"
  if [[ "$in" =~ ^[0-9]+$ ]]; then
    printf '%s' "$in"
  else
    date -d "$in" '+%s' 2>/dev/null || return 1
  fi
}

# "21600", "360m", "6h", "1d", "1w" -> seconds
duration_secs() {
  local in="${1,,}"
  [[ "$in" =~ ^([0-9]+)([smhdw]?)$ ]] || return 1
  local n="${BASH_REMATCH[1]}" u="${BASH_REMATCH[2]}"
  case "$u" in
    ""|s) printf '%s' "$n" ;;
    m)    printf '%s' $(( n * 60 )) ;;
    h)    printf '%s' $(( n * 3600 )) ;;
    d)    printf '%s' $(( n * 86400 )) ;;
    w)    printf '%s' $(( n * 604800 )) ;;
  esac
}

pretty_duration() {
  local s=$1 d h m
  d=$(( s / 86400 )); s=$(( s % 86400 ))
  h=$(( s / 3600 ));  s=$(( s % 3600 ))
  m=$(( s / 60 ));    s=$(( s % 60 ))
  printf '%dd %02dh %02dm %02ds' "$d" "$h" "$m" "$s"
}

# ask VAR "Prompt" [default]
ask() {
  local __var=$1 __prompt=$2 __def=${3-} __ans
  if [[ -n "$__def" ]]; then
    read -rp "$__prompt [$__def]: " __ans
    __ans=${__ans:-$__def}
  else
    read -rp "$__prompt: " __ans
  fi
  printf -v "$__var" '%s' "$__ans"
}

# ask_epoch VAR "Label"
ask_epoch() {
  local __var=$1 __label=$2 __raw __ep __ok
  while true; do
    ask __raw "$__label (epoch, or date e.g. '2026-04-08 00:00 UTC')"
    [[ -z "$__raw" ]] && continue
    if __ep=$(to_epoch "$__raw"); then
      printf '  -> %s = %s\n' "$__ep" "$(human "$__ep")"
      read -rp "  Correct? [Y/n]: " __ok
      [[ "${__ok,,}" == n* ]] && continue
      printf -v "$__var" '%s' "$__ep"
      return 0
    fi
    echo "  Could not parse '$__raw'."
  done
}

choose_range() {
  while true; do
    ask_epoch ET "Start time / earliest (et)"
    ask_epoch LT "End time / latest (lt)"
    (( LT > ET )) && return 0
    echo "  End time must be after start time. Try again."
  done
}

choose_window() {
  local raw s
  while true; do
    ask raw "Window size per run (seconds, or with unit s/m/h/d/w, e.g. 6h)" "${WIN:-6h}"
    if s=$(duration_secs "$raw") && (( s > 0 )); then
      WIN=$s
      printf '  -> %s seconds (%s)\n' "$WIN" "$(pretty_duration "$WIN")"
      return 0
    fi
    echo "  Invalid window '$raw'."
  done
}

# ------------------------------------------------------------------- main ---

cat <<'BANNER'
==============================================================
  Splunk Replay Assist
  Generates a windowed fill_summary_index.py replay script
==============================================================
BANNER

[[ -f "$FSI" ]] || echo "WARNING: $FSI not found on this host (script will still be generated)."

ask APP   "Splunk app context (-app)"                        "search"
ask NAME  "Saved search name (-name)"                        "data_replay"
ask JOBS  "Concurrent searches per run (-j)"                 "2"
[[ "$JOBS" =~ ^[1-9][0-9]*$ ]] || die "-j must be a positive integer."
ask DEDUP "Skip intervals already in summary index (-dedup)" "true"
[[ "$DEDUP" == true || "$DEDUP" == false ]] || die "-dedup must be true or false."

echo
choose_range
echo
choose_window

while true; do
  RANGE=$(( LT - ET ))
  TOTAL=$(( (RANGE + WIN - 1) / WIN ))
  LAST=$(( RANGE - (TOTAL - 1) * WIN ))

  echo
  echo "------------------------------ Summary -----------------------------"
  printf '  Saved search : %s  (app: %s, -j %s, -dedup %s)\n' "$NAME" "$APP" "$JOBS" "$DEDUP"
  printf '  Earliest (et): %s = %s\n' "$ET" "$(human "$ET")"
  printf '  Latest   (lt): %s = %s\n' "$LT" "$(human "$LT")"
  printf '  Total range  : %s seconds (%s)\n' "$RANGE" "$(pretty_duration "$RANGE")"
  printf '  Window size  : %s seconds (%s)\n' "$WIN" "$(pretty_duration "$WIN")"
  if (( LAST != WIN )); then
    printf '  Final window : %s seconds (%s) - truncated to lt\n' "$LAST" "$(pretty_duration "$LAST")"
  fi
  echo "--------------------------------------------------------------------"
  printf '  This will create %s individual captures.\n' "$(printf "%'d" "$TOTAL" 2>/dev/null || echo "$TOTAL")"
  (( TOTAL > 5000 )) && echo "  NOTE: that is a lot of runs - consider a larger window."
  echo
  read -rp "[P]roceed, adjust [W]indow, adjust [D]ates, [Q]uit: " choice
  case "${choice,,}" in
    p*) break ;;
    w*) choose_window ;;
    d*) choose_range ;;
    q*) echo "Aborted."; exit 0 ;;
    *)  echo "  Please choose P, W, D or Q." ;;
  esac
done

SAFE_NAME=${NAME//[^A-Za-z0-9_-]/_}
ask OUT "Output file" "./replay_${SAFE_NAME}_${ET}_${LT}.sh"
if [[ -e "$OUT" ]]; then
  read -rp "  $OUT exists. Overwrite? [y/N]: " ow
  [[ "${ow,,}" == y* ]] || die "Not overwriting $OUT."
fi

# ------------------------------------------------------------- generate ---

{
  # Header (comments only)
  cat <<EOF
#!/usr/bin/env bash
#
# Generated by Splunk Replay Assist on $(date -u '+%Y-%m-%d %H:%M:%S UTC')
#
# Saved search : ${NAME}  (app: ${APP})
# Earliest (et): ${ET} = $(human "$ET")
# Latest   (lt): ${LT} = $(human "$LT")
# Window size  : ${WIN} s ($(pretty_duration "$WIN"))
# Windows      : ${TOTAL}
#
# Usage:
#   ./$(basename "$OUT")                      run all windows
#   START_AT=417 ./$(basename "$OUT")         resume from window 417
#   STOP_ON_ERROR=false ./$(basename "$OUT")  log failures and carry on
#   LOG=/path/to/file.log ./$(basename "$OUT")
#
EOF

  # Baked-in configuration (safely quoted)
  printf 'SPLUNK_HOME="${SPLUNK_HOME:-%s}"\n' "$SPLUNK_HOME"
  printf 'APP=%q\n'   "$APP"
  printf 'NAME=%q\n'  "$NAME"
  printf 'JOBS=%q\n'  "$JOBS"
  printf 'DEDUP=%q\n' "$DEDUP"
  printf 'TOTAL=%d\n' "$TOTAL"

  # Runtime body (literal, no expansion at generation time)
  cat <<'EOF'

set -u
: "${START_AT:=1}"
: "${STOP_ON_ERROR:=true}"
: "${LOG:=./splunk_replay_$(date +%Y%m%d_%H%M%S).log}"

SPLUNK_BIN="${SPLUNK_HOME}/bin/splunk"
FSI="${SPLUNK_HOME}/bin/fill_summary_index.py"
[[ -x "$SPLUNK_BIN" ]] || { echo "splunk binary not found at $SPLUNK_BIN" >&2; exit 1; }
[[ -f "$FSI" ]]        || { echo "fill_summary_index.py not found at $FSI" >&2; exit 1; }

read -rp  "Splunk username: " SPLUNK_USER
read -rsp "Splunk password: " SPLUNK_PASS; echo

FAILED=0
echo "Replay started $(date '+%F %T'), $TOTAL windows, starting at $START_AT. Log: $LOG" | tee -a "$LOG"

run() {
  local i=$1 et=$2 lt=$3 rc
  (( i < START_AT )) && return 0
  printf '[%s] window %d/%d  et=%s lt=%s  (%s -> %s)\n' \
    "$(date '+%F %T')" "$i" "$TOTAL" "$et" "$lt" \
    "$(date -u -d "@$et" '+%F %T')" "$(date -u -d "@$lt" '+%F %T UTC')" | tee -a "$LOG"
  if "$SPLUNK_BIN" cmd python "$FSI" \
        -app "$APP" -name "$NAME" \
        -et "$et" -lt "$lt" \
        -j "$JOBS" -dedup "$DEDUP" -showprogress true \
        -auth "${SPLUNK_USER}:${SPLUNK_PASS}" >>"$LOG" 2>&1; then
    :
  else
    rc=$?
    echo "  !! window $i failed (rc=$rc). Resume with: START_AT=$i $0" | tee -a "$LOG"
    FAILED=$(( FAILED + 1 ))
    [[ "$STOP_ON_ERROR" == true ]] && exit "$rc"
  fi
}

EOF

  # One line per window
  i=1; s=$ET
  while (( s < LT )); do
    e=$(( s + WIN ))
    (( e > LT )) && e=$LT
    printf 'run %d %d %d\n' "$i" "$s" "$e"
    s=$e
    i=$(( i + 1 ))
  done

  cat <<'EOF'

echo "Replay finished $(date '+%F %T'). Failed windows: $FAILED. Log: $LOG" | tee -a "$LOG"
EOF
} > "$OUT"

chmod +x "$OUT"
echo
echo "Written: $OUT  ($TOTAL windows)"
echo "Review it, then run:  $OUT"
