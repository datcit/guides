# The Definitive Guide to Email Alerts with Proxmox VE 8.0

## Assumptions
I have successfully completed this procedure on both Proxmox Virtual Environment (PVE), versions `v8.0.4` and `v7.4-16`, also referred to as Proxmox VE. In the example provided below, I utilize the domain `mydomain.example.com`. It's essential to substitute this with the domain that you possess and manage. Throughout this guide, I employ two email addresses: `SendingAddress@example.com` and `ReceivingAddress@example.com`. You can choose to use the same address for both, but regardless, you must replace these placeholders with email addresses that you own and control.

## Step 1 - Configuring the Recieving Address
If you haven't installed Proxmox VE yet and are following this guide for a fresh installation, make sure that when you reach the `Administration Password & Email Address` screen, you use a valid email address that you own or have control over. This email address will be used to receive alerts.

If you are working with an existing installation, go to the WebUI and click on `Datacenter` in the left-hand tree-view. In this view, click on `Options`. Look for the `Email from address` field and double-click to edit it. Replace the current email address with a valid one that you own or have control over.

## Step 2 - Updating System & Packakges
Prior to installing new packages, it's crucial to confirm the currency of your package index files and the system. To achieve this, click on the "Refresh" button within the Proxmox VE web UI. Once you observe the status **`TASK OK`**, proceed to close the "Task Viewer" modal window and subsequently select the "Upgrade" button.

## Step 3 - Installing Required Packages
At this point, you should have the "Proxmox Console" window open. To proceed, install the following packages. Run the following command in the "Proxmox Console":
```bash
apt install -y libsasl2-modules mailutils
```
## Step 4 - Generating Fastmail App Password (Sending Address)
Generate a new [app password](https://www.fastmail.help/hc/en-us/articles/360058752854-App-passwords). 
In the **Name** dropdown select `Custom` and in the **`Access`** dropdown select `SMTP`

## Step 5 - Creating `sasl_passwd`
Create `sasl_passwd` and update permissions to the file so that owner (root) has
read and write permissions while all others have no access.
```bash
touch /etc/postfix/sasl_passwd
chmod 600 /etc/postfix/sasl_passwd
```

## Step 6 - Configuring `sasl_passwd`
Utilize your preferred text editor, such as nano, to configure the Postfix SASL authentication file:
```bash
nano /etc/postfix/sasl_passwd
```
Add the following line to the file, replacing `SendingAddress@example.com` and `YourAppPassword` with your specific values: 
```conf
[smtp.fastmail.com]:587 SendingAddress@example.com:YourAppPassword" > /etc/postfix/sasl_passwd
```
{: file="/etc/postfix/sasl_passwd" }

## Step 7 - Hashing `sasl_passwd` 
Generate a hash file for the sasl_passwd using the following command. 
> Note that if you make future changes to `sasl_passwd`, you will need to execute this step again
{: .prompt-info }
```bash
postmap hash:/etc/postfix/sasl_passwd
```

## Step 8 - Reloading Postfix
Continuing on, we need to reload Postfix to ensure that it picks up the changes we've made.
> Note that if you make future changes to `sasl_passwd`, you will need to execute this step again
{: .prompt-info }
```bash
postfix reload
```

## Step 9 - Sending a Test Email
To verify that everything is set up correctly, run the following command in your terminal:
```bash
echo "Test email from Proxmox: $(hostname)" | /usr/bin/proxmox-mail-forward
```
Note: An alternate method to send a test email is:
```bash
#echo "Test email from Proxmox: $(hostname)" | mail -s "Test Email from Proxmox" root
```

## Step 10 - Modifying the Email Display Name.
Shortyly after running the command in the previous step, you should receive an email coming from something like `root <SendingAddress@example.com`. In this case the "display name" is `root`. We want to change this display name to something more Identifable IE who or what sent the email. To complete this, we have two options [complete this]
<details>
<summary>Option A: Modify Root User's Finger Information</summary>
Change the finger information for the `root` user. 

### Option A Step 1
Run the command below being shore to
change `DATC.IT Guide` to a name sutiable to your needs.
  
```bash
chfn --full-name "DATC.IT Guide" root
```
</details>

<details>
<summary>Option B: Use Postfix PCRE to Change the Display Name</summary>
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

## Step 11 - Verifying SMART Activation on All Disks
To ensure that all your disks have SMART (Self-Monitoring, Analysis, and Reporting Technology) enabled, use the following command:
```bash
smartctl --scan | awk '{print $1}' | xargs -I {} sh -c "echo {}; smartctl -i {} | grep 'SMART support is:'"
```
This command will list all your disks and show whether SMART is supported and enabled.

## Step 12 - SMART Scrub Schedule

## Step 13 - Testing SMART notifications 

## Step 14 - ZFS Scrub Schedule

## Step 15 - Testing ZED notifications
To validate ZED (ZFS Event Daemon) notifications, follow these steps:
1. Navigate to the /tmp directory and create a sparse file:
```bash
cd /tmp
dd if=/dev/zero of=sparse_file bs=1 count=0 seek=512M
```
2. Create a test ZFS pool using the sparse file:
```bash
zpool create test /tmp/sparse_file
```
3. Initiate a scrub operation on the test pool:
```bash
zpool scrub test
```
The scrubbing should complete almost instantly since the test pool doesn't contain any data. If you receive an email notification, it confirms that ZED is functioning correctly.

4. After testing, clean up by exporting the test pool and removing the sparse file:
```bash
zpool export test
rm sparse_file
```

## Posible Future Guides
- Mailrise(Apprise) Notifications
- Opening github issue on notification

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
