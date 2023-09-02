# The Definitive Guide to Email Alerts with Proxmox VE 8.0

## Assumptions
I've accomplished this on both `v8.0.4` and `v7.4-16` of Proxmox Virtual Environment (PVE) also known as Proxmox VE. 
The example I use below uses the domain `mydomain.example.com`. This should be replaced with the domain that you have 
own and have control of.

## Step 1
Before installing new packages, always ensure that your package index files and 
the system itself are up-to-date. To do this, click the "Refresh" button in the 
Proxmox VE web-ui. Once you see **`TASK OK`**, close the "Task Viewer" modal 
window and then click the "Upgrade" button.

## Step 2
At this point, you should have the "Proxmox Console" window open.
To proceed, install the following packages. Run the following command in the 
"Proxmox Console":
```bash
apt install -y libsasl2-modules mailutils
```
## Step 3
Generate a new [app password](https://www.fastmail.help/hc/en-us/articles/360058752854-App-passwords). 
In the **Name** dropdown select `Custom` and in the **`Access`** dropdown select `SMTP`

## Step 4
Create `sasl_passwd` and update permissions to the file so that owner (root) has read and write permissions while all others have no access.
```bash
touch /etc/postfix/sasl_passwd && chmod 600 /etc/postfix/sasl_passwd
```

## Step 5
Configure postfix
```bash
echo "[smtp.fastmail.com]:587 your-email@example.com:YourAppPassword" > /etc/postfix/sasl_passwd
```

## Step 6
Create a hash file. If you make changes to `sasl_passwd` in the future you will need to run this again.
```bash
postmap hash:/etc/postfix/sasl_passwd
```

## Step N
Reload Postfix
```bash
postfix reload
```

## Step N
Send test emails
```bash
echo "Test email from Proxmox: $(hostname)" | mail -s "Test Email from Proxmox" root
echo "Test email from Proxmox: $(hostname)" | /usr/bin/proxmox-mail-forward
```

## Step N
<details>
<summary>Option A</summary>
Change the finger information for the `root` user.
  
```bash
chfn --full-name "Display Name" root
```
</details>

<details>
<summary>Option B</summary>
Use Postfix PCRE to change the root user display name on emails sent from postfix.
  
### Option B Step N
```bash
apt install -y postfix-pcre
```

### Option B Step N
/etc/postfix/smtp_header_check
```conf
/^From: (.*) (.*)$/ REPLACE From: "Display Name" <$2>
```

### Option B Step N
Create a hash file. If you make changes to `smtp_header_checks` in the future you will need to run this again.
```bash
postmap hash:/etc/postfix/smtp_header_checks
```

### Option B Step N
Add the following line to the end of the `/etc/postfix/main.cf` file
```conf
smtp_header_checks = pcre:/etc/postfix/smtp_header_checks
```
</details>

## Step N
Verify all disks have SMART enabled.
```bash
smartctl --scan | awk '{print $1}' | xargs -I {} sh -c "echo {}; smartctl -i {} | grep 'SMART support is:'"
```

## Helpful commands
```bash
tail -f /var/log/syslog
```
```bash
tail -f /var/log/mail.warn
```
```bash
tail -f /var/log/mail.info
```

## Useful Resources
The following pages I found usefull when learning what I did in order to put together this guide
- [Set up alerts in Proxmox before it's too late!](https://web.archive.org/web/20230901194249/https://technotim.live/posts/proxmox-alerts/)
- [Setting up email notification for ZED](https://web.archive.org/web/20230815011914/https://old.reddit.com/r/Proxmox/comments/15puwzc/setting_up_email_notification_for_zed/)
- [](https://forum.proxmox.com/threads/get-postfix-to-send-notifications-email-externally.59940/)
- [](https://i12bretro.github.io/tutorials/0717.html)
