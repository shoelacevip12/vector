# Ansible Role: vector-role

Роль для автоматизированной установки и настройки [**Vector**](https://vector.dev) — высокопроизводительного агента для сбора, преобразования и отправки логов.

## Описание

Роль выполняет:
- Загрузку и установку Vector указанной версии в `/opt`
- Создание symlink в `/usr/bin/vector`
- Настройку конфигурационного файла `/etc/vector/vector.yaml` через Jinja2-шаблон
- Развёртывание systemd unit-файла для управления службой
- Валидацию конфигурации перед применением изменений
- Запуск и автозагрузку службы

## Структура роли

```
vector-role/
├── defaults/main.yml     # Переменные по умолчанию (источники, sinks)
├── tasks/
│   ├── main.yml          # Обобщенный файл исполнения задач
│   ├── upd_inst.yml      # Установка бинарников
│   ├── upd_dir.yml       # Создание директорий и деплой конфига
│   ├── upd_serv.yml      # Настройка systemd и запуск службы
│   └── upd_verif.yml     # Проверки после установки
├── templates/
│   ├── vector.yaml.j2    # Шаблон конфигурации Vector
│   └── vector.service.j2 # Шаблон systemd unit-файла
├── handlers/main.yml     # Обработчики для перезапуска службы
└── README.md             # Этот файл
```

## Переменные

Основные переменные определяются в `roles/vector-role/defaults/main.yml`:

```yaml
vector_config:
  sources:
    var_logs:
      type: file
      include: ["/var/log/syslog", "/var/log/*.log"]
      data_dir: /var/lib/vector/
      # ... остальные параметры источника

  sinks:
    var_logs_clickhouse:
      type: clickhouse
      inputs: [var_logs]
      endpoint: http://192.168.0.254:8123
      database: skvvectordb
      table: mytable
      auth:
        strategy: basic
        user: skv
        password: 'test1qaz'
      # ... остальные параметры sink
```

> **Важно**: Чувствительные данные (пароли, endpoints) рекомендуется выносить в `vault` или переменные хостов/групп.

## Теги

Роль поддерживает следующие теги для выборочного выполнения:

| Тег | Описание |
|-----|----------|
| `install` | Установка бинарных файлов и зависимостей |
| `config` | Создание директорий и развёртывание конфигурации |
| `service` | Настройка systemd и управление службой |
| `verify` | Проверка статуса службы и валидация конфига |
| `vector` | Общий тег для всех задач роли |

Пример использования:
```bash
ansible-playbook playbook_vector.yaml --tags config,vector
```

## Быстрый старт

1. Подключите роль в свой playbook:
```yaml
- name: Установка Vector
  hosts: vector
  become: true
  roles:
    - vector-role
```

2. Переопределите переменные под вашу инфраструктуру (в `roles/vector-role/defaults/main.yml` или реестре списка машин).

3. Запуск:
```bash
ansible-playbook playbook_vector.yaml
```

## Локальная проверка установки

После выполнения роли можно убедиться в работоспособности:

```bash
# Статус службы
systemctl status vector

# Валидация конфигурации
vector validate /etc/vector/vector.yaml

# Просмотр логов службы
journalctl -u vector -f
```

## Требования

- **Ansible** ≥ 2.9
- **Целевая ОС**: Debian/Ubuntu
- **Архитектура**: x86_64
- **Доступ к интернету** для загрузки Vector

## Безопасность

- Конфигурационный файл создаётся с правами `0640` (`root:root`)
- Для хранения паролей используйте **Ansible Vault**:
  ```bash
  ansible-vault encrypt_string 'my_secret_password' --name 'vector_config.sinks.var_logs_clickhouse.auth.password'
  ```

## Обновление Vector

Для обновления версии:
1. Измените URL и версию в `roles/vector-role/tasks/upd_inst.yml`
2. При необходимости обновите параметры в `roles/vector-role/defaults/main.yml`
3. Запустите роль с тегом `install`:
   ```bash
   ansible-playbook playbook_vector.yaml --tags install
   ```

## Лицензия

MIT

---

> 📌 **Примечание**: Роль устанавливает Vector в `/opt` с созданием системной ссылки. Убедитесь, что путь `/opt` доступен для записи и не управляется другими инструментами.
