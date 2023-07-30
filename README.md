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
**Внешние переменные - extra-vars**
```yml
---
- name: Install Apache and Upload my Web Page
  hosts: "{{ MYHOSTS }}"
  become: yes

  roles:
    - { role: deploy_apache_web, when:ansible_system == 'Linux' }
```
Запускаем
```
ansible-playbook playbookvars.yml --extra-vars "MYHOSTS=server1"
ansible-playbook playbookvars.yml --extra-var "MYHOSTS=server1"
ansible-playbook playbookvars.yml --extra-var "MYHOSTS=server2 owner=DENIS!" # - перепишет owner, так как высший приоритет
```
**Использование Import, Include**

Выносим логику в отдельные файлы и подключаем в основном
```
include: create_folder.yml
```

**Import** - сразу все объединяет и выполняет

**Include** - когда видит файл, тогда идет и выполняет его
```yml
---
- name: My Playbook
  hosts: all
  become: yes

  vars:
    mytext: "Hello from Joos"

  tasks:
  - name: Ping test
    ping:

  - name: Create folders
    include: create_folder.yml

  - name: Create files
    include: create_file.yml mytext="Prosto Hello"
```
create_folder.yml
```yml
---
- name: Create folder1
  file:
    path: /home/secret/folder1
    state:  directory
    mode: 0755

- name: Create folder2
  file:
    path: /home/secret/folder2
    state:  directory
    mode: 0755
```
create_file.yml
```yml
---
- name: Create file1
  copy:
    dest: /home/secret/file1.txt
    content: |
      Text Line1, in file1
      Text Line2, in file1
      Text Line3, {{ mytext }}

- name: Create file2
  copy:
    dest: /home/secret/file2.txt
    content: |
      Text Line1, in file2
      Text Line2, in file2
      Text Line3, {{ mytext }}
```
**Перенаправление выполнения Task из Playbook на определённый сервер - delegate_to**
```
delegate_to: 127.0.0.1 # - выполнится только на мастер хосте
```
playbook-delegate.yml
```yml
---
- name: Play it delegate
  hosts: all
  become: yes

  vars:
    mytext: "Prinvet ot Joos"

  tasks:
    - name: Ping test
      ping:

    - name: Unregister Server from Load Balancer
      shell: echo This sever {{ inventory_hostname }} was deregistered from our Load Balancer,  nodename is {{ ansible_nodename }} >> /home/joos/log.txt # - запишем вывод в лог
      delegate_to: 127.0.0.1 # - выполнится и запишет лог только на мастер хосте

    - name: Update my Database
      shell: echo UPDATING Database...
      run_once: true  # - запустит только 1 раз на 1 хосте

    - name: Create file1
      copy:
        dest: /home/joos/file1.txt
        content: |
          This is FileN1
          On ENGLISH Hello World
          On RUSSIAN {{ mytext }}
      delegate_to: server1 # - выполнится только на server1

    - name: Create file2
      copy:
        dest: /home/joos/file2.txt
        content: |
          This is FileN2
          On ENGLISH Hello World
          On RUSSIAN {{ mytext }}

    - name: Reboot my servers; # - Перезагружаем сервера
      shell: sleep 3 && reboot now - # - ждем 3 секунды и перезагружаем
      async: 1
      poll: 0 # - кинули команду и не ждем ответа

    - name: Wait till my server will come up online
      wait_for:
        host: "{{ inventory_hostname}}"
        state: started
        delay: 5 # - ждем 5 секунд и проверяем
        timeout: 40 # - ждем максимум 40 секунд
      delegate_to: 127.0.0.1 # - ждем только на мастере (сервера в это время перезагружаются)
```
**Перехват и Контроль ошибок**
```
any_errors_fatal: true # - если ошибка то останавливаем выполнение на всех серверах
ignore_errors: yes # - если ошибка то игнорируем и выполняем таски дальше
failed_when: "'World' in results.stdout" # - Ошибка если в выводе есть слово World
failed_when: results.rc != 0 # - ошибка если return code не равен 0
```
plabook-error.yml
```yml
---
- name: Ansible and Error
  hosts: all
  any_errors_fatal: true # - если ошибка то останавливаем выполнение на всех серверах
  become: yes

  tasks:
    - name: task Number1
      apt: name=treeee state=latest
      ignore_errors: yes # - если ошибка то игнорируем и выполняем таски дальше

    - name: task Number2
      shell: echo Hello World
      register: results
#      failed_when: "'World' in results.stdout" # - Ошибка если в выводе есть слово World
      failed_when: results.rc != 0 # - ошибка если return code не равен 0

    - debug:
        var: results

    - name: read file
      shell: cat /home/joos/file1.txt # - на первом сервера этого файла нет - он отвалится и не выполнит следующую таску, но если any_errors_fatal: true то остановится выполнение
      register: myresult

    - name: task Number3
      shell: echo Hi Mir!!
```
**Хранение Секретов - ansible-vault**
```
ansible-vault create mysec.txt
ansible-vault view mysec.txt
ansible-vault edit mysec.txt
ansible-vault rekey mysec.txt - смена пароля
ansible-vault encrypt playbook-secret.yml - шифруем целый файл
ansible-vault decrypt playbook-secret.yml - расшифровываем целый файл
ansible-playbook playbook-secret.yml --ask-vault-pass - запускаем с запросом пароля
ansible-playbook playbook-secret.yml --vault-password-file mypass.txt - кладем пароль в файл и запускаем указывая файл
ansible-vault encrypt_string - запускаем и вводим строку которую надо зашифровать, !vault | - копируем все и ставим вместо пароля
ansible-vault encrypt_string --stdin-name "password" - как и выше, просто напишет перед блоком !vault - passwortd:
echo -n "tut_my_parol" | ansible-vault encrypt_string - сразу выводит зашифрованный tut_my_parol после того как мы введи vault пароль
```
**Dynamic Inventory AWS**

https://docs.ansible.com/ansible/latest/inventory_guide/intro_dynamic_inventory.html



