# NVR_docker
definitions of containers on the NVR VM

## Introduction
This repo contains installation information to create a VM that will then host a docker container for frigate, and later on more containers that are meant to collaborate with it.
The code here is mainly the 'automation' stack, meaning the definitions for the bundle of containers that will be started with a single docker-compose.yml

Most of this development is done using vscode and the 'remote SSH' extension, and also the Docker extension.
The repo is cloned onto the VM.

When the installation steps are over and it is time to start the containers, in vscode connected remotely, right click on the docker-compose.yml to use the 'Compose up' command, then quickly right click the automation folder within the Docker extension view to 'Compose Logs'



## Creation of the VM and the frigate docker container in it
mainly followed the instructions:
- setup debian 12 on a VM in Synology VMM as per https://docs.frigate.video/guides/getting_started/
  - used storage created on the SSD volume (volume 3)
  - name: frigate_VM
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
- setup git on VM for work with https://github.com/sramshaw/NVR_docker.git
  - install git
  - clone  https://github.com/sramshaw/NVR_docker.git in the folder /var/lib/
  - setup global user.name  and user.email
    ```
    git config --global user.name "Your Name"
    git config --global user.email "your.email@example.com"
    ```
  - ?? install github authentication using commands at https://cli.github.com/manual/gh_auth
    ```
    apt install gh
    gh auth login  # follow the questions, use mechanism with web authentication
    gh auth setup-git
    ```
    - note that when trying to auth via a console, the step trying to open a browser fails, you can then go and do it manually in a browser and use the code provided by the cli process
- setup docker on VM
  - install docker as per https://docs.docker.com/engine/install/debian/
  - setup user papa as part of docker group
    ```
    sudo moduser -aG docker papa
    ```
  - then exit all ssh (if already in vscode , exit all) , for the new change to take effect
  - test with a simple `docker ps`
- setup vscode to edit git folder /home/papa/NVR_docker in Remote SSH to papa@nvr.lan
- setup vscode extension for docker
- install PCIe Coral TPU drivers (apex)
  - see https://gweb-coral-full.uc.r.appspot.com/docs/m2/get-started/#2a-on-linux
    ```
    echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    sudo apt-get update
    sudo apt-get install gasket-dkms libedgetpu1-std
    sudo sh -c "echo 'SUBSYSTEM==\"apex\", MODE=\"0660\", GROUP=\"apex\"' >> /etc/udev/rules.d/65-apex.rules"
    sudo groupadd apex
    sudo adduser $USER apex
    ```
- create an iSCSI initiator (client) for an iSCSI remote disc (here a Synology LUN on a  NVR disk /volume2)
  - install the driver, see https://reintech.io/blog/setting-up-iscsi-target-debian-12
    ```shell
    sudo apt install open-iscsi
    ```
  -  discover the initiator (this debian machine) 's IQN
    ```shell
    sudo cat /etc/iscsi/initiatorname.iscsi
    ```
  -  discover the iSCSI target it will aim at based on IP address - step needed anytime there is a change on the target (aka here: NAS remote iSCSI drive)
    ```shell
    sudo iscsiadm -m discovery -t sendtargets -p <TARGET IP>
    # make sure there is only one entry discovered with IPv4 address, if IPv6 comes too, see below
    ```
  - map the remote drive (iSCSI target) to /nvr_disk
    - the iSCSI target on Synology was setup to only allow the initiator above, and with CHAP authentication
    - edit initiator config
    ```shell
    sudo nano /etc/iscsi/iscsid.conf
    ```
    - uncomment line : `node.session.auth.authmethod = CHAP`
    - add line: `node.session.auth.username = <target_provided_username>`
    - add line: `node.session.auth.password = <target_provided_password>`
    - restart initiator ??
    ```shell
    ### sudo systemctl restart open-iscsi
    ```
    - connect via cli:
    ```shell
    # make sure to do the discover part again if you change the CHAP settings on the target, it will be taken in account below
    # and note that the node command uses te CHAP creds setup in iscsid.conf
    sudo iscsiadm -m node --targetname iqn.2000-01.com.synology:TRON.default-target.9330fcd89aa --portal 10.0.0.183 --login
    # disconnect using -u instead of --login
    # notice how a new entry (sdb in my case) is available with: ls /dev |grep sd
    # make sure in NAS LUN permission settings, if not 'Accept All' but 'custom' , to add your initiator's host with Read/Write (not default of no access!!!)
    # I was not successful using the checksum for header or data in Synology SAN -> iSCSI settings,
    # but successful using CHAP credentials
    ```
    - check that open-iscsi service can now work properly, it should restart in less than 5s with no error, based on the last 'discovery'
    ```shell
    sudo systemctl restart open-iscsi
    
    # note that if there is issue at boot where open-iscsi is exiting, check that it is not trying to reach 2 discovered targets, one IPv4 and one IPv6 for the same target
    # I had to disable IPV6 on the LAN (Synology->Control panel->Network -> Network interface -> pick yours -> edit , set IPv6 to OFF
    # after that the 'discover' was only finding IPv4 , and the service started to work, boot time improved and mounting was successful
    ```
    - format the iSCSI drive if needed (I use NTFS)
      ```shell
      sudo apt install ntfs-3g #was already installed on debian 12
      #find your device
      ls -l /dev/disk/by-path/
      # ex: result:  /dev/disk/by-path/ip-SOME-IP:3260-iscsi-iqn.2000-01.com.synology:NNNN.default-target.XXXX-lun-1 --> ../../sdb  = /dev/sdb
      # then create partition with 'parted'
      sudo parted /dev/sdb
      # then type the following except '(parted) '
      (parted) mklabel gpt
      (parted) mkpart primary ntfs 0% 100%
      (parted) quit
      # now you have a partition like /dev/sdb1
      lsblk # will show sdb1
      # fast format, then mount
      sudo mkfs.ntfs -f -L "frigate_data" /dev/sdb1
      sudo mkdir -p /nvr_disk
      sudo mount -t ntfs-3g /dev/sdb1 /nvr_disk
      # get UUID for persistent mount:
      sudo blkid /dev/sdb1
      # add this line in fstab, replace [UUID] and using no double quotes apparently
      # UUID=[UUID] /nvr_disk ntfs-3g defaults,_netdev 0 0
      sudo nano /etc/fstab
      reboot
      # check that the mount works
      mount |grep nvr  #should give you a line
      ```
      
