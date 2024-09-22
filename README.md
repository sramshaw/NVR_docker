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
