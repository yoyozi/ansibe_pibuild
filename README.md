#########################################################
####             PI1.local Standard headless         ####
#########################################################

Built: 2 June 2020
Updated: 17 August 2021
Updated: 3 September 2021

##### OS Linux Raspios for Pi - HEADLESS

##### Image and OS
# Write image to micro SD card: 16-32GB
# Touch (create) the file ssh into the root folder to allow ssh

##### Boot the image on the pi
# Establish the IP and login via ssh

##### Setup basic user and pi user to run ansible to autoconfigure
# SSH: Login as user: pi password: raspberry
# Change the pw for pi
$ passwd (register the pw in the safe storage available)
$ sudo adduser craig
# Add admin user to groups adm and sudo
$ sudo usermod -a -G adm,sudo craig
# Set a static IP, edit the /etc/dhcpcd.conf file:
interface eth0
static ip_address=10.10.77.6/24
static routers=10.10.77.1
static domain_name_servers=10.10.30.1
# Change keyboard layout as was set to uk 
$ sudo vim /etc/default/keyboard (change gb to us)
# Configure Secure ssh
$ sudo raspi-config (interfaces allow ssh)
$ sudo vim /etc/ssh/sshd_config
AllowUsers craig pi
$ sudo systemctl restart sshd (use this to restart stop start service)
# REBOOT

##### Add admin user and generate keys
# On the admin workstation, generate two ssh-keys, 
$ ssh-keygen -f craigkey -t ed25519 -C "Craig_user"
$ ssh-keygen -f mrplodkey -t ed25519 -C "Mrplod_user"
# Copy the admin user key to the pi/server
$ ssh-copy-id -i ~/.ssh/craigkey.pub 10.10.77.7
# Copy mrplod PUBLIC key into bootstrap file

##### Generate file in the files folder:
1. sudoer_mrplod with:
mrplod ALL=(ALL) NOPASSWD: ALL
 
##### Create a config: ansible.cfg
[defaults]
inventory = inventory
private_key_file = ~/.ssh/craiged25519
deprecation_warnings=False
remote_user = craig

##### Generate the inventory file: inventory
10.10.77.7 

##### Test the connection (craig)
$ ansible all --keyfile ~/.ssh/ansible -i inventory -m ping

##### Use the craig user created to run the bootstrap file 
# Bootstrap file should do the following with tags:
1. Upgrade the distros: upgrade
2. Create the user mrplod: anuser
3. Add mrplod to /etc/ssh/sshd_config: anuser
4. Restart sshd: anuser
$ ansible-playbook --ask-become-pass bootstrap.yml 
---

- hosts: all
  become: true
  pre_tasks:

  - name: install updates
    tags: upgrade
    apt:
      upgrade: dist
      update_cache: yes

- hosts: all
  become: true
  tasks:

  - name: create mrplod user
    tags: anuser
    user:
      name: mrplod
      groups: sudo

  - name: add ssh key for mrplod
    tags: anuser
    authorized_key:
      user: mrplod
      key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIJHNnyyMLQUegWvSz4XdevBGjMSr4D+mQTY1q2l/W9y MrPlodED"

  - name: add sudoers file for mrplod
    tags: anuser
    copy:
      src: sudoer_mrplod
      dest: /etc/sudoers.d/mrplod
      owner: root
      group: root
      mode: 0440

  - name: enable login via sshd
    tags: anuser
    lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^AllowUsers'
      line: AllowUsers craig pi mrplod

  - name: restart sshd service after edit
    tags: anuser
    service:
      name: sshd
      state: restarted 

##### Now lets use mrplod: change the ansible.cfg file
[defaults]
inventory = inventory
private_key_file = ~/.ssh/mrploded25519
deprecation_warnings=False
remote_user = mrplod

##### Test the connection (mrplod)
$ ansible all -m ping

############################################################
# site.yml
# tag: pi
############################################################
##### item listing: tags
1. Pi user to ask for sudo pw: 
$ sudo visudo /etc/sudoers.d/010_pi-nopasswd
pi ALL=(ALL) PASSWD: ALL

  - name: pi user must use pw sudo
    tags: pi,piuser
    lineinfile:
      path: /etc/sudoers.d/010_pi-nopasswd
      regexp: '^pi'
      line: "pi ALL=(ALL) PASSWD: ALL"

2. Update repo indexing: uprepo

  - name: update repository index (Ubuntu)
    tags: pi,uprepo
    apt:
      update_cache: yes

3. Change the hostname: pihostname

  - name: change pi hostname (hosts)
    tags: pi,pihostname
    lineinfile:
      path: /etc/hosts
      regexp: 'raspberrypi'
      line: "127.0.1.1      pi01"

  - name: change pi hostname (hostname)
    tags: pi,pihostname
    lineinfile:
      path: /etc/hostname
      regexp: 'raspberrypi'
      line: pi01

4. Install vim: apt_vim

  - name: install vim
    tags: pi,apt_vim
    apt:
      name:
        - vim
      state: latest
    
5. Changed default keyboard layout to us: keyboard

  - name: change default keyboard layout to us
    tags: pi,keyboard
    lineinfile:
      path: /etc/default/keyboard
      regexp: '^XKBLAYOUT'
      line: XKBLAYOUT="us"


6. Disable Wifi: disable_wifi

  - name: disable Wifi
    tags: pi,disable_wifi
    lineinfile:
      path: /boot/config.txt
      line: dtoverlay=disable-wifi

7. Disable bluetooth: disable_bt

  - name: disable bluetooth
    tags: pi,disable_bt
    lineinfile:
      path: /boot/config.txt
      line: dtoverlay=disable-bt

8. Install firewall package ufw: apt_ufw

  - name: Install firewall ufw
    tags: pi,apt_ufw
    apt:
      name:
        - ufw
      state: latest

9. Enable firewall: ufw_defaults

  - name: Configure ufw defaults with community package (enable)
    tags: pi,ufw_defaults
    community.general.ufw:
      state: enabled
      policy: deny
      logging: "on"

  - name: Configure ufw defaults with community package (rules)
    tags: pi,ufw_defaults
    community.general.ufw:
      rule: allow
      port: "{{ item }}"
      proto: tcp
    with_items:
      - "22"

10. Disable firewall if changes needed: ufw_disable (NB no pi)

  - name: Disable ufw (disable)
    tags: ufw_disable
    community.general.ufw:
      state: disabled


11. Install database postgreql: apt_psql

  - name: Install database Postgresql
    tags: pi,apt_psql
    apt:
      name: "{{ item }}"
      update_cache: "yes"
    loop: 
      - postgresql
      - postgresql-contrib

12. Install pip3 for python

  - name: Install Pip for Python
    tags: pi,pypip
    apt:
      name: "{{ item }}"
      update_cache: "yes"
    loop: 
      - python-pip
      - python3-pip

13. Install git: apt_git

  - name: Install git
    tags: pi,apt_git
    apt:
      name: git
      update_cache: "yes"

14. Install tmux: spt_tmux

  - name: Install tmux
    tags: pi,apt_tmux
    apt:
      name: tmux
      update_cache: "yes"





