Normal Git Deployment Workflow
On Windows

From PowerShell:

cd $env:USERPROFILE\Desktop\ATC_PORTAL_SERVER
git status
git add .
git commit -m "Describe the change"
git push
On VPS
~/bin/update-demo-portal-from-git.sh

This performs:

Checks the VPS staging repo.
Ensures working tree is clean.
Fetches from GitHub.
Pulls origin/main with fast-forward only.
Verifies index.html exists.
Runs deploy dry-run.
Runs live deploy.
Creates pre-deploy backup.
Fixes permissions.
Runs HTTP health check.
Cleans old timestamped backups.
Manual Deploy From Staging

Use only if staging is already correct.

~/bin/deploy-demo-portal.sh dry
~/bin/deploy-demo-portal.sh live

Important warnings:

If dry-run shows `.git/`, stop.
If dry-run shows `deleting index.html`, stop.
If health check fails, do not continue blindly.
Rollback

List available timestamped backups:

ls -1t ~/backups/demo-portal-[0-9]*.tar.gz

Rollback to selected backup:

~/bin/rollback-demo-portal.sh /home/alex/backups/BACKUP_FILE.tar.gz

After rollback, confirm health:

curl -I -H "Host: demo-portal.test" http://127.0.0.1

Expected:

HTTP/1.1 200 OK

Important:

Rollback fixes the live site.
Git revert fixes the source of truth.

If a bad Git commit was deployed, rollback first, then revert/fix from Windows and push.

Nginx Checks

Test config:

sudo nginx -t

Reload config:

sudo systemctl reload nginx

Check demo portal:

curl -I -H "Host: demo-portal.test" http://127.0.0.1

Check static assets:

curl -I -H "Host: demo-portal.test" http://127.0.0.1/logo.png
curl -I -H "Host: demo-portal.test" http://127.0.0.1/favicon.ico

Expected:

HTTP/1.1 200 OK
Nginx Config Locations

Available sites:

ls -la /etc/nginx/sites-available

Enabled sites:

ls -la /etc/nginx/sites-enabled

Demo portal config:

sudo cat /etc/nginx/sites-available/demo-portal

Full effective Nginx config:

sudo nginx -T

Search Nginx config:

sudo grep -Rni "pattern" /etc/nginx
Logs

Nginx access log:

sudo tail -n 50 /var/log/nginx/access.log

Nginx error log:

sudo tail -n 50 /var/log/nginx/error.log

SSH logs:

sudo journalctl -u ssh -n 50 --no-pager

Live Nginx access monitoring:

sudo tail -f /var/log/nginx/access.log

Live Nginx error monitoring:

sudo tail -f /var/log/nginx/error.log

Live SSH monitoring:

sudo journalctl -u ssh -f

Stop live monitoring with:

Ctrl + C
Log Meaning
200 = OK
304 = cached / not modified
403 = forbidden / permission or directory index issue
404 = not found
500+ = server/backend problem

Common Nginx error log meanings:

Permission denied = Nginx cannot read/enter file or folder due to permissions.
directory index forbidden = folder requested, no index.html, directory listing disabled.
No such file or directory = wrong path or missing file.
SSH Security Checks

Effective SSH config:

sudo sshd -T | grep -Ei "port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|usepam|maxauthtries|allowusers"

Expected important values:

permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
pubkeyauthentication yes
allowusers alex
maxauthtries 3

Check users:

cut -d: -f1,3,7 /etc/passwd | awk -F: '$2 >= 1000 {print}'

Check sudo users:

getent group sudo

Check authorized SSH keys:

ls -la ~/.ssh
cat ~/.ssh/authorized_keys

Check successful logins:

last -a | head -20
Firewall / Fail2ban

UFW status:

sudo ufw status numbered

Fail2ban service:

sudo systemctl is-active fail2ban

Fail2ban SSH jail:

sudo fail2ban-client status sshd

Expected:

Status for the jail: sshd
Currently banned: may be 0 or more
Total banned: may increase over time
Maintenance Updates

Safe update process:

~/bin/server-health-check.sh
sudo apt update
apt list --upgradable
sudo apt upgrade
test -f /var/run/reboot-required && cat /var/run/reboot-required || echo "No reboot required"
systemctl is-active nginx
systemctl is-active ssh
systemctl is-active fail2ban
~/bin/server-health-check.sh

If reboot is required:

sudo reboot

After reboot, reconnect from Windows:

ssh vps1

Then run:

~/bin/server-health-check.sh
Disk Hygiene

Check disk usage:

df -h

Check top-level disk usage:

sudo du -xhd1 / | sort -h

Check /var:

sudo du -xhd1 /var | sort -h

Check logs:

sudo du -xhd1 /var/log | sort -h

Check Nginx logs:

sudo ls -lh /var/log/nginx

Check journal size:

sudo journalctl --disk-usage

Check backups:

ls -lh ~/backups

Rules:

Disk above 80% = investigate.
Disk above 90% = urgent action.
Large /var/log = inspect logs.
Large journal = check journald limit.
Backup Management

List backups:

ls -lh ~/backups

List timestamped demo portal backups newest first:

ls -1t ~/backups/demo-portal-[0-9]*.tar.gz

Keep latest 3 timestamped backups and preview old ones:

ls -1t ~/backups/demo-portal-[0-9]*.tar.gz | tail -n +4

Dry-run delete old backups:

ls -1t ~/backups/demo-portal-[0-9]*.tar.gz | tail -n +4 | xargs -r echo rm -v

Delete old backups:

ls -1t ~/backups/demo-portal-[0-9]*.tar.gz | tail -n +4 | xargs -r rm -v

Do not use broad patterns like:

~/backups/demo-portal-*.tar.gz

unless you are sure, because it may catch manual backups like:

demo-portal-backup.tar.gz
Static Asset Checks

Check logo:

curl -I -H "Host: demo-portal.test" http://127.0.0.1/logo.png

Check favicon:

curl -I -H "Host: demo-portal.test" http://127.0.0.1/favicon.ico

Expected for static assets:

HTTP/1.1 200 OK
Expires: ...
Cache-Control: max-age=2592000

Check if old large logo is gone:

ls -lh /var/www/demo-portal/ATC_LOGO_MAIN.png

Expected:

No such file or directory
Current Scripts

List scripts:

ls -lh ~/bin

Syntax check scripts:

bash -n ~/bin/server-health-check.sh
bash -n ~/bin/deploy-demo-portal.sh
bash -n ~/bin/update-demo-portal-from-git.sh
bash -n ~/bin/rollback-demo-portal.sh

Expected: no output.

Scripts:

~/bin/server-health-check.sh
~/bin/deploy-demo-portal.sh
~/bin/update-demo-portal-from-git.sh
~/bin/rollback-demo-portal.sh
Emergency Quick Commands

Health check:

~/bin/server-health-check.sh

Check Nginx:

sudo nginx -t
systemctl is-active nginx
sudo tail -n 50 /var/log/nginx/error.log

Check SSH attacks/logins:

sudo journalctl -u ssh -n 50 --no-pager
sudo fail2ban-client status sshd
last -a | head -20

Check disk:

df -h
sudo du -xhd1 /var | sort -h

Rollback latest timestamped backup:

LATEST_BACKUP="$(ls -1t ~/backups/demo-portal-[0-9]*.tar.gz | head -1)"
echo "$LATEST_BACKUP"
~/bin/rollback-demo-portal.sh "$LATEST_BACKUP"