# Installing Matomo Analytics software

Matomo is an opensource google analytics style solution.  As Im running the tech howto wiki now, I want to see what sort of traffic (if any) I get on the site.  So instead of using google analytics I thought I would go with Matomo.

In the style of automation, im going to do this as a playbook.  Right from scratch (well almost from scratch - should really terraform the infra build, and then do CICD with it, but thats an expansion for another time), im going to set this up with Ansible.

Ive actually built a Digital Ocean Droplet, just a small $5 one, which im hoping will be big enought to take the matomo application.

I need an upto date ubuntu server, so the droplet Ive built is a 20.04 droplet.

There are a few items that need to be setup, 

 * Webserver such as Apache, Nginx, IIS, LiteSpeed, etc.
 * Matomo 4.x requires PHP version 7.2.5 or greater. Matomo fully works with PHP 8 as well. (the older Matomo 3.x required PHP version 5.5.9 or PHP 7.x)
 * MySQL version 5.5 or greater, or MariaDB
 * (enabled by default) PHP extension pdo and pdo_mysql, or the mysqli extension.

Which is fairly standard.  With that in mind, let me create an inventory that holds the IP and details for the user to access the server, and then start with the playbook.

## Inventory file.

Nothing fancy at this stage, just a normal YAML file with the hosts in it.

```yaml
do:
  hosts:
    analytics:
      ansible_host: 138.68.131.170
      ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
      ansible_user: root
      host_key_checking: false

    wiki:
      ansible_host: 142.93.34.244
      ansible_ssh_common_args: '-o StrictHostKeyChecking=no'
      ansible_user: root
      host_key_checking: false
```
<pre>
Line 1   : name for the overall set of hosts, DO for digital Ocean in this case.
Line 2   : Grouping for the hosts.
Line 3-7 : The host 
</pre>  

Remote access to the hosts is via SSH keys only.  The key is set up at deployment time and is not included here as its loaded into my SSH Agent keyring. 

A quick check of the inventory file with `ansible -i inventory -m ping all` shows the two hosts are accessible and we can connect.   
  
Next need a playbook to do the base OS upgrade to ensure all applications installed by default are up to date.

This is taken from https://www.redpill-linpro.com/sysadvent/2017/12/24/ansible-system-updates.html and modified to suit my setup.

```yaml
---
#
# Install package updates on servers
#

- name: upgrade packages and reboot (if necessary)
  become: true
  hosts: all
  tasks: # tasks are done in order
    # do an "apt-get update", to ensure latest package lists
    - name: apt-get update
      apt:
        update-cache: yes
      changed_when: 0

    # do the actual apt-get dist-upgrade
    - name: apt-get dist-upgrade
      apt:
        upgrade: dist # upgrade all packages to latest version
      register: upgrade_output

    # if a new kernel is incoming, remove old ones to avoid full /boot
    - name: apt-get autoremove
      command: apt-get -y autoremove
      args:
        warn: false
      changed_when: 0

    # check if we need a reboot
    - name: check if reboot needed
      stat: path=/var/run/reboot-required
      register: file_reboot_required

    # "meta: end_play" aborts the rest of the tasks in the current «tasks:»
    # section, for the current server
    # "when:" clause ensures that the "meta: end_play" only triggers if the
    # current webserver does _not_ need a reboot
    - meta: end_play
      when: not file_reboot_required.stat.exists

    # because of the above meta/when we at this point know that the current
    # host needs a reboot

    - name: reboot node
        shell: sleep 2 && shutdown -r now "Reboot triggered by ansible"
        async: 1
      poll: 0
      ignore_errors: true

    # poll ssh port until we get a tcp connect
    - name: wait for node to finish booting
      wait_for_connection:
        delay: 60
        timeout: 300

    #- name: waiting for SSH to be in a sensible state
    #  wait_for:
    #    timeout: 120
    #    delay: 10
    #    port: 22
    #    host: "{{ inventory_hostname }}"
    #  delegate_to: localhost
    
    # wait a few minutes between hosts, unless we're on the last
    #- name: waiting between hosts
    #  pause:
    #    minutes: 10
    #  when: inventory_hostname != ansible_play_hosts[-1]

  ```
  
While the above does work, there are areas where it trips up.  The last section where its waiting for SSH to settle down and be available does not seem to work properly, all the time. 

Anyway, with the main servers updated and at the latest OS releases (well package fixes anyway) we can look to update a few other areas.

1) Setup so security updates only are done, not full system updates with the autoupdate script.
2) install the required pre-requisite packages on the analytics server
3) install the Matomo software.

