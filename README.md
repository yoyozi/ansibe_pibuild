##### Starting an ansible server


##### Generate a strong key
$ ssh-keygen -t ed25519 -C "Ansible"
$ ssh-copy-id -i ~/.ssh/id_ed2559.pub 10.10.77.7


##### Setup git repo for this ansible job
$ which git (check is installed)
$ sudo apt install git (if not installed)
# Clone repo 
$ git clone git@github.com:yoyozi/ansibe_pibuild.git
# Set your id and files not to Manager
$ git config --global user.name "Craig Leppan"
$ git config --global user.email "craig@yoyozi.com"


##### Installing ansible (pip or apt)
$ sudo apt update
$ sudo apt install ansible
# GOT errors: ModuleNotFoundError: No module named 'ansible'
$ sudo pip3 install ansible
# and also
$ sudo /usr/local/bin/python3.8 -m pip install --upgrade pip


##### Adding inventory file in the same directory
10.10.77.1
$ git add inventory
$ git commit -d "Added inventory file"
$ git push origin main


#### Run a ping to the files in inventory
$ ansible all --keyfile ~/.ssh/ansible -i inventory -m ping
# GOT:
[DEPRECATION WARNING]: Distribution debian 10.10 on host 10.10.77.7 should use 
/usr/bin/python3, but is using /usr/bin/python for backward compatibility with prior Ansible 
releases. Deprecation warnings can be disabled
 by setting deprecation_warnings=False in ansible.cfg.
10.10.77.7 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}

##### Edited ansible.cfg
deprecation_warnings = False
# GOT:
10.10.77.7 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}


##### Creating ansible.cfg [$ ansible all --keyfile ~/.ssh/ansible -i inventory -m ping]
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible
deprecation_warnings=False

# now we can run
$ ansible -m ping (get the same as above)
$ ansible all --list-hosts
$ ansible all gather_facts
$ ansible all gather_facts --limit 10.10.77.7


##### Runing commands with elevated privaledges
# sudo apt update is:
$ ansible all -m apt -a update_cache=true --become --ask-become-pass
# install vim-nox 
$ ansible all -m apt -a name=vim-nox --become --ask-become-pass
$ ansible all -m apt -a name=tmux --become --ask-become-pass
$ ansible all -m apt -a "name=tmux state=latest"  --become --ask-become-pass
# the state= latest upgrades to latest package
$ ansible all -m apt -a "upgrade=dist" --become --ask-become-pass
# The above upgrades the Distribution


##### Playbooks
$ vim install_apache.yml
---

- hosts: all
  become: true
  tasks:

  - name: install apache2 package
    apt:
      name: apache2 

# and to run 
$ ansible-playbook --ask-become-pass install_apache.yml


# version 2
---

- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes

  - name: install apache2 package
    apt:
      name: apache2


# version 3
---
 
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes

  - name: install apache2 package
    apt:
    name: apache2

  - name: add php support for apache
    apt:
      name: libapache2-mod-php


# version 4
---

- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes

  - name: install apache2 package
    apt:
      name: apache2
      state: latest

  - name: add php support for apache
    apt:
      name: libapache2-mod-php
      state: latest


# Remove all installed packages:
$ cp install_apache.yml remove_apache.yml
---

- hosts: all
  become: true
  tasks:

  - name: remove apache2 package
    apt:
      name: apache2
      state: absent

  - name: remove php support for apache
    apt:
      name: libapache2-mod-php
      state: absent


##### The when Conditional
# change the file so it only runs if apt is present
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: update repository index
     apt:
       update_cache: yes
     when: ansible_distribution == "Ubuntu"
 
   - name: install apache2 package
     apt:
       name: apache2
       state: latest
     when: ansible_distribution == "Ubuntu"
 
   - name: add php support for apache
     apt:
       name: libapache2-mod-php
       state: latest
     when: ansible_distribution == "Ubuntu"


# Now add the other distros part to build the server ---
 
- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes
    when: ansible_distribution == "Ubuntu"

  - name: install apache2 package
    apt:
      name: apache2
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: add php support for apache
    apt:
      name: libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: update repository index
    dnf:
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install httpd package
    dnf:
      name: httpd
      state: latest
    when: ansible_distribution == "CentOS"
 
  - name: add php support for apache
    dnf:
      name: php
      state: latest
    when: ansible_distribution == "CentOS"


##### Refactoring the distros when using "when"
---

- hosts: all
  become: true
  tasks:

  - name: update repository index
    apt:
      update_cache: yes
    when: ansible_distribution == "Ubuntu"

  - name: install apache2 and php packages for Ubuntu
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: update repository index
    dnf:
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install apache and php packages for CentOS
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"


##### Consolidate even more
---

- hosts: all
  become: true
  tasks:

  - name: install apache2 package
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
      update_cache: yes
    when: ansible_distribution == "Ubuntu"

  - name: install httpd package
    dnf:
      name:
        - httpd
        - php
      state: latest
      update_cache: yes
    when: ansible_distribution == "CentOS"


##### And even more: package: RECOGNISES the package manager needed
 ---
 
 - hosts: all
   become: true
   tasks:
 
   - name: install apache and php
     package:
       name:
         - "{{  apache_package  }}"
         - "{{ php_package }}"
       state: latest
       update_cache: yes

