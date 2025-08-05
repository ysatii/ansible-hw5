# Ansible Роль: vector-role

Данная роль устанавливает и настраивает [Vector](https://vector.dev) — высокопроизводительный инструмент для сбора, трансформации и отправки логов.

##  Возможности

- Создаёт системного пользователя `vector`
- Загружает и устанавливает бинарный файл Vector
- Создаёт необходимые директории
- Разворачивает конфигурационный файл через шаблон
- Обеспечивает исполняемость и установку в систему

##  Переменные роли

Вы можете переопределить переменные в инвентаре или playbook-е.

### Из `defaults/main.yml`

```yaml
vector_download_url: "https://packages.timber.io/vector/latest/vector-x86_64-unknown-linux-gnu.tar.gz"
vector_install_path: "/usr/local/bin/vector"
vector_config_file: "/etc/vector/vector.toml"
```

### Из `vars/main.yml`

```yaml
vector_user: "vector"
```

##  Пример использования

```yaml
- hosts: all
  roles:
    - role: vector-role
```

##  Шаблоны

Роль ожидает наличие Jinja2-шаблона `vector_vector.toml.j2`, используемого для генерации конфигурации Vector.

##  Требования

- Linux
- root-доступ или `become: true`
- Ansible-модули: `get_url`, `unarchive`, `template`, `file`, `user`

##  Лицензия

MIT
