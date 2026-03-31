# Ansible

#### Задание:
Подготовить стенд на Vagrant с сервером. На этом сервере, используя Ansible необходимо развернуть nginx со следующими условиями:
- необходимо использовать модуль yum/apt
- конфигурационный файлы должны быть взяты из шаблона jinja2 с переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible
##### использование модуля yum/apt
```
administrator@PK:~/Ansible$ cat nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true

  tasks:
    - name: update
      apt:
        update_cache=yes

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest

administrator@PK:~/Ansible$
administrator@PK:~/Ansible$ ansible-playbook nginx.yml
PLAY [NGINX | Install and configure NGINX] **************************************************************************
TASK [Gathering Facts] **********************************************************************************************
[WARNING]: Host 'nginx' is using the discovered Python interpreter at '/usr/bin/python3.10', but future installation of another Python interpreter could cause a different interpreter to be discovered. See https://docs.ansible.com/ansible-core/2.20/reference_appendices/interpreter_discovery.html for more information.
ok: [nginx]
TASK [update] *******************************************************************************************************
changed: [nginx]
TASK [NGINX | Install NGINX] ****************************************************************************************
changed: [nginx]
PLAY RECAP **********************************************************************************************************
nginx                      : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
administrator@PK:~/Ansible$
```
##### Создаем и прописываем конфигурационные файлы из шаблона jinja2 с переменными
```
administrator@PK:~/Ansible$ mkdir templates
administrator@PK:~/Ansible$ vi templates/nginx.conf.j2
administrator@PK:~/Ansible$ cat templates/nginx.conf.j2
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
administrator@PK:~/Ansible$
administrator@PK:~/Ansible$ cat nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true

  tasks:
    - name: update
      apt:
        update_cache=yes

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
- name: NGINX | Create NGINX config file from template
  template:
    src: templates/nginx.conf.j2
    dest: /tmp/nginx.conf
    tags:
      - nginx-configuration

administrator@PK:~/Ansible$
```
##### Используя переменные в ansible включаем порт сайта на 8080
```
administrator@PK:~/Ansible$ vi nginx.yml
administrator@PK:~/Ansible$ cat nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
      tags:
        - nginx-package

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /tmp/nginx.conf
      tags:
        - nginx-configuration

administrator@PK:~/Ansible$
```
##### Переводим nginx в режим enabled в systemd. Создаем handler с добавлением notify для перезагрузки сервиса веб при изменении конфигурации. 
```
administrator@PK:~/Ansible$ vi nginx.yml
administrator@PK:~/Ansible$ cat nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /tmp/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
administrator@PK:~/Ansible$
```
##### После запуска итоговой конфигурации имеем вывод с проверкой на порт 8080:
```
administrator@PK:~/Ansible$
administrator@PK:~/Ansible$ vagrant ssh
Last login: Tue Mar 31 10:32:15 2026 from 10.0.2.2
vagrant@nginx:~$ 
vagrant@nginx:~$ curl http://192.168.11.150:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
vagrant@nginx:~$ exit
logout
administrator@PK:~/Ansible$
```
