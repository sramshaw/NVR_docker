# NVR_docker
definitions of containers on the NVR VM

## Creation of the VM and the frigate docker container in it
mainly followed the instructions:
- setup debian 12 on a VM in Synology VMM as per https://docs.frigate.video/guides/getting_started/
  - used storage created on the SSD volume (volume 3)
  - name: NvrVM
  - host name in laptop hosts list: nvr.lan
  - non root user: papa
  - note that usermod is available at /usr/sbin/usermod
- setup user as part of docker group
    ```/usr/sbin/usermod -aG docker papa```
  - had to reboot the VM for vscode to be able to use proper group when using docker commands
- setup ssh withouth password as per https://superuser.com/questions/96051/ssh-from-windows-to-linux-without-entering-a-password
  - setup passwordless ssh from laptop to nvr.lan
  - (setup passwordless ssh from laptop to admin.lan)
- ?? setup VM to have its own identity ??
- setup git for work with https://github.com/sramshaw/NVR_docker.git
  - install github authentication using commands at https://cli.github.com/manual/gh_auth
    ``` apt install gh
    gh auth login  # follow the questions, use mechanism with web authentication
    gh auth setup-git
    ```
    - note that when trying to auth via a console, the step trying to open a browser fails, you can then go and do it manually in a browser and use the code provided by the cli process
  - setup global user.name  and user.email
  - clone  https://github.com/sramshaw/NVR_docker.git in the folder /var/lib/
- setup vscode to edit git folder /var/lib/NVR_docker in Remote SSH to papa@nvr.lan
- setup vscode extension for docker
- First launch of the 'stack', aka docker compose
  - in the explorer view of vscode, right click on docker-compose.yml , click 'compose up'
  - quickly in the docker extension view, find the 'automation' stack , and right click to see the logs
    - note that the log contains the new password generated, which is needed to login into the frigate app
    - which is at https://nvr.lan:8971

## Move the storage to the suveillance disk

As the VM runs on a small SSD, use a dedicated HDD to store the video. There is no value to the videos in case of the HDD failure, just change it, hence no worries about the content of the vDisk

This is done via remapping of the VM side (left) folder of the bind:
```
(from docker-compose.yml)

/var/lib/NVR_docker/automation/frigate/storage:/media/frigate
```

steps so far:
- it seems there is no direct way to achieve this in VVM
- there is a trick by swapping iSCSI storage definitions between 2 VMs as per https://www.reddit.com/r/synology/comments/sixqn7/comment/i08jyw6/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button
- create a fake VM based in the HDD targetted, named fake4frigateVol3swap, using storage created on the HDD of size 4TB , which is most of the drive, so be used for video
- create a 2nd virtual storage of smallest size (10GB) for the true VM , which will be assigned in the SSD
- observe the definitions for those iSCSI drives:
    - ``` cat /volume2/@iSCSI/LUN/iscsi_lun.conf ```
        shows one entry
    - ``` cat /volume3/@iSCSI/LUN/iscsi_lun.conf ```
        shows 2 entries, one for good VM OS disk, one to be replaced with the HDD disk
- step1: identify the guids to swap, edit script to do the swap swap.sh
- step2: stop virtualisation and iSCSI service with
```
synopkgctl stop Virtualization
synopkgctl stop ScsiTarget
```
- step3: run swap.sh
  - actual call
```
./swap.sh 2 1 3 2  # means to swap volume 2 1st disk and volume 3 2nd disk
```
  - content of swap.sh
```
#!/bin/bash  
v1="$1"  #ex: 2 for /volume2
l1="$2"  #ex: 1 for 1st SCSI disk
v2="$3"  #ex: 3 for /volume3
l2="$4"  #ex: 2 for 2nd SCSI disk

# real prefix + sufix
prefix="/volume"
sufix="/@iSCSI/LUN"
# test prefix + sufix
#prefix="/volume1/homes/sylvain_admin/NAS/pci_coral_on_synology/swap_disks/originals/vol"
#sufix=""

vol1="${prefix}${v1}${sufix}"
vol2="${prefix}${v2}${sufix}"
echo vol1 $vol1
echo vol2 $vol2
guid1=`cat $vol1/iscsi_lun_acl.conf |grep lun_uuid | sed "${l1}q;d" | sed 's/lun_uuid=//'`
guid2=`cat $vol2/iscsi_lun_acl.conf |grep lun_uuid | sed "${l2}q;d" | sed 's/lun_uuid=//'`
echo guid1 $guid1
echo guid2 $guid2
mv $vol1/VDISK_BLUN/$guid1  $vol1/VDISK_BLUN/$guid2
mv $vol2/VDISK_BLUN/$guid2  $vol2/VDISK_BLUN/$guid1
find $vol1 -name "*.conf" -exec sed -i "s/${guid1}/${guid2}/" {} \;
find $vol2 -name "*.conf" -exec sed -i "s/${guid2}/${guid1}/" {} \;
```
- step4: start virtualisation and iSCSI service with
```
synopkgctl start Virtualization
synopkgctl start ScsiTarget
```
