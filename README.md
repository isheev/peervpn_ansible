# Описание проекта

Данный playbook включает одну роль VPN, которая выполняет 4 глобальные задачи:

1. Сборка peervpn из исходников.
2. Упаковка бинарного файла в deb-пакет.
3. Настройка http-репозитория.
4. Установка deb-пакета peervpn из репозитория.

Структура файлов проекта:
```
sheyh@host:~/peervpn_ansible$ tree
.
├── ansible.cfg
├── hosts
├── README.md
├── roles
│   └── vpn
│       ├── files
│       │   └── control
│       └── tasks
│           ├── main.yml
│           └── vpn.yml
└── vpn_book.yml
```
Далее идет подробное описание задач с подзадачами:

# 1. Сборка peervpn из исходников.
Обновление информации о пакетах в репозитории.
```yaml
    - name: Update repository
      command: apt-get update --allow-insecure-repositories
 ```
 Установка необходимых пакетов последних версий для компиляции и настройки репозитория. Для предоставления репозитории по HTTP будет использоваться Apache, т.к. для его настроки ничего не требуется.
 ```yaml
     - name: Install base packages for compiling and making repo
      apt: 
        name:
          - make
          - gcc
          - libssl1.0-dev
          - libz-dev
          - apache2
          - dpkg-dev
        state: latest
```
Копирование исходных файлов peervpn в директорию /tmp/peervpn
```yaml
    - name: Clone a peervpn src
      git: repo=https://github.com/peervpn/peervpn.git dest=/tmp/peervpn
```
Компиляция бинарного файла в директории /tmp/peervpn
```yaml
    - name: Make peervpn bin
      command: chdir=/tmp/peervpn/ make
```
# 2. Упаковка бинарного файла в deb-пакет.
Создание директорий для упаковки.
```yaml
    - name: Create directory for deb packages
      file:
        path: "{{ item }}"
        state: directory
        mode: 0775
      loop:
        - /tmp/deb
        - /tmp/deb/usr/local/bin
        - /tmp/deb/DEBIAN
        - /tmp/install
```
Копирование файла control с описанием пакета.
```yaml
    - name: Put control file
      copy: src=control dest=/tmp/deb/DEBIAN mode=0775
```
Копирование подготовленного бинарного файла для пакета.
```yaml
    - name: Copy bin to deb dir
      copy: remote_src=yes src=/tmp/peervpn/peervpn dest=/tmp/deb/usr/local/bin mode=0775
```
Сборка пакета в директорию /tmp/install.
```yaml
    - name: Build Debian package.
      command: dpkg-deb --build /tmp/deb /tmp/install
```
# 3. Настройка http-репозитория.
Создание директории для доступа через http-сервер
```yaml
    - name: Create directory for repo
      file:
        path: /var/www/html/debs/amd64
        state: directory
        mode: 0775
```
Копирование deb-пакета в директорию репозирория.
```yaml
    - name: Copy debs to repo
      copy: remote_src=yes src=/tmp/install/ dest=/var/www/html/debs/amd64 mode=0775
```
Подготовка deb-пакетов в репозитории для пердоставления операционной системе через http.
```yaml
    - name: Scan debs and create packages
      shell: 
        chdir: /var/www/html/debs/ 
        cmd: dpkg-scanpackages amd64 | gzip -9c > amd64/Packages.gz
```
Добавление созданного http-репозитория в sources.list для установки пакетов через apt-get.
```yaml
    - name: Add repo to source list
      lineinfile:
        path: /etc/apt/sources.list
        line: "deb http://localhost/debs/ amd64/"
        state: present
```
Обновление информации о пакетах в репозитории. Используется флаг --allow-insecure-repositories для считывания репозитория без GPG шифрования.
```yaml
    - name: apt update
      command: apt-get update --allow-insecure-repositories
```
# 4. Установка deb-пакета peervpn из http-репозитория.
```yaml
- name: Install peervpn
  apt: name=peervpn state=latest force=yes
```
Проверка на целевом хосте.
```yaml
root@ubuntu-bionic:/home/vagrant# which peervpn
/usr/local/bin/peervpn
root@ubuntu-bionic:/home/vagrant# peervpn
PeerVPN v0.044
(c)2016 Tobias Volk <mail@tobiasvolk.de>

usage: peervpn <configfile>
```