# Then change the inventory
10.10.77.6 apache_package=apache2 php_package=libapache2-mod-php
10.10.77.7 apache_package=httpd php_package=php


##### Targetting specific nodes and Site wide servers
---

- hosts: all
  become: true
  tasks:

  - name: install apache and php for Ubuntu servers
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
      update_cache: yes
    when: ansible_distribution == "Ubuntu"

  - name: install apache and php for CentOS servers
    dnf:
      name:
        - httpd
        - php
      state: latest
      update_cache: yes
    when: ansible_distribution == "CentOS"

# and update the inventory
 [web_servers]
 172.16.250.132
 172.16.250.248
 
 [db_servers]
 172.16.250.133
 
 [file_servers]
 172.16.250.134


 ##### Site wide with more variants
 $ cp install_apache.yml site.yml
# site.yml
# Version 1
---

- hosts: all
  become: true
  tasks:

  - name: install updates (CentOS)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    apt:
      upgrade: dist
      update_cache: yes
    when: ansible_distribution == "Ubuntu"


- hosts: web_servers
  become: true
  tasks:

  - name: install apache and php for Ubuntu servers
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: install apache and php for CentOS servers
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"

# site.yml
# Version 2
---

- hosts: all
  become: true
  pre_tasks:

  - name: install updates (CentOS)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    apt:
      upgrade: dist
      update_cache: yes
    when: ansible_distribution == "Ubuntu"


- hosts: web_servers
  become: true
  tasks:

  - name: install apache2 package
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: install httpd package
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"

# site.yml
# Version 3
---

- hosts: all
  become: true
  pre_tasks:

  - name: install updates (CentOS)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    apt:
      upgrade: dist
      update_cache: yes
   when: ansible_distribution == "Ubuntu"


- hosts: web_servers
  become: true
  tasks:

  - name: install httpd package (CentOS)
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"
 
  - name: install apache2 package (Ubuntu)
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

- hosts: db_servers
  become: true
  tasks:

  - name: install httpd package (CentOS)
    dnf:
      name: mariadb
      state: latest
    when: ansible_distribution == "CentOS"
 
  - name: install mariadb server
    apt:
      name: mariadb-server
      state: latest
    when: ansible_distribution == "Ubuntu"

# site.yml
# Version 4
---

- hosts: all
  become: true
  pre_tasks:

  - name: install updates (CentOS)
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    apt:
      upgrade: dist
      update_cache: yes
    when: ansible_distribution == "Ubuntu"


- hosts: web_servers
  become: true
  tasks:

  - name: install httpd package (CentOS)
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"

  - name: install apache2 package (Ubuntu)
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

- hosts: db_servers
  become: true
  tasks:

  - name: install mariadb server package (CentOS)
    dnf:
      name: mariadb
      state: latest
    when: ansible_distribution == "CentOS"

  - name: install mariadb server
    apt:
      name: mariadb-server
      state: latest
    when: ansible_distribution == "Ubuntu"

- hosts: file_servers
  become: true
  tasks:

  - name: install samba package
    package:
      name: samba
      state: latest

#### Ansible Tags
# Adding tags to playbooks allowing us to only run certain tags
---

- hosts: all
  become: true
  pre_tasks:

  - name: install updates (CentOS)
    tags: always
    dnf:
      update_only: yes
      update_cache: yes
    when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    tags: always
    apt:
      upgrade: dist
      update_cache: yes
    when: ansible_distribution == "Ubuntu"


- hosts: web_servers
  become: true
  tasks:

  - name: install httpd package (CentOS)
    tags: apache,centos,httpd
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"

  - name: install apache2 package (Ubuntu)
    tags: apache,apache2,ubuntu
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

- hosts: db_servers
  become: true
  tasks:

  - name: install mariadb server package (CentOS)
    tags: centos,db,mariadb
    dnf:
      name: mariadb
      state: latest
    when: ansible_distribution == "CentOS"

  - name: install mariadb server
    tags: db,mariadb,ubuntu
    apt:
      name: mariadb-server
      state: latest
    when: ansible_distribution == "Ubuntu"

- hosts: file_servers
  tags: samba
  become: true
  tasks:

  - name: install samba package
    tags: samba
    package:
      name: samba
      state: latest


# List the tags available 
$ ansible-playbook --list-tags site_with_tags.yml

# output$ ansible-playbook --list-tags site.yml
[WARNING]: Could not match supplied host pattern, ignoring: web_servers
[WARNING]: Could not match supplied host pattern, ignoring: db_servers
[WARNING]: Could not match supplied host pattern, ignoring: file_servers

playbook: site.yml

  play #1 (all): all	TAGS: []
      TASK TAGS: [always]

  play #2 (web_servers): web_servers	TAGS: []
      TASK TAGS: [apache, apache2, centos, httpd, ubuntu]

  play #3 (db_servers): db_servers	TAGS: []
      TASK TAGS: [centos, db, mariadb, ubuntu]

  play #4 (file_servers): file_servers	TAGS: [samba]
      TASK TAGS: [samba]

# Examples of running a playbook but targeting specific tags
 ansible-playbook --tags db --ask-become-pass site.yml
 ansible-playbook --tags centos --ask-become-pass site.yml
 ansible-playbook --tags apache --ask-become-pass site.yml
