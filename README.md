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
- there is a trick I detailed in a separate repo: https://github.com/sramshaw/synology_vdisks_swap by swapping iSCSI LUN definitions between 2 VMs
- you will end up with the HDD 's vDIsk mounted as /nvr_disk

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