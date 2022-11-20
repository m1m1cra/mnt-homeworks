# Домашнее задание к занятию "2. Работа с Playbook"

## Подготовка к выполнению

1. (Необязательно) Изучите, что такое [clickhouse](https://www.youtube.com/watch?v=fjTNS2zkeBs) и [vector](https://www.youtube.com/watch?v=CgEhyffisLY)
2. Создайте свой собственный (или используйте старый) публичный репозиторий на github с произвольным именем.
3. Скачайте [playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.
4. Подготовьте хосты в соответствии с группами из предподготовленного playbook.

## Основная часть

1. Приготовьте свой собственный inventory файл `prod.yml`.
```yaml
---
clickhouse:
  hosts:
    clickhouse-01:
      ansible_connection: docker
vector:
  hosts:
    vector-01:
      ansible_connection: docker
```
2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает [vector](https://vector.dev).
#### Я сделал 2 варианта, с использованием готового пакета rpm и с использованием unarchive tar.gz

Установка через tar.gz. В целях уменьшения кода в ответе прикладываю только установку vector, первый play, в котором ставится clickhouse можно посмотреть в исходниках в репозитарии.

```yaml
### download and install vector
- name: Install vector
  hosts: vector-01
  handlers:
    - name: Restart vector service
      become: true
      become_method: su
      ansible.builtin.service:
        name: vector
        state: restarted
  tasks:
    - name: Get vector src
      ansible.builtin.get_url:
        url: "https://packages.timber.io/vector/0.25.1/vector-{{ vector_version_tar }}-x86_64-unknown-linux-gnu.tar.gz"
        dest: "/tmp/vector-{{ vector_version_tar }}.tar.gz"
      tags:
       - Get vector
    - name: unarchive vector src
      ansible.builtin.unarchive:
        src: /tmp/vector-{{ vector_version_tar }}.tar.gz
        dest: /tmp/
        remote_src: yes
    - name: copy vector to bin
      ansible.builtin.copy:
        src: /tmp/vector-x86_64-unknown-linux-gnu/bin/vector
        dest: /bin/vector
        remote_src: yes
        mode: 0755
    - name: create vector dir
      ansible.builtin.file:
        path: /etc/vector
        state: directory
        mode: '0755'
    - name: copy vector_cfg from local to remote_src
      template:
        src: vector.toml.j2
        dest: /etc/vector/vector.toml
    - name: install as a service and restart
      ansible.builtin.copy:
        src: /tmp/vector-x86_64-unknown-linux-gnu/etc/systemd/vector.service
        dest: /etc/systemd/system    
        remote_src: true    
      notify: Restart vector service
```
#### Установка через rpm
```yaml
### download and install vector
- name: Install vector
  hosts: vector-01
  handlers:
    - name: Start vector service
      become: true
      become_method: su
      ansible.builtin.service:
        name: vector
        state: restarted
  tasks:
    - block:
        - name: Get vector distrib
          become: true
          become_method: su
          ansible.builtin.get_url:
            url: "https://packages.timber.io/vector/0.25.1/vector-{{ vector_version }}.x86_64.rpm"
            dest: "/tmp/vector-{{ vector_version }}.x86_64.rpm"
          tags:
            - Get vector
    - name: Install vector
      yum:
        disable_gpg_check: true
        name: "{{ item }}"
        state: present
      with_items:
        - /tmp/vector-{{ vector_version }}.x86_64.rpm
      tags:
        - Install vector
      notify: Start vector service
```
4. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.
#### Использовано
6. Tasks должны: скачать нужной версии дистрибутив, выполнить распаковку в выбранную директорию, установить vector.
```bash
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i inventory/prod.yml site_tar.yml --start-at-task "Get vector src"

PLAY [Install Clickhouse] *****************************************************************************************************************************************************************************************************************************

PLAY [Install vector] *********************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [Get vector src] *********************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [unarchive vector src] ***************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [copy vector to bin] *****************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [create vector dir] ******************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [copy vector_cfg from local to remote_src] *******************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [install as a service and restart] ***************************************************************************************************************************************************************************************************************
ok: [vector-01]

PLAY RECAP ********************************************************************************************************************************************************************************************************************************************
vector-01                  : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook#
```
8. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
```bash
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-lint site_tar.yml
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site_tar.yml
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-lint site_yum.yml
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site_yum.yml
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook#
```
10. Попробуйте запустить playbook на этом окружении с флагом `--check`.
```bash
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i inventory/prod.yml site_tar.yml --check

PLAY [Install Clickhouse] *****************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Get clickhouse distrib] *************************************************************************************************************************************************************************************************************************
changed: [clickhouse-01] => (item=clickhouse-client)
changed: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "/tmp/temp/clickhouse-common-static_22.3.3.44_all_deb", "elapsed": 0, "item": "clickhouse-common-static", "msg": "Request failed", "response": "HTTP Error 404: Not Found", "status_code": 404, "url": "https://packages.clickhouse.com/deb/pool/main/c/clickhouse-common-static/clickhouse-common-static_22.3.3.44_all.deb"}

TASK [Try to get clickhouse distrib] ******************************************************************************************************************************************************************************************************************
changed: [clickhouse-01]

TASK [Install clickhouse packages] ********************************************************************************************************************************************************************************************************************
failed: [clickhouse-01] (item=/tmp/temp/clickhouse-common-static_22.3.3.44_amd64.deb) => {"ansible_loop_var": "item", "changed": false, "item": "/tmp/temp/clickhouse-common-static_22.3.3.44_amd64.deb", "msg": "Unable to install package: E:Could not open file /tmp/temp/clickhouse-common-static_22.3.3.44_amd64.deb - open (2: No such file or directory)"}
failed: [clickhouse-01] (item=/tmp/temp/clickhouse-client_22.3.3.44_all_deb) => {"ansible_loop_var": "item", "changed": false, "item": "/tmp/temp/clickhouse-client_22.3.3.44_all_deb", "msg": "Unable to install package: E:Could not open file /tmp/temp/clickhouse-client_22.3.3.44_all_deb - open (2: No such file or directory)"}
failed: [clickhouse-01] (item=/tmp/temp/clickhouse-server_22.3.3.44_all_deb) => {"ansible_loop_var": "item", "changed": false, "item": "/tmp/temp/clickhouse-server_22.3.3.44_all_deb", "msg": "Unable to install package: E:Could not open file /tmp/temp/clickhouse-server_22.3.3.44_all_deb - open (2: No such file or directory)"}

PLAY RECAP ********************************************************************************************************************************************************************************************************************************************
clickhouse-01              : ok=2    changed=1    unreachable=0    failed=1    skipped=0    rescued=1    ignored=0
```
12. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
```bash
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i inventory/prod.yml site_tar.yml --diff

PLAY [Install Clickhouse] *****************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Get clickhouse distrib] *************************************************************************************************************************************************************************************************************************
changed: [clickhouse-01] => (item=clickhouse-client)
changed: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "/tmp/temp/clickhouse-common-static_22.3.3.44_all_deb", "elapsed": 0, "item": "clickhouse-common-static", "msg": "Request failed", "response": "HTTP Error 404: Not Found", "status_code": 404, "url": "https://packages.clickhouse.com/deb/pool/main/c/clickhouse-common-static/clickhouse-common-static_22.3.3.44_all.deb"}

TASK [Try to get clickhouse distrib] ******************************************************************************************************************************************************************************************************************
changed: [clickhouse-01]

TASK [Install clickhouse packages] ********************************************************************************************************************************************************************************************************************
Selecting previously unselected package clickhouse-common-static.
(Reading database ... 7070 files and directories currently installed.)
Preparing to unpack .../clickhouse-common-static_22.3.3.44_amd64.deb ...
Unpacking clickhouse-common-static (22.3.3.44) ...
Setting up clickhouse-common-static (22.3.3.44) ...
changed: [clickhouse-01] => (item=/tmp/temp/clickhouse-common-static_22.3.3.44_amd64.deb)
Selecting previously unselected package clickhouse-client.
(Reading database ... 7084 files and directories currently installed.)
Preparing to unpack .../clickhouse-client_22.3.3.44_all_deb ...
Unpacking clickhouse-client (22.3.3.44) ...
Setting up clickhouse-client (22.3.3.44) ...
changed: [clickhouse-01] => (item=/tmp/temp/clickhouse-client_22.3.3.44_all_deb)
Selecting previously unselected package clickhouse-server.
(Reading database ... 7095 files and directories currently installed.)
Preparing to unpack .../clickhouse-server_22.3.3.44_all_deb ...
Unpacking clickhouse-server (22.3.3.44) ...
Setting up clickhouse-server (22.3.3.44) ...
changed: [clickhouse-01] => (item=/tmp/temp/clickhouse-server_22.3.3.44_all_deb)

TASK [Create database] ********************************************************************************************************************************************************************************************************************************
ok: [clickhouse-01]

RUNNING HANDLER [Start clickhouse service] ************************************************************************************************************************************************************************************************************
changed: [clickhouse-01]

PLAY [Install Vector] *********************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [Get vector src] *********************************************************************************************************************************************************************************************************************************
changed: [vector-01]

TASK [unarchive vector src] ***************************************************************************************************************************************************************************************************************************
changed: [vector-01]

TASK [copy vector to bin] *****************************************************************************************************************************************************************************************************************************
changed: [vector-01]

TASK [create vector dir] ******************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [copy vector_cfg from local to remote_src] *******************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [install as a service and restart] ***************************************************************************************************************************************************************************************************************
ok: [vector-01]

PLAY RECAP ********************************************************************************************************************************************************************************************************************************************
clickhouse-01              : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0
vector-01                  : ok=7    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook#
```
14. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
```bash
root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook# ansible-playbook -i inventory/prod.yml site_tar.yml --diff

PLAY [Install Clickhouse] *****************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Get clickhouse distrib] *************************************************************************************************************************************************************************************************************************
ok: [clickhouse-01] => (item=clickhouse-client)
ok: [clickhouse-01] => (item=clickhouse-server)
failed: [clickhouse-01] (item=clickhouse-common-static) => {"ansible_loop_var": "item", "changed": false, "dest": "/tmp/temp/clickhouse-common-static_22.3.3.44_all_deb", "elapsed": 0, "item": "clickhouse-common-static", "msg": "Request failed", "response": "HTTP Error 404: Not Found", "status_code": 404, "url": "https://packages.clickhouse.com/deb/pool/main/c/clickhouse-common-static/clickhouse-common-static_22.3.3.44_all.deb"}

TASK [Try to get clickhouse distrib] ******************************************************************************************************************************************************************************************************************
ok: [clickhouse-01]

TASK [Install clickhouse packages] ********************************************************************************************************************************************************************************************************************
ok: [clickhouse-01] => (item=/tmp/temp/clickhouse-common-static_22.3.3.44_amd64.deb)
ok: [clickhouse-01] => (item=/tmp/temp/clickhouse-client_22.3.3.44_all_deb)
ok: [clickhouse-01] => (item=/tmp/temp/clickhouse-server_22.3.3.44_all_deb)

TASK [Create database] ********************************************************************************************************************************************************************************************************************************
ok: [clickhouse-01]

PLAY [Install Vector] *********************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ********************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [Get vector src] *********************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [unarchive vector src] ***************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [copy vector to bin] *****************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [create vector dir] ******************************************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [copy vector_cfg from local to remote_src] *******************************************************************************************************************************************************************************************************
ok: [vector-01]

TASK [install as a service and restart] ***************************************************************************************************************************************************************************************************************
ok: [vector-01]

PLAY RECAP ********************************************************************************************************************************************************************************************************************************************
clickhouse-01              : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=1    ignored=0
vector-01                  : ok=7    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

root@bhdevops:/home/avdeevan/myvar/mnt-homeworks/08-ansible-02-playbook/playbook#
```
16. Подготовьте README.md файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
#### Положил в папку playbook
18. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-02-playbook` на фиксирующий коммит, в ответ предоставьте ссылку на него.
---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