- reboot the VM
- First launch of the 'stack', aka docker compose
  - in the explorer view of vscode, right click on docker-compose.yml , click 'compose up'
  - quickly in the docker extension view, find the 'automation' stack , and right click to see the logs
    - note that the log contains the new password generated, which is needed to login into the frigate app
    - which is at https://nvr.lan:8971

## Notes on configuration

Configuration reference: 

The unifi G3 camera advertises a **rtsps** protocol, which does not seem covered by the configuration as per 
However I found here https://github.com/AlexxIT/WebRTC/blob/master/README.md#known-work-cameras to change 
- from : rtsps://192.168.1.1:7441/XXXX?enableSrtp
- to   : rtsp://192.168.1.1:7447/XXXX?enableSrtp


## Move the storage to the suveillance disk

As the VM runs on a small SSD, use a dedicated HDD to store the video. There is no value to the videos in case of the HDD failure, just change it, hence no worries about the content of the vDisk

This is done via remapping of the VM side (left) folder of the bind:
```
(from docker-compose.yml)

/var/lib/NVR_docker/automation/frigate/storage:/media/frigate
```

steps so far:
- it seems there is no direct way to achieve this in VVM
- use iSCSI LUN to give access to NVR storage to frigate
- you will end up with the HDD 's iSCSI disk mounted as /nvr_disk

- move the container binding to the new drive and move contents
  - shutdown automation stack
  - modify docker-compose.yml to binding:
  ```
  - /nvr_disk:/media/frigate
  ```
  - because there is already content in the old folder, copy it in the new one
  ```
  rsync -av /var/lib/NVR_docker/automation/frigate/storage/ /nvr_disk/
  ```
- restart the automation stack

## passthrough and use Coral TPU for accelerator

### passing through 
done as described in https://github.com/sramshaw/pci_coral_on_synology

### install drivers

instructions from https://coral.ai/docs/m2/get-started/#4-run-a-model-on-the-edge-tpu

but there is an issue with adding Debian package repository
fix the update https://github.com/google-coral/edgetpu/issues/550#issuecomment-1627908277

stop after installing the drivers

#### trying an example FAILS due to python3 version
issue with python3 missing numpy

sudo apt install python3-pip
sudo apt install python3-numpy
sudo apt install python3-pillow

ends up in dead end with python too recent.
The following packages have unmet dependencies:
 python3-pycoral : Depends: python3-tflite-runtime (= 2.5.0.post1) but it is not going to be installed
                   Depends: python3 (< 3.10) but 3.11.2-1+b1 is to be installed

found a message that is about coral on frigate container
https://github.com/google-coral/edgetpu/issues/771#issuecomment-1609875625

it was not worth the effort, I focused on frigate instead

#### passing device to 'frigate' container
added section to the docker-compose.yml
passing device /dev/apex_0

created detector in frigate config
