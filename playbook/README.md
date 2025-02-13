```yaml
---
- name: Install Clickhouse
  hosts: clickhouse
  handlers:
    - name: Start clickhouse service
      become: true
      become_method: su
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
  tasks:
    - block:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/deb/pool/main/c/{{ item }}/{{ item }}_{{ clickhouse_version }}_all.deb"
            dest: "/tmp/temp/{{ item }}_{{ clickhouse_version }}_all_deb"
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Try to get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/deb/pool/main/c/clickhouse-common-static/clickhouse-common-static_{{ clickhouse_version }}_amd64.deb"
            dest: "/tmp/temp/clickhouse-common-static_{{ clickhouse_version }}_amd64.deb"
    - name: Install clickhouse packages
      apt:
        deb: "{{ item }}"
      with_items:
        - /tmp/temp/clickhouse-common-static_{{ clickhouse_version }}_amd64.deb
        - /tmp/temp/clickhouse-client_{{ clickhouse_version }}_all_deb
        - /tmp/temp/clickhouse-server_{{ clickhouse_version }}_all_deb
      notify: Start clickhouse service
    - name: Create database
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0
### download and install vector
- name: Install Vector
  hosts: vector-01
  handlers:
  #Создание обработчика для запуска (первично) и для перезапуска демона
    - name: Restart vector service
      become: true
      #использование привилегий рута
      become_method: su
      #нативно указываем метод
      ansible.builtin.service:
        name: vector
        state: restarted
  tasks:
    - name: Get vector src
    #скачиваем архив, используя метод get_url
      ansible.builtin.get_url:
        url: "https://packages.timber.io/vector/0.25.1/vector-{{ vector_version_tar }}-x86_64-unknown-linux-gnu.tar.gz"
        dest: "/tmp/vector-{{ vector_version_tar }}.tar.gz"
      tags:
        - Get vector
    #Извлекаем файлы из архива, используя метод unarchive
    - name: unarchive vector src
      ansible.builtin.unarchive:
        src: /tmp/vector-{{ vector_version_tar }}.tar.gz
        dest: /tmp/
        remote_src: true
        #Копируем файл в папку /bin, аналогично могли бы глобально поправить переменную PATH
    - name: copy vector to bin
      ansible.builtin.copy:
        src: /tmp/vector-x86_64-unknown-linux-gnu/bin/vector
        dest: /bin/vector
        remote_src: true
        mode: 0755
        #Создаем директорию /etc/vector, где будет располагаться cfg
    - name: create vector dir
      ansible.builtin.file:
        path: /etc/vector
        state: directory
        mode: 0644
        #Используя метод template копируем конфиг vector'a из папки /templates в /etc/vector/vector.toml
    - name: copy vector_cfg from local to remote_src
      template:
        src: vector.toml.j2
        dest: /etc/vector/vector.toml
        mode: 0644
        #Добавляем сервис в /systemd/system
    - name: install as a service and restart
      ansible.builtin.copy:
        src: /tmp/vector-x86_64-unknown-linux-gnu/etc/systemd/vector.service
        dest: /etc/systemd/system
        remote_src: true
        mode: 0644
        #Запускаем handler при наличии изменений
      notify: Restart vector service
```
