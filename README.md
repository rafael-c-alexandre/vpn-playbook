# NAS setup Ansible Playbook

[![CI][badge-gh-actions]][link-gh-actions]

This playbook installs and configures most of the software I use for my raspberry pi NAS. Some things are slightly difficult to automate and test, so I still have a few manual installation steps.

## Available playbooks

This project contains multiple playbooks that can run in sequence or separately.

**`playbooks/prepare.yml`**: This playbook creates a new ansible user with a new private key in the target machine and changes the default ssh port.

**`playbooks/main.yml`**: This playbook runs the major chunk of the NAS setup. It runs the following ansible roles:
- `rafael-c-alexandre.security`
- `rafel-c-alexandre.ntp`
- `geerlingguy.docker`
- `vladgh.samba`

installs the following software:
- [docker](https://www.docker.com/)
- [samba](https://www.samba.org/)
- [OMV 7](https://www.openmediavault.org/)
- [immich](https://immich.app/) 
- [rclone](https://rclone.org/) command line utility

and copies my NAS backup configuration:
- [backblaze B2](https://www.backblaze.com/cloud-storage) backup/restore scripts
- logrotation files.

**`playbooks/upgrade_immich.yml`**: This playbook pulls new immich-related docker images, if available, and restarts the compose service.

## Installation

The recommend installation guide is as follows:

1. Ensure the Raspberry Pi has Pi OS installed and SSH running and reachable.
2. Get the PI IP address or hostname through `ip a` (or any other network-related tool) or in the router UI.
3. Add the IP address/hostname in an inventory file such as `inventories/hosts.local.ini`.
4. Clone or download this repository to your local drive.
5. Run `ansible-galaxy install -r requirements.yml` inside this directory to install required Ansible roles and collections.
6. To prepare the ansible environment,
    - Change host in `inventories/hosts.ini` file.
    - Edit any ansible variables in the `configs/ansible_pre_prepare.cfg` file, namely `remote_user` and `private_key_file`.
    - Add any custom playbook configuration variables in a new `configs/config_pre_prepare.yml` file.

7. In order to copy the configurations needed for the `prepare` playbook, in the root folder, run 
```bash
cp configs/ansible_pre_prepare.cfg ansible.cfg && cp configs/config_pre_prepare.yml config.yml && cp inventories/hosts.ini inventory`
```
8. In the root folder, run 
``` bash
ansible-playbook playbooks/prepare.yml
``` 
9. Because the ssh config and other properties were changed, we need to update our configs:
    - Edit any ansible variables in the `configs/ansible_after_prepare.cfg` file, namely `remote_user` and `private_key_file`.
    - Add any custom configuration variables in a new `configs/config_after_prepare.yml` file, for instance, the two samba usernames (for me and my partner), respective shares, plus the common shared one.
10. In order to copy the configurations needed for the `main` playbook, in the root folder, run 
```bash
cp configs/ansible_after_prepare.cfg ansible.cfg && cp configs/config_after_prepare.yml config.yml && cp inventories/hosts.ini inventory` 
``` 
11. In the root folder, run  
``` bash
ansible-playbook playbooks/main.yml
``` 
12. (Additional) On demand,in the root folder, run (with the same 'after-prepare' configs)
```bash
ansible-playbook playbooks/upgrade_immich.yml
``` 

### Manual actions

Automating the OMV configuration is difficult so when it comes to set OMV up, the following actions need to be performed manually.

  1. Login to OMV.
  2. Create a RAID 1 array with the connected SSDs *(Storage -> Multiple Device)*.
  3. Make two partitions to the RAID 1, with desired sizes (using `fdisk` or `parted`).
  4. Create filesystems for both partitions *(Storage -> File Systems)*.
  5. Add backup scheduled tasks *(System -> Scheduled Tasks)*. They should all belong to the dedicated backups user.
        1. For shared SMB folder: `/usr/bin/bash /opt/scripts/backups/backup.sh -s /path/to/nas/partition/mounting/point/ -r pinas-backups-b2-crypt:nas`
        2. For immich: `/usr/bin/bash /opt/scripts/backups/backup.sh -s /path/to/immich/partition/mounting/point/ -r pinas-backups-b2-crypt:immich`
  6. Activate S.M.A.R.T. monitoring *(Storage -> S.M.A.R.T.)*
        1. Add the two SSDs *(Devices)*.
        2. Add scheduled `short self-test`s for the SSDs *(Scheduled Tasks)*.
  7. Enable notifications through e-mail *(System -> Notifications -> Settings)*, by adding the correct SMTP configuration.
  8. Replace `UPLOAD_LOCATION` and `DB_DATA_LOCATION` with the mounting points for the SSDs in the immich `.env` file.
  9. Copy rclone config file to the backups user `/$HOME/.config/` location.
  10. Change `/srv/mounting/points/` ownership to `root:{{ nas_group }}` and permissions to `2770`:
  ```bash
    chown -R root:{{ nas_group }} /srv/mounting/points/
    chmod -R "2770" /srv/mounting/points/
  ```

## Overriding Defaults

You can override any of the defaults configured in `default.config.yml` by creating a `config.yml` file and setting the overrides in that file. For example, you can customize the omv or immich setup directories packages and apps with something like:

```yaml
---
ssh_pub_key_relative_location: ansible.pub

security_enabled: true
ntp_enabled: true
docker_enabled: true
omv_enabled: false
samba_enabled: true
immich_enabled: true

nas_ssh_config_path: /etc/ssh/sshd_config
nas_ssh_port: 2849
nas_omv_port: 80
nas_immich_port: 2283
nas_smb_port: 445
nas_rclone_port: 5572
nas_user_group: nas

security_ufw_rules:
  - rule: allow
    to_port: "{{ nas_ssh_port }}"
    protocol: tcp
    comment: allow-ssh
  - rule: allow
    to_port: "{{ nas_omv_port }}"
    protocol: tcp
    comment: allow-omv
  - rule: allow
    to_port: "{{ nas_immich_port }}"
    protocol: tcp
    comment: allow-immich
  - rule: allow
    to_port: "{{ nas_smb_port }}"
    protocol: tcp
    comment: allow-smb
  - rule: allow
    to_port: "{{ nas_rclone_port }}"
    protocol: tcp
    comment: allow-rclone

omv_work_dir: /tmp/omv

samba_enable_netbios: false
samba_users:
  - name: ansible
    password: FIXME
samba_shares:
  - name: ansible
    path: /ansible
    comment: 'A test ansible share'
    group: "{{ nas_user_group }}"
    valid_users: ansible
    create_mode: "0660"
    force_create_mode: "0660"
    directory_mode: "0770"
    force_directory_mode: "0770"
    read_only: false
    browseable: false

immich_work_dir: /opt/immich
immich_user: immich

rclone_logrotate_config_path: /etc/logrotate.d/rclone
rclone_log_file_path: /var/log/rclone-b2-backup.log
rclone_remote_nas: pinas-backups-b2-crypt:nas
rclone_remote_immich: pinas-backups-b2-crypt:immich
rclone_backup_scripts_dir: /opt/scripts/backups

backups_user: backups

new_groups:
  - "{{ nas_user_group }}"
  - docker
new_users:
  - user: "{{ immich_user }}"
    groups: "{{ nas_user_group }},docker"
    system: true
    shell: /usr/sbin/nologin
  - user: "{{ backups_user }}"
    groups: nas
    system: true
    shell: /usr/sbin/nologin

```

Any variable can be overridden in `config.yml`; see the supporting roles' documentation for a complete list of available variables.


## Testing the Playbooks

This project is [continuously tested on GitHub Actions](https://github.com/rafael-c-alexandre/nas-playbook/actions/workflows/ci.yml), where all its playbooks are tested in sequence. A note on this: as of now, OMV installation cannot be tested since it does not work with Docker.


License
-------

MIT

[badge-gh-actions]: https://github.com/rafael-c-alexandre/nas-playbook/actions/workflows/ci.yml/badge.svg
[link-gh-actions]: https://github.com/rafael-c-alexandre/nas-playbook/actions/workflows/ci.yml
