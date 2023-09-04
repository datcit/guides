# The Definitive Guide to Email Alerts with Proxmox VE 8.0

## Assumptions
I've accomplished this on both `v8.0.4` and `v7.4-16` of Proxmox Virtual
Environment (PVE) also known as Proxmox VE. The example I use below uses the
domain `mydomain.example.com`. This should be replaced with the domain that you
have own and have control of. In this guide we use two email address
`SendingAddress@example.com` and `RecievingAddress@example.com`. If you prefer
it, they can be the same address, but either way, you will need to replace them
with addresses you are the owner of.

## Step 1 - Configure the Recieving Address
If  you have not already installed Proxmox VE and are following this guide as
part of a fresh install, when you get to the
`Administration Password & Email Address` screen, be sure to use a vaild email
address that you own / have control of. If this an exsisting instilation, in the
WebUI left-click `Datacenter` in the lefthand tree-view, and in this view, 
left-click `Options`. From here you should see a field labeled
`Email from address`. Double click this field and replace it with a vaild email
address that you own / have control of.

## Step 2 - Update System & Packakges
Before installing new packages, always ensure that your package index files and 
the system itself are up-to-date. To do this, click the "Refresh" button in the 
Proxmox VE web-ui. Once you see **`TASK OK`**, close the "Task Viewer" modal 
window and then click the "Upgrade" button.

## Step 3 - Install Required Packages
At this point, you should have the "Proxmox Console" window open.
To proceed, install the following packages. Run the following command in the 
"Proxmox Console":
```bash
apt install -y libsasl2-modules mailutils
```
## Step 4 - Generate Fastmail App Password (Sending Address)
Generate a new [app password](https://www.fastmail.help/hc/en-us/articles/360058752854-App-passwords). 
In the **Name** dropdown select `Custom` and in the **`Access`** dropdown select `SMTP`

## Step 5 - Create `sasl_passwd`
Create `sasl_passwd` and update permissions to the file so that owner (root) has
read and write permissions while all others have no access.
```bash
touch /etc/postfix/sasl_passwd && chmod 600 /etc/postfix/sasl_passwd
```

## Step 6 - Configure `sasl_passwd`
Using nano or your faviorite text editor, configure the Postfix SASL authentication file.
```bash
nano /etc/postfix/sasl_passwd
```
Add the following line to the file, replacing `SendingAddress@example.com` and `YourAppPassword` with your values. 
```conf
[smtp.fastmail.com]:587 SendingAddress@example.com:YourAppPassword" > /etc/postfix/sasl_passwd
```
{: file="/etc/postfix/sasl_passwd" }

## Step 7 - Hash `sasl_passwd` 
Create a hash file. If you make changes to `sasl_passwd` in the future you will need to run this again.
```bash
postmap hash:/etc/postfix/sasl_passwd
```

## Step 8 - Reload Postfix
Next we want to reload postfix for it to pickup our changes.
```bash
postfix reload
```

## Step 9 - Send test email

```bash
#echo "Test email from Proxmox: $(hostname)" | mail -s "Test Email from Proxmox" root #Alternate method
echo "Test email from Proxmox: $(hostname)" | /usr/bin/proxmox-mail-forward
```
Shortly (~30 seconds) after running the above command you should recieve an email from something like
"root <configured.email@example.com". In this case the "display name" is `root`. We want to change this
display name to something more 1) Identifable (who/what sent the email) and 2) more "pro".

## Step 10 - Change the email display name.

<details>
<summary>Option A</summary>
Change the finger information for the `root` user. 

### Option A Step 1
Run the command below being shore to
change `DATC.IT Guide` to a name sutiable to your needs.
  
```bash
chfn --full-name "DATC.IT Guide" root
```
</details>

<details>
<summary>Option B</summary>
Use Postfix PCRE to change the root user display name on emails sent from postfix.
  
### Option B Step 1
```bash
apt install -y postfix-pcre
```

### Option B Step 2
Use nano to add `/^From: .*<(.*)>.*$/ REPLACE From: "DATC.IT Guide" <$1>` to
`/etc/postfix/smtp_header_check` being sure to replace `DATC.IT Guide` with your 
desired email display name.
```shell
nano /etc/postfix/smtp_header_check
```
{: .nolineno }

```conf
/^From: .*<(.*)>.*$/ REPLACE From: "DATC.IT Guide" <$1>
```
{: file="/etc/postfix/smtp_header_check" }

### Option B Step 3
Create a hash file. If you make changes to `smtp_header_checks` in the future you will need to run this again.
```bash
postmap hash:/etc/postfix/smtp_header_checks
```

### Option B Step 4
Add the following line to the end of the `/etc/postfix/main.cf` file
```conf
smtp_header_checks = pcre:/etc/postfix/smtp_header_checks
```
{: file="/etc/postfix/main.cf" }

</details>

## Step N
Verify all disks have SMART enabled.
```bash
smartctl --scan | awk '{print $1}' | xargs -I {} sh -c "echo {}; smartctl -i {} | grep 'SMART support is:'"
```

## Step N
Test SMART notifications 

## Step N
Test ZED notifications

## Step N
Mailrise(Apprise)

## Helpful commands
Restart the postfix service:
```bash
systemctl restart postfix.service
```
Logs:
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
