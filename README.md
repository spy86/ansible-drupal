Role Name
=========

[![AnsibleTest](https://github.com/spy86/ansible-drupal/actions/workflows/main.yml/badge.svg)](https://github.com/spy86/ansible-drupal/actions/workflows/main.yml) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Ansible Role install and configure Drupal CMS

Requirements
------------
 - Ubuntu

Role Variables
--------------

Configuration file is splited to 4 sections:

- `drupal_core_version` : The core version you want to use (e.g. 8.5.x, 8.6.x).
- `drupal_core_path` : The path where Drupal will be downloaded and installed.
- `domain` : The resulting domain will be [domain].test (with .test appended).
- `drupal_site_name` : Your Drupal site name.

Dependencies
------------

None

Example Playbook
----------------
```YAML
---
- hosts: all

  vars_files:
    - vars.yml

  pre_tasks:
    - name: Update apt cache if needed.
      apt: update_cache=yes cache_valid_time=3600

  handlers:
    - name: restart apache
      service: name=apache2 state=restarted

  tasks:
    - name: Get software for apt repository management.
      apt: "name={{ item }} state=present"
      with_items:
        - python-apt
        - python-pycurl

    - name: Add ondrej repository for later versions of PHP.
      apt_repository: repo='ppa:ondrej/php' update_cache=yes

    - name: "Install Apache, MySQL, PHP, and other dependencies."
      apt: "name={{ item }} state=present"
      with_items:
        - git
        - curl
        - unzip
        - sendmail
        - apache2
        - php7.1-common
        - php7.1-cli
        - php7.1-dev
        - php7.1-gd
        - php7.1-curl
        - php7.1-json
        - php7.1-opcache
        - php7.1-xml
        - php7.1-mbstring
        - php7.1-pdo
        - php7.1-mysql
        - php-apcu
        - libpcre3-dev
        - libapache2-mod-php7.1
        - python-mysqldb
        - mysql-server

    - name: Disable the firewall (since this is for local dev only).
      service: name=ufw state=stopped

    - name: "Start Apache, MySQL, and PHP."
      service: "name={{ item }} state=started enabled=yes"
      with_items:
        - apache2
        - mysql

    - name: Enable Apache rewrite module (required for Drupal).
      apache2_module: name=rewrite state=present
      notify: restart apache

    - name: Add Apache virtualhost for Drupal 8.
      template:
        src: "templates/drupal.test.conf.j2"
        dest: "/etc/apache2/sites-available/{{ domain }}.test.conf"
        owner: root
        group: root
        mode: 0644
      notify: restart apache

    - name: Symlink Drupal virtualhost to sites-enabled.
      file:
        src: "/etc/apache2/sites-available/{{ domain }}.test.conf"
        dest: "/etc/apache2/sites-enabled/{{ domain }}.test.conf"
        state: link
      notify: restart apache

    - name: Remove default virtualhost file.
      file:
        path: "/etc/apache2/sites-enabled/000-default.conf"
        state: absent
      notify: restart apache

    - name: Adjust OpCache memory setting.
      lineinfile:
        dest: "/etc/php/7.1/apache2/conf.d/10-opcache.ini"
        regexp: "^opcache.memory_consumption"
        line: "opcache.memory_consumption = 96"
        state: present
      notify: restart apache

    - name: Remove the MySQL test database.
      mysql_db: db=test state=absent

    - name: Create a MySQL database for Drupal.
      mysql_db: "db={{ domain }} state=present"

    - name: Create a MySQL user for Drupal.
      mysql_user:
        name: "{{ domain }}"
        password: "1234"
        priv: "{{ domain }}.*:ALL"
        host: localhost
        state: present

    - name: Download Composer installer.
      get_url:
        url: https://getcomposer.org/installer
        dest: /tmp/composer-installer.php
        mode: 0755

    - name: Run Composer installer.
      command: >
        php composer-installer.php
        chdir=/tmp
        creates=/usr/local/bin/composer

    - name: Move Composer into globally-accessible location.
      shell: >
        mv /tmp/composer.phar /usr/local/bin/composer
        creates=/usr/local/bin/composer

    - name: Check out drush 8.x branch.
      git:
        repo: https://github.com/drush-ops/drush.git
        version: 8.x
        dest: /opt/drush

    - name: Install Drush dependencies with Composer.
      shell: >
        /usr/local/bin/composer install
        chdir=/opt/drush
        creates=/opt/drush/vendor/autoload.php

    - name: Create drush bin symlink.
      file:
        src: /opt/drush/drush
        dest: /usr/local/bin/drush
        state: link

    - name: Check out Drupal Core to the Apache docroot.
      git:
        repo: http://git.drupal.org/project/drupal.git
        version: "{{ drupal_core_version }}"
        dest: "{{ drupal_core_path }}"

    - name: Install Drupal dependencies with Composer.
      shell: >
        /usr/local/bin/composer install
        chdir={{ drupal_core_path }}
        creates={{ drupal_core_path }}/vendor/autoload.php

    - name: Install Drupal.
      command: >
        drush si -y --site-name="{{ drupal_site_name }}"
        --account-name=admin
        --account-pass=admin
        --db-url=mysql://{{ domain }}:1234@localhost/{{ domain }}
        --root={{ drupal_core_path }}
        creates={{ drupal_core_path }}/sites/default/settings.php
      notify: restart apache

    # SEE: https://drupal.org/node/2121849#comment-8413637
    - name: Set permissions properly on settings.php.
      file:
        path: "{{ drupal_core_path }}/sites/default/settings.php"
        mode: 0744

    - name: Set permissions properly on files directory.
      file:
        path: "{{ drupal_core_path }}/sites/default/files"
        mode: 0777
        state: directory
        recurse: yes
```