Lets tackle 1 first.

## Configure unattended upgrades to only do security updates.

Im actually in two minds as to this being a good idea or not.  On one hand, keeping systems up to date is always a good idea.  However, letting a server blindly update packages, even if they are security patches, is a real bad idea and opens up to losing access to the server, or worse, totaling the server should an update go wrong. 

By default, on a Ubuntu server, automatic updates are enabled.   The following playbook will walk through how to install and set them up, should it not be done automatically.  The playbook after that will walk through disabling and removing automatic updates.

```yaml
---
#
# Setup automatic updates
#
- name: Install and setup unattended upgrades
  become: true
  hosts: all
  tasks:
    # do a quick apt update to bring cache uptodate
    - name: Apt Update
      apt: 
        update-cache: yes
    
    # install unattended updates now.
    - name: Install unattended updates package
      apt: 
        package: unattended-upgrades
        state: present

    # configure the config file to use the right email address, and other settings.    
    - name: configure email address to send notifications to.
      ansible.builtin.lineinfile: 
        path: /etc/apt/apt.conf.d/50unattended-upgrades
        regexp: '\/\/Unattended-Upgrade\:\:Mail \"\"\;'
        line: 'Unattended-Upgrade::Mail "unattended-upgrades@techize.co.uk";'

    - name: configure email notifications to be sent always.
      ansible.builtin.lineinfile: 
        path: /etc/apt/apt.conf.d/50unattended-upgrades
        regexp: '\/\/Unattended-Upgrade\:\:MailReport \"on-change\"\;'
        line: 'Unattended-Upgrade::MailReport "always";'

    - name: configure unused dependancies to be removed
      ansible.builtin.lineinfile: 
        path: /etc/apt/apt.conf.d/50unattended-upgrades
        regexp: '\/\/Unattended-Upgrade\:\:Remove-Unused-Dependencies\"false\"\;'
        line: 'Unattended-Upgrade::Remove-Unused-Dependencies "true";'

    - name: Verbose logging
      ansible.builtin.lineinfile: 
        path: /etc/apt/apt.conf.d/50unattended-upgrades
        regexp: '\/\/Unattended-Upgrade\:\:Verbose\"false\"\;'
        line: 'Unattended-Upgrade::Verbose "true";'


```
So thats how we get Unattended upgrades set up and working.  Dont forget to change the email address used, no point sending me all the details of your unattended upgrades!

Now we can look to disabling / removing unattended upgrades.  This one is straight forward.  Do the uninstall, and purge all bits so its no longer available.

```yaml
---
#
# Remove unattended updates
#
- name: Remove unattended upgrades
  become: true
  hosts: all
  tasks:
    # do a quick apt update to bring cache uptodate, granted we dont actually need to do this, but good habit to get into I feel. 
    - name: Apt Update
      apt: 
        update-cache: yes
    - name: remove the unattended updates package
      apt:
        name: unattended-updates
        state: absent
    - name: purge cache
      apt:
        autoclean: yes
    - name: purge unused dependancies
      apt:
        autoremove: yes    
    - name: ensure the configuration for unattended upgrades is removed.
      file:
        state: absent
        path: /etc/apt/apt.conf.d/50unattended-upgrades
    - name: remove the unattended upgrades line from auto-upgradres
      ansible.builtin.lineinfile:
        path: /etc/apt/apt.conf.d/20auto-upgrades
        regexp: '^APT\:\:Periodic\:\:Unattended-Upgrade';
        state: absent
```



  
Choose whichever one suites your purposes best. It will be a mix of the two, sometimes its best to have the unattended updates enabled, others, do the management through other means.

With those completed, lets move onto the actual install of the prerequisites now.

## Matomo Analytics pre-requisites.

Matomo has a few pre-requisites, namely a database and php.  These can be easily installed with the following playbook.  At the same time im going to set up the database needed with a dedicated user for Matomo with the right permissions.

```yaml
---

#
# Setup pre-reqs
#
- name: Install pre-requisites
  become: true
  hosts: all
  tasks:
    # do a quick apt update to bring cache uptodate
    - name: Apt Update
      apt: 
        update-cache: yes
    
    # install pre-reqs now.
    - name: Install pre-requisites
      apt: 
        package: 
        - php
        - php-curl
        - php-gd
        - php-cli
        - mysql-server
        - mysql-client
        - php-mysql
        - php-xml
        - php-mbstring
        - python3-pymysql
        - php7.4-fpm
        - php-pear
        state: present

```

