# Wireguard VPN Ansible setup

[![CI][badge-gh-actions]][link-gh-actions]

This project installs and configures my debian-based [Wireguard](https://www.wireguard.com/) VPN configuration and a management UI ([WG Portal](https://wgportal.org/latest/)). In addition, since exposing a VPN to the internet can be a security risk if you don't know what you're doing, this project also sets up some features and services to enhance the server security. The VPN can be installed in two different methods.
- Vanilla Wireguard, which installs and configures a plain Wireguard service.
**Note:** The steps performed during the vanilla Wireguard installation are based on the [How to set up wireguard on ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-wireguard-on-ubuntu-20-04) article from Digital Ocean. 
- Wireguard through [PiVPN](https://www.pivpn.io/).
**Note:** The steps performed during the PiVPN Wireguard installation are based on the [PiVPN official website](https://www.pivpn.io/). 
- Wireguard through [Wireguard Portal](https://wgportal.org/), which provdes a management UI and a wrapper around [wgctrl-go](https://github.com/WireGuard/wgctrl-go) and [netlink](https://github.com/vishvananda/netlink), for interface handling.
**Note:** The steps performed during the Wireguard Portal installation are based on the [Wireguard Portal official website](https://wgportal.org/latest/documentation/getting-started/binaries/). 

This project also provides a way to migrate wireguard configurations from one host to another while keeping existing peers working.

## Available playbooks

This project contains multiple playbooks that can run in sequence or separately.

**`playbooks/prepare.yml`**: This playbook creates a new ansible user with a new private key in the target machine and changes the default ssh port.

**`playbooks/main.yml`**: This playbook runs the major chunk of the VPN setup. It runs the following ansible roles:
- `rafael-c-alexandre.security`
- `rafel-c-alexandre.ntp`

and installs a Wireguard server (by default in `/etc/wireguard/${INTERFACE}.conf)`), plus generates the requested clients configs (by default in `/etc/wireguard/configs`). 

**`playbooks/backup.yml`**: This playbook backs up the current wireguard configuration, depending on the installation mode. It creates a `$PROJECT_ROOT/migration/wireguard-migration.zip` path where the backed up data is stored.


## Installation

The recommend installation guide is as follows:

1. Ensure the Raspberry Pi has Pi OS installed and SSH running and reachable.
2. Get the PI IP address or hostname through `ip a` (or any other network-related tool) or in the router UI.
3. Add the IP address/hostname in an inventory file such as `inventories/hosts.ini`.
4. Clone or download this repository to your local drive.
5. Run `ansible-galaxy install -r requirements.yml` inside this directory to install required Ansible roles and collections.
6. To prepare the ansible environment,
    - Change host in `inventories/hosts.ini` file.
    - Edit any ansible variables in the `configs/ansible_pre_prepare.cfg` file, namely `remote_user` and `private_key_file`.
    - Add any custom playbook configuration variables in a new `configs/config_pre_prepare.yml` file.

7. In order to copy the configurations needed for the `prepare` playbook, in the root folder, run 
```bash
cp configs/ansible_pre_prepare.cfg ansible.cfg && cp configs/config_pre_prepare.yml config.yml && cp inventories/hosts.ini inventory
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
cp configs/ansible_after_prepare.cfg ansible.cfg && cp configs/config_after_prepare.yml config.yml && cp inventories/hosts.ini inventory
``` 
11. In the root folder, run  
``` bash
ansible-playbook playbooks/main.yml
``` 

**Note:** In order to migrate  a previously backed up wireguard setup, the variable `wireguard_migration_zip_path` needs to be defined and pointing at the .zip file where the migration data is stored. The playbook is then run the same way.

12. (Optional). If you just want to run the Wireguard-related task, to either setup the VPN or generate/edit/delete a peer, you can run 
```bash
ansible-playbook playbooks/main.yml --tags wireguard
``` 

### Manual actions

This set of playbooks does pretty much everything automatically. However, some actions still require manual intervention.

#### Without WG Portal

If wireguard is to be used without WG Portal management UI, the following actions should be performed:

1. If the VPN host is behid a device that performs NAT, port forwarding should be configured on that device to route any traffic on Wireguard's port (by default 51820) to the VPN host. Conversely, if the VPN host has a public IP address, this step is not necessary.

2. In order to get the peers set up, we need to retrieve the configs from the peer configs folder. The main.yml playbook also installs [qrencode](https://linux.die.net/man/1/qrencode), which is an handy tool to generate QR codes out of the Wireguard peer config files, which can then be scanned with a mobile device. Example:
```bash 
ssh $USER@$VPN_HOST:$VPN_PORT -i ${IDENTITY_FILE}
qrencode -t ansiutf8 < /etc/wireguard/configs/${CLIENT}.conf
```

Alternatively, the configs can be downloaded from the server through tools like [scp](https://linux.die.net/man/1/scp). Example: 
```bash 
scp $USER@$VPN_HOST:$VPN_PORT:/etc/wireguard/configs/${PEER} ${DESIREDTARGETLOCATION} -i ${IDENTITY_FILE}
```

#### With Wireguard Portal

If wireguard is to be used with WG Portal management UI, the following actions should be performed:

1. Navigate ton the UI, login and edit the already created wg interface (in *server mode*) by selecting the *Peer Defaults* tab.
2. Add the following values:
  - Endpoint (server_hostname:wireguard_port)
  - IP networks (the IP addresses range from which the peers will get addresses, the default should be fine unless some other subnet may interfer)
  - Allowed IP Addresses (which target IP ranges should be routed through the VPN interface, if nothing specific, enter: `0.0.0.0/0` and `::0/0` for all the traffic to be router)
  - DNS (DNS servers which the peer should use when connectd to the VPN)
3. Save and exit.

## Overriding Defaults

You can override any of the defaults configured in `default.config.yml` by creating a `config.yml` file and setting the overrides in that file. For example, you can customize the wireguard ip addresses, interfaces and peers to be generated with something like:

```yaml
---
---
ssh_pub_key_relative_location: ~/.ssh/ansible.pub
ssh_config_path: /etc/ssh/sshd_config
ssh_port: 2849
wireguard_port: 51820

wireguard_interface: wg0
wireguard_installation_mode: pivpn # 'vanilla_wireguard', 'pivpn' or "wireguard_portal".
# Migratation zip file path, relative to the playbooks folder
# wireguard_migration_zip_path:

wireguard_conf_dir: /etc/wireguard
wireguard_peers_allowed_ips:
  - 0.0.0.0/0
  - ::0/0
wireguard_dns:
  - 9.9.9.9
  - 149.112.112.112

wireguard_server_interface: "{{ wireguard_interface }}"
wireguard_server_addresses:
  - 10.8.0.1/24
  - fd11:5ee:bad:c0de::a9c:1801/64
wireguard_server_mtu: 1420
wireguard_server_listen_port: "{{ wireguard_port }}"
wireguard_server_default_interface: eth0
# Define the following variable if there is a dynamic DNS record
# for the VPN server.
# wireguard_server_ddns_name: pivpn.ddns.net

wireguard_peers: []

# PiVPN related configs
pivpn_dhcpReserv: 1
pivpn_install_home: /root
pivpn_install_user: root
pivpn_VPN: wireguard
pivpn_pivpnNET: 10.8.0.0
pivpn_subnetClass: 24
pivpn_pivpnPROTO: udp
pivpn_pivpnPORT: "{{ wireguard_port }}"
pivpn_pivpnDNS1: 9.9.9.9
pivpn_pivpnDNS2: 149.112.112.112
pivpn_UNATTUPG: 0
pivpn_pivpnenableipv6: 1
pivpn_pivpnNETv6: "fd11:5ee:bad:c0de::"
pivpn_subnetClassv6: 64
pivpn_pivpnDEV: "{{ wireguard_interface }}"
pivpn_IPv6dev: "{{ wireguard_interface }}"
pivpn_pivpnINSTALLED_PACKAGES: "(grepcidr bsdmainutils wireguard-tools qrencode)"
# Define the following two variable if the VPN server
# is behind a NAT gateway.
# pivpn_IPv4addr: 192.168.2.13/24
# pivpn_IPv4gw: 192.168.2.254

# Wireguard portal related configs
wireguard_portal_version: v2.0.0-beta.7
wireguard_portal_host: 192.168.1.68
wireguard_portal_port: 8888
wireguard_portal_binary_directory: /opt/wg-portal
wireguard_portal_log_file_path: /var/log/wg-portal.log
wireguard_portal_logrotate_config_path: /etc/logrotate.d/wg-portal

security_ufw_rules:
  - rule: allow
    to_port: "{{ ssh_port }}"
    protocol: tcp
    comment: allow-ssh
  - rule: allow
    to_port: "{{ wireguard_port }}"
    protocol: udp
    comment: allow-wireguard

```

Any variable can be overridden in `config.yml`; see the supporting roles' and documentation for a complete list of their available variables.


## Testing the Playbooks

This project is [continuously tested on GitHub Actions](https://github.com/rafael-c-alexandre/vpn-playbook/actions/workflows/ci.yml), where all its playbooks are tested in sequence (including both VPN installation options).


License
-------

MIT

[badge-gh-actions]: https://github.com/rafael-c-alexandre/vpn-playbook/actions/workflows/ci.yml/badge.svg
[link-gh-actions]: https://github.com/rafael-c-alexandre/vpn-playbook/actions/workflows/ci.yml
