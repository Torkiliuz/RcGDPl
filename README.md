# Rclone + Google Drive + Plex
<p align="center"><img src="https://healthchecks.io/badge/1481177c-f2fd-4210-8c76-335b22/dEOieWBA/Google%25C2%25B7Drive.svg"> <img src="https://healthchecks.io/badge/1481177c-f2fd-4210-8c76-335b22/q8Tr5b3Q/Rclone.svg"> <img src="https://healthchecks.io/badge/1481177c-f2fd-4210-8c76-335b22/KYydfvv6/Server.svg"> <img src="https://healthchecks.io/badge/1481177c-f2fd-4210-8c76-335b22/RkteH8QW/www.svg"></p>

For a while I've been running my media server setup as a hybrid cloud solution. It has been working well for me, and although there's a small amount of added latency it's still snappy enough to compete with a fully local setup. There are many things that make a hybrid cloud setup immensely more complicated than a strictly local setup though, and the learning curve is fairly steep. If you feel up for it and have some time available, this might be for you.  
In order to keep this from being a really long post I will not go as much into detail about each step, but I'll rather point to the resources necessary to understand how the setup works. Some of the scripts that are used in this guide are written by me, and they do a simple form of error checking, but they're in no way flawless. Use at your own risk.  
I know that some of the solutions in here can be automated and improved upon, this is not a *be all and end all* solution by any means.

First and foremost; using Google Drive has its limitations; most importantly the limit of `750 GB` uploaded per user every `24 hour`. One way to work around this limitation is to use Google Team Drive. Google Team Drive does not limit you to only use Google Business-registered accounts, so create a few free gmail accounts; link them to your Team Drive, and that limitation is circumvented. In my testing the maximum allowed free Google accounts created in a day is `3`; so it's a good idea to start creating these before going through the next steps.  
*We'll also use the same account-switching strategy when we mount drives in order to bypass restrictions set on the amount of API-requests allowed per user per day.*

