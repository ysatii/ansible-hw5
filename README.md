# Домашнее задание к занятию 5 «Тестирование roles» - Мельник Юрий Александрович

## Подготовка к выполнению

1. Установите molecule и его драйвера: `pip3 install "molecule molecule_docker molecule_podman`.
2. Выполните `docker pull aragast/netology:latest` —  это образ с podman, tox и несколькими пайтонами (3.7 и 3.9) внутри.


## Проверка готовности среды  


![рисунок 1](https://github.com/ysatii/ansible-hw5/blob/main/img/img1.jpg)

## Основная часть

Ваша цель — настроить тестирование ваших ролей. 

Задача — сделать сценарии тестирования для vector. 

Ожидаемый результат — все сценарии успешно проходят тестирование ролей.

### Molecule

1. Запустите  `molecule test -s ubuntu_xenial` (или с любым другим сценарием, не имеет значения) внутри корневой директории clickhouse-role, посмотрите на вывод команды. Данная команда может отработать с ошибками или не отработать вовсе, это нормально. Наша цель - посмотреть как другие в реальном мире используют молекулу И из чего может состоять сценарий тестирования.
2. Перейдите в каталог с ролью vector-role и создайте сценарий тестирования по умолчанию при помощи `molecule init scenario --driver-name docker`.
3. Добавьте несколько разных дистрибутивов (oraclelinux:8, ubuntu:latest) для инстансов и протестируйте роль, исправьте найденные ошибки, если они есть.
4. Добавьте несколько assert в verify.yml-файл для  проверки работоспособности vector-role (проверка, что конфиг валидный, проверка успешности запуска и др.). 
5. Запустите тестирование роли повторно и проверьте, что оно прошло успешно.
5. Добавьте новый тег на коммит с рабочим сценарием в соответствии с семантическим версионированием.

### Решение тестирование с помощью Molecule
1. структура роли 
![рисунок 2](https://github.com/ysatii/ansible-hw5/blob/main/img/img2.jpg)
2. Инициализируем Molecule-сценарий
 ```
cd ~/Рабочий\ стол/ansible-hw4/playbook/roles/vector-role
molecule init scenario --driver-name docker
 ```
3. листинг vector-role/molecule/default/molecule.yml
 ```
 driver:
  name: docker

platforms:
  - name: centos7
    build_image: true
    dockerfile: Dockerfile.j2
    image: centos7-systemd
    privileged: true
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    command: /usr/sbin/init

provisioner:
  name: ansible
  playbooks:
    converge: converge.yml

 ```
4. листинг vector-role/molecule/default/converge.yml
 ```
 ---
 - name: Converge
   hosts: all
   roles:
     - role: vector-role
 ```
5. Пишем тесты в molecule/default/verify.yml
 ```
---
- name: Verify
  hosts: all
  tasks:
    - name: Check if vector binary exists
      stat:
        path: /usr/bin/vector
      register: vector_stat

    - name: Fail if vector binary not found
      fail:
        msg: "Vector binary not found in /usr/bin"
      when: not vector_stat.stat.exists

 ```
6. Запускаем тестирование, Поднимаем контейнеры:
 ```
 molecule create
 ```
7. Удаляем все и запускаем заново
 ```
 molecule destroy
 molecule test
 ```
 ![рисунок 3](https://github.com/ysatii/ansible-hw5/blob/main/img/img3.jpg)

Для успешной прохождения теста сделал следующее
1. листинг  playbook/roles/vector-role/meta/main.yml
 ```
galaxy_info:
  role_name: vector
  namespace: lamer
  author: lamer
  description: Install and configure Vector on CentOS 7
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: EL
      versions:
        - 7
  galaxy_tags:
    - logging
    - vector
dependencies: []
 ```
2. часть кода roles/vector-role/tasks/main.yaml мешало прохождению теста на идемпотентность, скачивался пакет вектор, распаковывался, и устанавливался! теперь проблемы с этим нет! все скачиваеться и перемещаеться в папку для установки, используеться маркер файл при первой распаковке!
```
##########################
#- name: Extract Vector
#  unarchive:
#    src: /tmp/vector.tar.gz
#    dest: /tmp
#    remote_src: true
#    mode: '0755'


- name: Check if Vector already extracted
  stat:
    path: /tmp/vector_extracted
  register: vector_extracted

- name: Extract Vector
  unarchive:
    src: /tmp/vector.tar.gz
    dest: /tmp
    remote_src: true
    mode: '0755'
  when: not vector_extracted.stat.exists

- name: Create marker file after extraction
  file:
    path: /tmp/vector_extracted
    state: touch
  when: not vector_extracted.stat.exists

- name: Find extracted vector binary
  find:
    paths: /tmp
    patterns: "vector"
    recurse: yes
    file_type: file
  register: vector_binary

- name: Fail if vector binary not found
  fail:
    msg: "Vector binary not found in extracted archive"
  when: vector_binary.matched == 0

- name: Move Vector binary to install path
  copy:
    src: "{{ vector_binary.files[0].path }}"
    dest: /usr/bin/vector
    mode: '0755'
    remote_src: true



#- name: Ensure Vector is executable
#  file:
#    path: /usr/local/bin/vector
#    mode: '0755'
#################
```

3. Создан DockerFile centos:7
```
FROM quay.io/centos/centos:7

ENV container docker

# Устанавливаем systemd
RUN yum -y install initscripts systemd sudo && \
    yum clean all && \
    (cd /lib/systemd/system/sysinit.target.wants/; for i in *; do [ $i == systemd-tmpfiles-setup.service ] || rm -f $i; done); \
    rm -f /lib/systemd/system/multi-user.target.wants/*; \
    rm -f /etc/systemd/system/*.wants/*; \
    rm -f /lib/systemd/system/local-fs.target.wants/*; \
    rm -f /lib/systemd/system/sockets.target.wants/*udev*; \
    rm -f /lib/systemd/system/sockets.target.wants/*initctl*; \
    rm -f /lib/systemd/system/basic.target.wants/*; \
    rm -f /lib/systemd/system/anaconda.target.wants/*

VOLUME [ "/sys/fs/cgroup" ]
CMD ["/usr/sbin/init"]
```

4. поправлен файл /vector-role/handlers/main.yaml

```
---
- name: Restart Vector
  systemd:
    name: vector
    state: restarted
  when: ansible_virtualization_type != 'docker'
```


5. Полный вывод логава тестирования
 ![рисунок 4](https://github.com/ysatii/ansible-hw5/blob/main/img/img4.jpg)
 ![рисунок 5](https://github.com/ysatii/ansible-hw5/blob/main/img/img5.jpg)
 ![рисунок 6](https://github.com/ysatii/ansible-hw5/blob/main/img/img6.jpg)
 ![рисунок 7](https://github.com/ysatii/ansible-hw5/blob/main/img/img7.jpg)
 ![рисунок 8](https://github.com/ysatii/ansible-hw5/blob/main/img/img8.jpg)
 ![рисунок 9](https://github.com/ysatii/ansible-hw5/blob/main/img/img9.jpg)
 ![рисунок 10](https://github.com/ysatii/ansible-hw5/blob/main/img/img10.jpg)
 ![рисунок 11](https://github.com/ysatii/ansible-hw5/blob/main/img/img11.jpg)

6. для полного выполнения задания расширим verify.yml
```
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Check if Vector binary exists
      stat:
        path: /usr/bin/vector
      register: vector_stat

    - name: Fail if Vector binary not found
      fail:
        msg: "Vector binary not found in /usr/bin"
      when: not vector_stat.stat.exists

    - name: Check if Vector config exists
      stat:
        path: /etc/vector/vector.toml
      register: vector_config
 


```
и проведем тесты снова ! 
**Роль работает только под операциолнной системой Centos7**
Также заметим что работа приложения в контейнере и на реальной операционной системе отличаються! 
Исходно роли прорабатывались для для операционной системы !
работа сделана на основе предыдущей работы https://github.com/ysatii/ansible-hw4/ 
роль https://github.com/ysatii/ansible-hw5/tree/main/vector-role
Молекула  https://github.com/ysatii/ansible-hw5/tree/main/vector-role/molecule/default

7. Добавим теги
```
git tag -a v1.0.0 -m "First stable version of vector-role with Molecule tests"
git push origin main --tags
```
https://github.com/ysatii/ansible-hw5/tags 





### Tox

1. Добавьте в директорию с vector-role файлы из [директории](./example).
2. Запустите `docker run --privileged=True -v <path_to_repo>:/opt/vector-role -w /opt/vector-role -it aragast/netology:latest /bin/bash`, где path_to_repo — путь до корня репозитория с vector-role на вашей файловой системе.
3. Внутри контейнера выполните команду `tox`, посмотрите на вывод.
5. Создайте облегчённый сценарий для `molecule` с драйвером `molecule_podman`. Проверьте его на исполнимость.
6. Пропишите правильную команду в `tox.ini`, чтобы запускался облегчённый сценарий.
8. Запустите команду `tox`. Убедитесь, что всё отработало успешно.
9. Добавьте новый тег на коммит с рабочим сценарием в соответствии с семантическим версионированием.

После выполнения у вас должно получится два сценария molecule и один tox.ini файл в репозитории. Не забудьте указать в ответе теги решений Tox и Molecule заданий. В качестве решения пришлите ссылку на  ваш репозиторий и скриншоты этапов выполнения задания. 

## Необязательная часть

1. Проделайте схожие манипуляции для создания роли LightHouse.
2. Создайте сценарий внутри любой из своих ролей, который умеет поднимать весь стек при помощи всех ролей.
3. Убедитесь в работоспособности своего стека. Создайте отдельный verify.yml, который будет проверять работоспособность интеграции всех инструментов между ними.
4. Выложите свои roles в репозитории.

В качестве решения пришлите ссылки и скриншоты этапов выполнения задания.

---

### Как оформить решение задания

Выполненное домашнее задание пришлите в виде ссылки на .md-файл в вашем репозитории.