Next is to get the database setup. For this im going to make use of the ansible modules for db access. But first need to set up a root password.

Please make sure a vault.yml file with an ansible-vault encrypted file has been set up. For this I have used the following

```yaml
---
# mysql settings
mysql_root_pass: "<root password goes in here>"
```

That file is dropped into the vars folder as vault.yml.

When the playbook is now run, specify the --ask-vault-passsword flag on the command line to prompt for the vault password.

```yaml
---
#
# Setup the db
#
- name: Set the root password up.
  become: true
  hosts: all
  tasks:
  - name: Include only files matching vault.yml (2.2)
    include_vars:
      dir: ../vars
      files_matching: vault.yml
  
# This command will exit non-zero if the root password was set previously
  - name: Check if root password is unset
    shell: >
      mysql -u root
      -p'{{ mysql_root_pass }}'
      -h localhost
      -S /var/run/mysqld/mysqld.sock
      -e "quit"
    changed_when: false
    ignore_errors: true
    register: root_pwd_check

# Repeat runs with the same password can continue idempotently, otherwise fail.
  - name: Check if the specified root password is already set
    shell: >
      mysqladmin -u root -p{{ mysql_root_pass }} status
    changed_when: false
    when: root_pwd_check.rc == 0

  - name: Set MySQL root password for the first time (root@localhost)
    mysql_user:
      login_host: 'localhost'
      login_user: 'root'
      login_unix_socket: /var/run/mysqld/mysqld.sock
      #login_password: ''
      name: 'root'
      password: '{{ mysql_root_pass }}'
      state: present
    when: root_pwd_check.rc != 0

- name: Configure the matomo database  
  become: true
  hosts: all
  tasks:
  - name: Include only files matching vault.yml (2.2)
    include_vars:
      dir: ../vars
      files_matching: vault.yml
  
  - name: create database
    mysql_db:
      name: matomo
      login_host: 'localhost'
      login_user: 'root'
      login_unix_socket: /var/run/mysqld/mysqld.sock
      login_password: '{{ mysql_root_pass }}'
  
  - name: create matomo user
    mysql_user:
      login_host: 'localhost'
      login_user: 'root'
      login_unix_socket: /var/run/mysqld/mysqld.sock
      login_password: '{{ mysql_root_pass }}'
      name: matomo
      password: '{{ matomo_mysql_pass }}'
      state: present
      priv: 'matomo.*:ALL'
  - name: Grant file privilege as well.
    mysql_user:
      login_host: 'localhost'
      login_user: 'root'
      login_unix_socket: /var/run/mysqld/mysqld.sock
      login_password: '{{ mysql_root_pass }}'
      name: matomo
      append_privs: yes
      state: present
      priv: '*.*:FILE'
```

A couple of things to note.  The privilege setting is tempermental.  The file privilege cannot be applied at the same time as the ALL privileges, unless you apply all to all dbs (not whats wanted really).  Also note that a socket connection and not IP connection needs to be used as MySQL is not listening to the main network interface but only localhost.

At last the database is set up, so we can proceed with the web server and ssl setup.

## Remove Apache

As part of the initial install, Apache was installed.  For this project we want to use Nginx, so Apache needs to be removed.  

The following playbook does the clean up for apache.  Simply removes the packages and then removes any lingering configuration aspects, as well as the default web structure.  **Dont run this on a live system that has a web structure, it will delete it!**

```yaml
---
#
# Remove Apache2 from the system.  Will add Nginx with a different playbook later.
#
- name: Remove Apache from the server
  become: true
  hosts: all # restricted by the inventory
  tasks:
  - name: remove apache
    apt:
      name: apache2
      state: absent
  - name: purge cache
    apt:
      autoclean: yes
  - name: purge unused dependancies
    apt:
      autoremove: yes    

  - name: remove files for config and web.
    file:
      state: absent
      path: /var/www
  - name: remove files for config and web.
    file:
      state: absent
      path: /etc/apache2
```

# Installing Nginx and Lets Encrypt certificates.

For this part, I started to setup playbooks to do the bits, but then thought there must be a better way to do all of this, and someone will have done it,  I was right, <https://github.com/jibiabraham/ansible-letsencrypt-nginx> is a fantastic set of playbooks and documentation that walks through setting things up.

Im going to use most of the bits from these playbooks to set this up for myself.

Im using the same inventory as before, that does not need to change.

This time round, roles are used to set up the peices.

While it is best to actually use the roles at the git repo listed (or the one set up for this howto), I'll duplicate things here a little.