Tools we use in order to get this set up and working:  
[MergerFS](https://github.com/trapexit/mergerfs) - (*configuration*)  
[Rclone](https://rclone.org/) - (*configuration*)  
[Systemd Rclone Mount](https://raw.githubusercontent.com/animosity22/homescripts/master/rclone-systemd/gmedia-rclone.service) - (*modified*)  
[Nightly upload-script](https://raw.githubusercontent.com/animosity22/homescripts/master/scripts/upload_cloud) - (*modified*)  
[Swatchdog](https://github.com/ToddAtkins/swatchdog) - (configuration files)  
[Reclone](https://gist.github.com/Torkiliuz/90c7d50845dec168cc0de2c82c3672c3) - (shell script)  
[Plex Autoscan](https://github.com/l3uddz/plex_autoscan) - (*optional*)  

# MergerFS
MergerFS is best installed by grabbing the appropriate release for your distribution [here](https://github.com/trapexit/mergerfs/releases).  
Set up your cache drives as the first drives that files get written to. You'll need a set amount of drives based on how much media you process through a week. I'd recommend at least `8 TB`, just in order to have a bit of a buffer in case an upload session fails, etc..  
Your `/etc/fstab` should look a bit like this:
```
#Cache Drives
UUID="b6ed03ee-b1dc-417f-b5a6-fe5603f03c13" /media/disk01 ext4 defaults,nofail,x-systemd.device-timeout=10 0 2
UUID="d3689128-4477-4328-a7a0-46b5b699f738" /media/disk02 ext4 defaults,nofail,x-systemd.device-timeout=10 0 2
UUID="7ffe898b-30f9-4ac7-9a07-c5cdeab7bc76" /media/disk03 ext4 defaults,nofail,x-systemd.device-timeout=10 0 2

#MergerFS
/media/disk* /files fuse.mergerfs defaults,sync_read,allow_other,category.action=all,category.create=ff,minfreespace=100G,fsname=Files 0 0
```
Check the MergerFS github for an explanation of the parameters used. Most importantly it gets the Cache Drives listed as the first drives, and therefore `category.create=ff` makes sure that every write is tried on these drives first, instead of writing to the Google Drive mount directly.  
We're not using `/etc/fstab` to mount the Google Drive remote, as we need a more dynamic way to remount based on when rate limiting occurs. We'll use `Systemd` combined with `Swatchdog` to solve this.

# Rclone
First you'll install [Rclone](https://rclone.org/downloads/). This differs from platform to platform, and also depends a bit on if you want the latest development release, latest stable release, or if you want to go for the one already in your package manager. I'd recommend going with the latest stable release.  
In order to configure Rclone run `rclone config` in your terminal.  
Press `n` to create a new remote, which will define your connection to Google Drive.  
Name it something that makes sense, and add a number to the end of it. For example `gtdrive1`. We'll use the number for autorotation of users later.  
Press the number that corresponds to **Google Drive** in the next part; this number changes throughout different versions of Rclone.  
Create a custom `Client Id` by following **[this guide(!)](https://rclone.org/drive/#making-your-own-client-id)**; add that to **Application Client Id**, press `Enter`.  
Do the same for **Application Client Secret**.  
Choose **Full access all files** if you get asked.  
Leave **ID of the root folder** blank, as we're only using a subfolder on the Team Drive; just press `Enter`.  
Just press `Enter` for **Service Account Credentials JSON file path**.  
Press `n` to skip **advanced config**.  
Answer the question about auto config based on how you're running the commands. If you're running through SSH choose `n`.  
Open the URL you're giving in a browser locally if you're running through SSH.  
Choose the account to log in with (use an account that is linked to the Team Drive you want to access).  
Paste the verification code that you get back into the SSH-session.  
Press `y` when asked to configure it as a Team Drive.  
Choose the Team Drive you want to access from the list of choices that gets printed by writing the number corresponding to it.  
Verify that things look OK, and press `y` if it is.  
Now we're back to the list of configured remotes.  
Press `n` to add another remote; we will encrypt our data so only we can access it, and also so that we don't get metadata leaked.  
This drive also needs a name. Building on the name from the first remote, name it something that makes sense, and add a number to the end for scripted account-rolling. For example `gtcrypt1`.  
Since we're encrypting a remote we'll choose the option **Encrypt/Decrypt a remote** in the menu.  
We then get asked for the remote to encrypt. In order to have the ability to use the Google Drive for other files than media we will use a subfolder for media. This way our top level directory only has 1 folder and doesn't get as cluttered. Add the remote you previosly created. In the example this would look like `gtdrive1:Media`.  
For encryption choose **Encrypt the filenames**, also known as **standard** (option `2` as of writing this guide).  
Then choose **Encrypt directory names** (`true`).  
Since we'll only have one account connected at a time it's important that we set the encryption password and salt the same for each account/remote that we set up. Otherwise we will only see the files uploaded by that particular user for each remote. Therefore you'll choose **Yes type in my own password** for both. The password and salt can and should be different though, but use the same password and salt for each account.  
We do not need to edit advanced config, so choose `n` for this.  
If everything looks OK, type `y`.  

To check that the remote mount works you can run `rclone lsd gtcrypt1:`. If you don't get a warning message everything should be OK.

## Systemd Rclone Mount
In order to mount our remote reliably with rolling accounts we'll use Systemd combined with log-watching done by Swatchdog. Animosity22 inspired this part, and the Systemd-file we're using is very similar to [his](https://raw.githubusercontent.com/animosity22/homescripts/master/rclone-systemd/gmedia-rclone.service).  
The Systemd-file we're using should be written to `/etc/systemd/system/rclone.service` and contain the following:
```
[Unit]
Description=RClone Service
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
Environment=RCLONE_CONFIG=/home/username/.config/rclone/rclone.conf
ExecStart=/usr/bin/rclone mount gtcrypt1: /media/diskdrive \
   --allow-other \
   --dir-cache-time 96h \
   --drive-chunk-size 32M \
   --log-level INFO \
   --log-file /home/username/logs/rclone.log \
   --timeout 1h \
   --umask 002 \
   --rc
ExecStop=/bin/fusermount -uz /media/diskdrive
Restart=on-failure
User=username
Group=plex

[Install]
WantedBy=multi-user.target
```
Replace every instance of `/home/username` with the homefolder of the user you want to run Rclone as. When running `rclone config` the configuration is automatically saved to the homefolder of the user running that command. In the case of running as the user `username`, the configuration would be saved to `/home/username/.config/rclone/rclone.conf`.  
In order to have the logs for rclone be saved to `/home/username/logs/rclone.log` the folder `/home/username/logs` needs to exist first. Create the folder by running `mkdir /home/username/logs` in the terminal.  
Also change `User=username` to the correct username.

## Nightly upload script
Once again we're getting some inspiration from Animosity22's [github](https://raw.githubusercontent.com/animosity22/homescripts/master/scripts/upload_cloud). The modified version we're using is named `/home/username/rcup`, and looks like this:
```
#!/bin/bash
# RClone Config file
RCLONE_CONFIG=/home/username/.config/rclone/rclone.conf
export RCLONE_CONFIG
LOCKFILE="/var/lock/`basename $0`"


new="gtcrypt1"

for newfolder in "$1"
do
        case $newfolder in
                "1" )
                new="gtcrypt1";;
                "2" )
                new="gtcrypt2";;
                "3" )
                new="gtcrypt3";;
                "4" )
                new="gtcrypt4";;
                "5" )
                new="gtcrypt5";;
                "6" )
                new="gtcrypt6";;
                "7" )
                new="gtcrypt7";;
                "8" )
                new="gtcrypt8";;
        esac
done

(
  # Wait for lock for 5 seconds
  flock -x -w 5 200 || exit 1

# Move older local files to the cloud
/usr/bin/rclone move /disk/ $new: --checkers 3 --max-transfer 730G --fast-list --log-file /home/username/rcup.log --tpslimit 3 --transfers 3 -v --exclude-from /home/username/.exclude --delete-empty-src-dirs

) 200> ${LOCKFILE}
```
We're limiting the upload to 730 GB, as it's better to have uploaded a complete file and not hit the rate limit, than getting halfway through an uploaded file, and then fail because of rate limiting.  
There's an added functionality to be able to push to different remotes manually by writing a specific number after `rcup`, for example `bash /home/username/rcup 3` to use `gtcrypt3`.  
For the nightly upload it will use `gtcrypt1` by default.  
We use `cron` in order to run this every night at 02:00. Edit crontab for root by typing `sudo crontab -e` in terminal.  
Add the following to crontab:
```
0 2 * * * /home/username/rcup 2>&1
```

We're using `/home/username/.exclude` to ignore some files and folders that we don't want to upload:
```
Games/**
lost+found/**
Misc/**
Music/**
Software/**
.Trash/**
*.fuse_hidden*
```

# Swatchdog
Swatchdog is part of your distributions package manager; install it as any other application you'd install through the package manager of your respective distribution.  
For Swatchdog we'll also create a Systemd-file. Edit it as `/etc/systemd/system/swatch.service`, and add the following to it:
```
[Unit]
Description=Swatch Log Monitoring Daemon
After=syslog.target network.target auditd.service sshd.service

[Service]
ExecStart=/usr/bin/swatchdog -c /home/username/reclone.conf -t /home/username/logs/rclone.log --pid-file=/var/run/swatch.pid --daemon
ExecStop=/bin/kill -s KILL $(cat /var/run/swatch.pid)
Restart=on-failure
Type=forking
PIDFile=/var/run/swatch.pid

[Install]
WantedBy=multi-user.target
```
Edit `/home/username` as applicable here too.  
We're telling Swatchdog to run with the configuration from `/home/username/reclone.conf`; that file looks like this:
```
watchfor /googleapi: Error 403: Rate Limit Exceeded, rateLimitExceeded/
exec sudo bash /home/username/reclone
```
Rather simple stuff, right? This configuration tells Swatchdog to look for any event in the log file `/home/username/logs/rclone.log` that matches with `Error 403: Rate Limit Exceeded`.  
Everytime a `Rate Limit Exceeded` event occurs we get poor performance from the Google Drive mount. Therefore we switch over to a different user account that does not have the Rate Limit Exceeded for that day. 
The switching is done by using the following script (Swatchdog fixes this automatically):
```
#!/bin/bash
#Made by Torkiliuz

#This is used multiple times, so it's a variable (tr -dc '0-9' gets only numbers from the output)
check=$(grep [0-99]\: /etc/systemd/system/rclone.service|awk '{print $3}'|tr -dc '0-9')

#Define amount of rclone-mounts here
num=8

#---------------------------------------------------------------------------------------#

#Adding option for force-parameters
force=0
while getopts f option; do
    case $option in
        f)
          force=1 >&2; echo "forcing rotation" ;;
       \?)
          echo "Invalid option: -$OPTARG" >&2; exit 2 ;;
    esac
done

#Check if Plex Transcodes are running, as they will crash if so
while pgrep -x "Plex Transcoder" >/dev/null; do
    if [ $(echo $force) -lt 1 ]; then
    echo -e "ERROR: Plex Transcoder is currently running. Please stop any open transcodes\nSleeping for 10 seconds" >&2
    sleep 10
    elif [ $(echo $force) == 1 ]; then
        break
    fi
done

###Inform and also unmount drive
echo "ReeeCloning" && fusermount -uz /media/diskdrive

#If we're on the highest number we should start at the beginning (uses exactly the highest number here)
if [ "$check" == "$num" ]; then
    sed -i "s/gtcrypt[0-9]\+\:/gtcrypt1\:/" /etc/systemd/system/rclone.service
    echo "changed to gtcrypt1:"
    systemctl daemon-reload && systemctl restart rclone.service
    exit 0
fi

#Runs a rolling increment of drive-numbers (uses 1 less than what you want to go up to here)
if [[ "$check" -ge 1 && "$check" -le $((num-1)) ]]; then
    ((check++))
    sed -i "s/gtcrypt[0-9]\+\:/gtcrypt$check\:/" /etc/systemd/system/rclone.service
    echo "changed to gtcrypt"$check":"
    systemctl daemon-reload && systemctl restart rclone.service
    exit 0
fi


#Kill it with fire if everything else fails
exit 1
```
In this example we have 8 accounts, going from `gtcrypt1` all the way up to `gtcrypt8`. Adjust the numbers to the amount of accounts you've set up with `rclone config`.

# Plex Autoscan
The guide on [Github](https://github.com/l3uddz/plex_autoscan) is already quite good here, so I'll just make this short. As an additional tool, Plex Autoscan can make you have less API-requests to Google Drive, so you don't need to do as much account-rolling. This tool is most useful for cases when you have Sonarr, Radarr and Lidarr autodownloading files.
