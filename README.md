# Overview [![Build Status](https://travis-ci.org/bau-sec/ansible-openvpn-hardened.svg?branch=master)](https://travis-ci.org/bau-sec/ansible-openvpn-hardened)
*ansible-openvpn-hardened* is an [*Ansible*](http://www.ansible.com/home) playbook written to create and manage a hardened OpenVPN server instance in the cloud or locally. The created instance is configured for use as a tunnel for internet traffic allowing more secure internet access when using public WiFi or other untrusted networks. With this setup you don't have to trust or pay a shady VPN service; you control your data and the security of the VPN.

Other Ansible playbooks and roles exist to automate the installation of OpenVPN, but none configure the OpenVPN server to run entirely as an unprivileged user [as described in the OpenVPN docs](https://community.openvpn.net/openvpn/wiki/UnprivilegedUser) or use *systemd* to sandbox the OpenVPN server process.

*ansible-openvpn-hardened* also includes a playbook to run audits against the created server that independently verify the steps taken to harden the server. The supported auditing tools include [*OpenSCAP*](https://www.open-scap.org/) (with assistance from [*ubuntu-scap*](https://github.com/GovReady/ubuntu-scap) on Debian/Ubuntu), [*lynis*](https://cisofy.com/lynis/) and [*tiger*](http://www.nongnu.org/tiger/). Example output from these tools can be reviewed on the project wiki.

## Supported Targets

The intended target is a fresh instantiation of an image running on a cloud provider like [Digital Ocean](https://www.digitalocean.com/?refcode=5ab3eb461bcb) (warning: referal link) or Microsoft's [Azure](https://azure.microsoft.com/). Physical boxes or local VMs should work as well assuming they are accessible over SSH, but haven't been tested.

The following Linux distros are supported:

- CentOS 7.2 x64 (And by extension RHEL 7.2 should work, but this hasn't been tested)
- Ubuntu 16.04 x64
- Debian 8.5 x64

Other distros and versions may work but no promises. If support for another distro is desired, submit an issue ticket. Pull requests are always welcome.

# Hardening
Some of the steps taken to harden the server:
### General
- OpenVPN both **starts** and runs as the `openvpn` user instead of starting as `root` and dropping privileges
  - OpenVPN recommends this as being the most secure way to run on Linux. The OpenVPN wiki [describes how to do this](https://community.openvpn.net/openvpn/wiki/UnprivilegedUser), but those instructions don't work on distros using *systemd*. The ansible tasks defined in this project show how to get this working with *systemd*
- Firewall configured to only allow SSH access on the VPN LAN
  - The point of this script is to create a VPN tunnel; why not use that VPN to protect the SSH daemon as well? This not only makes the server more secure, it also eliminates the hundreds of daily log entries created by automated scripts trolling the internet for unsecured SSH ports. This makes the system logs easier to sift through.
- Numerous modifications to `/etc/ssh/sshd_config` for hardening. See [`harden_sshd.yml`](playbooks/roles/openvpn/tasks/harden_sshd.yml)
- [*auditd*](http://linux.die.net/man/8/auditd) is installed and configured to monitor administrative actions and/or suspicious activity. See [`harden_auditd.yml`](playbooks/roles/openvpn/tasks/harden_auditd.yml)
- [*AIDE*](http://aide.sourceforge.net/) is a file and directory integrity checker. It's installed, configured and an initial baseline is taken using `aide --init` just before the playbook finishes. See [`harden_aide.yml`](playbooks/roles/openvpn/tasks/harden_aide.yml)

### *systemd* sandboxing
There are lots of *systemd* detractors out there, but it's the default init system for Debian, Ubuntu and Red Hat. It does provide some useful features for sandboxing services. See [`etc_systemd_system_openvpn@.service.d_override.conf.j2`](playbooks/roles/openvpn/templates/etc_systemd_system_openvpn@.service.d_override.conf.j2) for how some of these features are enabled.
- The *systemd* unit option `CapabilityBoundingSet` is used to bound the Linux capabilities available to the OpenVPN server process, allowing only `CAP_NET_ADMIN` and `CAP_NET_BIND_SERVICE`
- The *systemd* unit options `ReadOnlyDirectories`, `InaccessibleDirectories`, `ProtectSystem` and `ProtectHome` are used to restrict the OpenVPN server process' access to the filesystem.
  - These aren't perfect as noted [in the *systemd* docs](https://www.freedesktop.org/software/systemd/man/systemd.exec.html), but better to have them than not:

    > Note that restricting access with these options does not extend to submounts of a directory that are created later on.

### OpenVPN server configuration
For the full server configuration, see [`etc_openvpn_server.conf.j2`](playbooks/roles/openvpn/templates/etc_openvpn_server.conf.j2)
- The OpenVPN server uses `--tls-auth`
  - From the [OpenVPN hardening guide](https://community.openvpn.net/openvpn/wiki/Hardening):

    > The tls-auth option uses a static pre-shared key (PSK) that must be generated in advance and shared among all peers. This features adds "extra protection" to the TLS channel by requiring that incoming packets have a valid signature generated using the PSK key... The primary benefit is that an unauthenticated client cannot cause the same CPU/crypto load against a server as the junk traffic can be dropped much sooner. This can aid in mitigating denial-of-service attempts.

- `push block-outside-dns` used by OpenVPN server to fix a potential dns leak on Windows 10
  - See https://community.openvpn.net/openvpn/ticket/605
- `cipher` set to `AES-256-CBC` by default
- `2048` bit RSA key size by default.
  - This can be increased to `4096` by changing `openvpn_key_size` in [`defaults/main.yml`](playbooks/roles/openvpn/defaults/main.yml) if you don't mind extra processing time. Consensus seems to be that 2048 is sufficient for all but the most sensitive data.

### OpenVPN client configuration
For the full client configuration, see [`client_common.ovpn.j2`](playbooks/roles/add_clients/templates/client_common.ovpn.j2)
- The OpenVPN client configurations generated use `--verify-x509-name` to prevent MitM attacks by verifing the server name in the supplied certificate matches the clients configuration.

### PKI
- [easy-rsa](https://github.com/OpenVPN/easy-rsa) is used to manage the public key infrastructure.
- OpenVPN is configured to read the CRL generated by *easy-rsa* so that a single client's access can be revoked without having to reissue credentials to all of the clients.
- The private keys generated for the clients and CA are all protected with a randomly generated passphrase to facilitate secure distribution to client devices.

The bullets above are just an overview. See the task definitions with filenames in the form `harden_<category>.yml` to review the complete set of steps.

The [`audit.yml`](playbooks/audit.yml) playbook can be run on the target server to independently verify the steps taken and their efficacy so you don't have to trust a random guy on the internet.

### Hardening - Audit Notes

None of the audit tools used by the `audit.yml` playbook give the server a perfect score, here are some brief notes on audit findings

- Separate partitions for /tmp, /home, etc.
  - Partitioning can be tricky on cloud providers and the creation of partitions can be difficult to script without risking data loss. This is probably best left to the OS image creater or installer of the OS.
- mount options such as `noexec`, `nosuid`, and `nodev` for /tmp, /dev/shm, etc.
  - This will likely be addressed in future versions. Pull requests welcome.
- Password requirements
  - The created server should only have one account able to login over SSH and it will be protected with a randomly generated password and pub/priv key pair. Unless you're using this server for other purposes with users who login regularly, setting requirements for password minimum length, complexity, expirations, etc. seems like box-checking, not adding additional security.
  - The *pam* module *pwquality* is installed if password requirements are necessary for your setup

# Quick start

> Warning: **Potential to lock yourself out of the target box.** One of the hardening steps configures SSH to only listen on the VPN interfaces. Make sure you have a backup method to access the server in case the VPN doesn't come up. For example, Digital Ocean provides console access through their admin panel.

> Requirement: **Currently a static IP is required** This may change in future releases with support for dynamic IP addresses. Static IPs are available on both Digital Ocean and Azure.

> Suggestion: **Use a fresh install or image as a target** This playbook should work well, but issues will be less likely and more easily resolved if you can start over easily. Also, given the potential for locking yourself out of the box, you don't want to lose access to import things that may be on an existing server.

Install the required packages if you don't have them already. On Ubuntu or Debian use the commands below. On other OSes, sub-in the appropriate package manager.

    sudo apt-get install python-pip git
    sudo pip install ansible

Get *ansible-openvpn-hardened*

    git clone https://github.com/bau-sec/ansible-openvpn-hardened.git

Create a target machine using your cloud provider of choice. The CentOS 7.2, Ubuntu 16.04 and Debian 8.5 images on Digital Ocean and Microsoft's Azure have been tested and should work well. Cloud providers are ideal because you can easily spin up a test box to try things out on and delete the instance when you're done or when you no longer need the VM. Other cloud providers, a local VM or box should work fine as well but haven't been tested.

Make sure you can ssh into the target machine that will become your OpenVPN box. If using a cloud provider they should provide you with login credentials and instructions. For example, to log into the `root` account on a box with the ip `192.168.1.10` use

    ssh root@192.168.1.10

Copy the example Ansible inventory to edit for your setup. `inventory.example` has example values for different ssh configurations. If *vim* isn't your editor of choice, substitute a different editor.

    cd ansible-openvpn-hardened/
    cp inventory.example inventory
    vim inventory

Run the install playbook

    ansible-playbook playbooks/install.yml

The playbook should run for **5-30 minutes** depending on how good your target box is at hashing and crypto operations. Assuming the above steps were successful, you should now have directory called `fetched_creds`. This contains the openvpn configuration files and private keys that can be distributed to your clients.

Try connecting to the newly created OpenVPN server

    cd fetched_creds/[server ip]/[client name]/
    openvpn [client name]@[random domain]-pki-embedded.ovpn

You'll be prompted for the private key passphrase, this is stored in a file ending in `.txt` in the client directory you just entered in the step above.

## Distributing key files

*ansible-openvpn-hardened* provides three different OpenVPN configuration files because OpenVPN clients on different platforms have different requirements for how the PKI information is referenced by the .ovpn file. This is just for convenience. All the configuration information and PKI info is the same, it's just formatted differently to support different OpenVPN clients.

- **PKI embedded** - the easiest if your client supports it. Only one file required and all the PKI information is embedded.
  - `X-pki-embedded.ovpn`
- **PKCS#12** - all the PKI information is stored in the PKCS#12 file and referenced by the config. This can be more secure on Android where the OS can store the information in the PKCS#12 file in hardware backed encrypted storage.
  - `X-pkcs.ovpn`
  - `X.p12`
- **PKI files** - if the above two fail, all clients should support this. All of the PKI information is stored in separate files and referenced by the config.
  - `X-pki-files.ovpn` - OpenVPN configuration
  - `ca.pem` - CA certificate
  - `X.key` - client private key
  - `X.pem` - client certificate

All private keys (embedded in config, pkcs, and .key) are encrypted with a passphrase to facilitate secure distribution to client devices.

For maximum security when copying the PKI files and configs to client devices don't copy the .txt file containing the randomly generated passphrase. Enter the passphrase manually onto the device after the key has been transferred.

## Private key passphrases

Entering a pass phrase every time the client is started can be annoying. There are a few options to make this less burdensome after the keys have been securely distributed to the client devices.

1. When starting the client, use `openvpn --config [config] --askpass [pass.txt]` if you don't want to enter the password for the private key

  From the OpenVPN man page:

  > If file is specified, read the password from the first line of file. Keep in mind that storing your password in a file to a certain extent invalidates the extra security provided by using an encrypted key.

1. Remove or change the passphrase on the private key

        openssl rsa -in enc.key -out not_enc.key

# Managing the OpenVPN server

## Credentials

Credentials are generated during the install process and are saved as yml formatted files in the Ansible file hierarchy so they can be used without requiring the playbook caller to take any action. The locations are below.

- CA Private key passphrase - saved in `group_vars/all.yml`
- User account name and password - saved in  `group_vars/openvpn-vpn.yml`

After the install.yml playbook has successfully been run, **you'll only be able to SSH into the box when connected to the VPN** using the account defined in `group_vars/openvpn-vpn.yml`

    ssh [created_user]@10.9.0.1

## Add clients

By default only two clients are created: `laptop` and `phone`. (These defaults can be changed by editing `openvpn_clients` in [`defaults/main.yml`](playbooks/roles/openvpn/defaults/main.yml))

Connect to the VPN before running the playbook. For example, to create a client named `cool_client` use

    ansible-playbook playbooks/add_clients.yml -e clients_to_add=cool_client

## Revoke client access

First connect to the VPN. To revoke `cool_client`'s access

    ansible-playbook playbooks/revoke_client -e client=cool_client

## Audit server

First connect to the VPN. Run

    ansible-playbook playbooks/audit.yml

The reports will be placed in `fetched_creds/[client_ip]/`

# Contributing

Contributions via pull request, feedback, bug reports are all welcome.
