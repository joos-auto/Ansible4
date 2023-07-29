# Ansible4
Практика

**Создание Ролей - Roles**
```
mkdir roles
cd roles/
ansible-galaxy init deploy_apache_web # - называем как хотим, если делаем свое
Получаем папку с директориями
└── deploy_apache_web
    ├── defaults
    │   └── main.yml #  <-- default lower priority variables for this 
    ├── handlers
    │   └── main.yml #  <-- handlers file
    ├── meta
    │   └── main.yml #  <-- role dependencies
    ├── README.md
    ├── tasks
    │   └── main.yml #  <-- tasks file can include smaller files if warranted
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml #  <-- variables associated with this role
```
https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html

defaults/main.yml
```yml
---
# defaults file for deploy_apache_web
destin_folder: /var/www/html
```
handlers/main.yml
```yml
---
# handlers file for deploy_apache_web
- name: Restart Apache Debian
  service: name=apache2 state=restarted
  when: ansible_os_family == "Debian"

- name: Restart Apache RedHat
  service: name=httpd statr=restarted
  when: ansible_os_family == "RedHat"
```
tasks/main.yml
```yml
---
# tasks file for deploy_apache_web
- name: Check and Print Linux-Family
  debug: var=ansible_os_family

- block: # for Debian

  - name: Install Apache Web Server for Debian
    apt: name=apache2 state=latest
  - name: Start WebServer and make it enable for Debian
    service: name=apache2 state=started enabled=yes
  when: ansible_os_family == "Debian"

- name: Generate INDEX.HTML file
  template: src=index.j2 dest={{ destin_folder }}/index.html mode=0555
  notify:
    - Restart Apache Debian
```
cp index.j2 templates/

playbookroles.yml
```yml
---
- name: Install Apache and Upload my Web Page
  hosts: all
  become: yes

  roles:
    - { role: deploy_apache_web, when:ansible_system == 'Linux' } # - запустим только если Linux
```
Запускаем
```
ansible-playbook playbookroles.yml
```

