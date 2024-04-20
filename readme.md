## Mailcow January Update 2023 Installation Guide for Unprivileged LXC Container with Proxmox

I have successfully managed to run Mailcow January Update 2022 in an unprivileged LXC container with Proxmox. However, you may need to modify some configurations, such as the maximum number of processes for dovecot docker.

### I. Prepare your Proxmox hypervisor

Log in to your Proxmox hypervisor.

1. **Load overlay and aufs module on Proxmox:**
   
   ```bash
   echo -e "overlay\naufs" >> /etc/modules-load.d/modules.conf
   ```

2. **Install cgroups-mount:**

   ```bash
   apt-get install cgroups-mount
   reboot
   ```

### II. Prepare an unprivileged LXC container

I chose to use `debian-11-standard_11.0-1_amd64` as the CT template.

1. **Create a new container** using the Proxmox GUI. My basic configuration was:

   - **Arch:** amd64
   - **Cores:** 2
   - **Hostname:** mail.domain.wan
   - **Memory:** 6144
   - **Network:**
     ```bash
     net0: name=eth0,bridge=vmbr0,firewall=1,gw=192.168.0.254,hwaddr=B2:F9:44:FB:4E:EC,ip=192.168.0.4/24,type=veth
     ```
   - **OS Type:** debian
   - **Rootfs:** local:102/vm-102-disk-0.raw,size=60G
   - **Swap:** 2048
   - **Unprivileged:** 1

Log in to your VPS (LXC container just created).

2. **Ensure the system is up to date:**

   ```bash
   apt-get update
   apt-get upgrade
   apt-get dist-upgrade
   ```

3. **Configure your timezone:**

   ```bash
   dpkg-reconfigure tzdata
   ```

4. **Remove postfix and use msmtp (an SMTP client) to manage local mail of the container:**

   ```bash
   apt-get purge postfix
   apt-get install msmtp-mta
   ```

5. **Edit the msmtp config file:**

   ```bash
   nano /etc/msmtprc
   ```

   Use the following configuration (replace `USERNAME` and `PASSWORD`):

   ```bash
   #account default
   defaults
   account default
   auth on
   tls on
   tls_starttls on
   tls_trust_file /etc/ssl/certs/ca-certificates.crt
   logfile /var/log/msmtp.log
   host smtp.gmail.com
   port 587
   from USERNAME@gmail.com
   user USERNAME@gmail.com
   password PASSWORD
   aliases /etc/aliases
   ```

6. **Install a command-line mail client:**

   ```bash
   apt-get install bsd-mailx
   ```

7. **Edit your aliases file:**

   ```bash
   nano /etc/aliases
   ```

   Customize your aliases with:

   ```bash
   postmaster: root
   webmaster: root
   root: USERNAME@gmail.com
   local: USERNAME@gmail.com
   default: USERNAME@gmail.com
   ```

8. **Secure your LXC container with an email notification when someone logs into your system.**

   Edit bash config:
   
   ```bash
   nano /etc/bash.bashrc
   ```

   Add this line at the end of the file:

   ```bash
   echo 'ALERT - Shell Access on: ' `date` `who` | mail -s "Alert: Shell Access on `hostname -f`" root
   ```

   If you log out and log in again, you should receive an email with this alert. If not, check `/var/log/msmtp.log` to debug.

### III. Mailcow Installation

Boot your VPS and login again. Follow the installation documentation: [Mailcow - Dockerized Documentation](https://mailcow.github.io/mailcow-dockerized-docs/i_u_m/i_u_m_install/)

At step 5, don’t run `docker-compose up -d`; we need to modify some config before.

1. **Edit /opt/mailcow-dockerized/docker-compose.yml:**

   ```bash
   nano /opt/mailcow-dockerized/docker-compose.yml
   ```

   - **In the `redis-mailcow` section, comment/remove these lines:**

     ```yaml
     # sysctls:
     # - net.core.somaxconn=4096
     ```

     This step is unnecessary since on 5.4 kernel (the case of proxmox 6.4-13), `somaxconn` is already 4096, but docker in LXC doesn't support this option.

   - **In the `dovecot-mailcow` section, modify the nproc limits:**

     ```yaml
     ulimits:
       nproc: 30000 # Instead of 65535