The tasks we need to perform are as follows;

* Install a repo for Nginx
* Install Nginx itself
* ensure its setup to start at boot, and is started now.
* If UFW is used (it should be) enable Nginx to pass through the firewall.


```yaml
---
# tasks file for nginx
- name: Add Nginx Repository
  apt_repository:
    repo: ppa:nginx/stable
    state: present

- name: Install Nginx
  apt:
    pkg: nginx
    state: present
    update_cache: true
  notify:
    - Start Nginx

- name: Enable ufw access for Nginx Full
  ufw:
    rule: allow
    name: "Nginx Full"

```
This is run from the following playbook.

```yaml
- hosts: analytics   # this time ive restricted the host
  become: true
  gather_facts: False

  roles:
    - role: nginx
```

Thats Nginx installed, next lets look at getting an SSL certificate setup.  For this the playbook and role from Jibi Abraham (<https://github.com/jibiabraham>) is used as before. Here is the top level playbook that is run that calls the letsencrypt role.

```yaml

- hosts: analytics
  gather_facts: true
  become: true

  roles:
    - role: letsencrypt
      vars:
        letsencrypt_email: spam@techize.co.uk
        main_domain_name: analytics.techize.co.uk
        all_domain_names:
          - analytics.techize.co.uk
          # - www.example.com include as many more SAN as needed
        deploy_sample_html: true
```

You will need to set the vars to fit your needs. Also note the main_domain_name is also included in the all_domain_names, along with any other domain names you need. 

Now you can test this out by visiting the website seeing if you get a nice fancy index page and if its ssl or not. If it is, then one more step closer to getting the analytics software working!

## Installing Matomo and configuring automatic archive reporting

This is the final step for installing Matomo.  You do need to do manual configuration once the install is complete, but the online docs run through that process really well.  

So lets get on with the playbook to install matomo!

```yaml
---
#
# Download and install the Matomo application.
#
# we know from previous setup that the install directory is /var/www/<domain>/ so thats where we want to unpack files.
#
- name: Download and install Matomo
  become: true
  hosts: analytics
  tasks:
  - name: Download zip file.
    get_url: 
      url: https://builds.matomo.org/matomo.zip
      dest: /tmp/matomo.zip

  # now unzip the file in /tmp and then move all files in the /tmp/matomo directory created into the web root.
  
  - name: unzip downloaded matomo file.
    unarchive:
      src: /tmp/matomo.zip
      dest: /tmp
      remote_src: yes      

  - name: copy files to web root
    copy:
      remote_src: yes
      src: /tmp/matomo/
      dest: /var/www/analytics.techize.co.uk/   
```      
Now lets set some permissions on directories and then remove the index.html thats present

```yaml
---
#
# Change ownership and permissions on some files and folders
#
- name: Update Permissions
  become: true
  hosts: analytics
  tasks:
  - name: Update permissions
    ansible.builtin.shell: |
      chown -R www-data:www-data /var/www/analytics.techize.co.uk
      find /var/www/analytics.techize.co.uk/tmp -type f -exec chmod 644 {} \;
      find /var/www/analytics.techize.co.uk/tmp -type d -exec chmod 755 {} \;
      find /var/www/analytics.techize.co.uk/tmp/assets/ -type f -exec chmod 644 {} \;
      find /var/www/analytics.techize.co.uk/tmp/assets/ -type d -exec chmod 755 {} \;
      find /var/www/analytics.techize.co.uk/tmp/cache/ -type f -exec chmod 644 {} \;
      find /var/www/analytics.techize.co.uk/tmp/cache/ -type d -exec chmod 755 {} \;
      find /var/www/analytics.techize.co.uk/tmp/logs/ -type f -exec chmod 644 {} \;
      find /var/www/analytics.techize.co.uk/tmp/logs/ -type d -exec chmod 755 {} \;
      find /var/www/analytics.techize.co.uk/tmp/tcpdf/ -type f -exec chmod 644 {} \;
      find /var/www/analytics.techize.co.uk/tmp/tcpdf/ -type d -exec chmod 755 {} \;
      find /var/www/analytics.techize.co.uk/tmp/templates_c -type f -exec chmod 644 {} \;
      find /var/www/analytics.techize.co.uk/tmp/templates_c -type d -exec chmod 755 {} \;  
  - name: Remove index.html
    file:
      state: absent
      path: /var/www/analytics.techize.co.uk/index.html
```

You should now be good to go!  Matomo is installed and secured as it should be. 

The entire repo can be found <https://github.com/techize/matomo>



