 12.8. Резервное копирование баз данных — Зарубов Николай

## Задание 1. Резервное копирование

### Кейс

Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования.

### 1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день

**Решение:** Ежедневное полное резервное копирование (Full Backup)

**Описание стратегии:**
- Выполнять полное резервное копирование базы данных раз в сутки, желательно в период минимальной нагрузки (например, ночью)
- Хранить копии минимум за последние 7-30 дней в зависимости от требований бизнеса
- Использовать автоматизацию через планировщики задач (cron в Linux, Task Scheduler в Windows)

**Преимущества:**
- Простота восстановления — требуется только одна резервная копия
- Гарантированное восстановление данных на начало предыдущего дня
- Отсутствие зависимостей между копиями

**Недостатки:**
- Большой объём занимаемого пространства
- Длительное время создания резервной копии для больших баз данных
- Потеря данных за текущий день при восстановлении

**Пример для PostgreSQL:**
```bash
# Ежедневный скрипт резервного копирования
pg_dump -U postgres -Fc mydatabase > /backups/full_backup_$(date +%Y%m%d).dump
```

**Пример для MySQL:**
```bash
# Полное резервное копирование с указанием даты
mysqldump -u root -p --all-databases --single-transaction | gzip > /backups/full_backup_$(date +%Y%m%d).sql.gz
```

---

### 1.2. Необходимо восстанавливать данные за час до предполагаемой поломки

**Решение:** Комбинированная стратегия — полное резервное копирование + инкрементное резервное копирование + PITR (Point-in-Time Recovery)

**Описание стратегии:**

**Вариант 1: Полное + Инкрементное резервное копирование**
- Выполнять полное резервное копирование раз в сутки (например, в 00:00)
- Выполнять инкрементное резервное копирование каждый час
- Инкрементная копия сохраняет только изменения с момента последнего резервного копирования

**Вариант 2: PITR (Point-in-Time Recovery)**
- Создавать базовую полную копию (base backup)
- Архивировать журналы транзакций (WAL для PostgreSQL, Binary Log для MySQL)
- Возможность восстановления на любой момент времени с точностью до секунды

**Преимущества:**
- Минимальные потери данных (до 1 часа или меньше)
- Точное восстановление на конкретный момент времени
- Экономия дискового пространства по сравнению с почасовым полным копированием

**Недостатки:**
- Более сложный процесс восстановления
- Требуется последовательное применение всех инкрементных копий
- Зависимость от целостности цепочки резервных копий

**Пример для PostgreSQL с WAL:**
```bash
# Включение архивирования WAL в postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'

# Создание базовой резервной копии
pg_basebackup -U postgres -D /backups/base_backup -Fp -Xs -P

# Восстановление на конкретный момент времени
# В recovery.conf указать:
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2025-10-20 14:00:00'
```

**Пример для MySQL с Binary Log:**
```bash
# Включение binary log в my.cnf
[mysqld]
log_bin = /var/log/mysql/mysql-bin
expire_logs_days = 10
binlog_format = ROW

# Полное резервное копирование с flush logs
mysqldump -u root -p --all-databases --single-transaction --flush-logs --master-data=2 > /backups/full_backup.sql

# Инкрементное резервное копирование (сохранение binary logs)
mysqlbinlog --start-datetime="2025-10-20 13:00:00" --stop-datetime="2025-10-20 14:00:00" /var/log/mysql/mysql-bin.000001 > /backups/incremental.sql
```

---

### 1.3.* Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных

**Ответ:** Да, такой кейс возможен и реализуется через механизм **репликации с автоматическим переключением (Automatic Failover)** и **высокую доступность (High Availability)**.

**Основные решения:**

#### **1. Синхронная репликация с автоматическим failover**

**Компоненты:**
- **Primary (Master)** — основной сервер базы данных, принимающий операции записи
- **Standby (Replica/Slave)** — резервный сервер, синхронизированный с основным
- **Witness/Monitor** — наблюдатель, контролирующий состояние серверов и инициирующий переключение

**Типы Standby:**
- **Hot Standby** — резервный сервер работает и может обслуживать операции чтения
- **Warm Standby** — резервный сервер находится в режиме восстановления, готов к быстрому переключению
- **Cold Standby** — резервный сервер выключен, требует времени на запуск

**Процесс автоматического failover:**
1. Система мониторинга обнаруживает отказ основного сервера
2. Автоматически промотирует standby-сервер в роль primary
3. Перенаправляет клиентские подключения на новый primary через виртуальный IP или DNS
4. Время переключения: 30 секунд - 2 минуты

**Преимущества:**
- **Нулевая потеря данных** при синхронной репликации (RPO = 0)
- Минимальное время простоя (RTO = 30-120 секунд)
- Автоматическое восстановление без вмешательства администратора
- Непрерывная доступность для критичных приложений

**Недостатки:**
- Высокая стоимость инфраструктуры (минимум 2 сервера)
- Сложность настройки и поддержки
- Задержки записи при синхронной репликации из-за сетевой латентности
- Ограничение расстояния между серверами (до 300 км для синхронной репликации)

#### **2. Решения для PostgreSQL**

**pg_auto_failover:**
```bash
# Установка и настройка автоматического failover
pg_autoctl create monitor --hostname monitor.example.com
pg_autoctl create postgres --hostname primary.example.com --monitor postgresql://monitor.example.com
pg_autoctl create postgres --hostname standby.example.com --monitor postgresql://monitor.example.com
```

**repmgr:**
- Управление репликацией и автоматическое переключение
- Мониторинг состояния кластера
- Возможность ручного и автоматического failover

**Patroni:**
- Управление HA-кластером PostgreSQL
- Интеграция с etcd, Consul или ZooKeeper
- Автоматический failover и switchback

#### **3. Решения для MySQL**

**MySQL Group Replication:**
- Встроенный механизм автоматического failover
- Поддержка нескольких мастеров (multi-master)
- Автоматическое обнаружение и исключение неработающих узлов

**MySQL InnoDB Cluster:**
```bash
# Создание кластера
mysqlsh
dba.createCluster('myCluster')
cluster = dba.getCluster()
cluster.addInstance('user@host2:3306')
cluster.addInstance('user@host3:3306')
```

**MHA (Master High Availability):**
- Мониторинг master-сервера
- Автоматическое переключение на replica при сбое
- Минимальное время простоя (10-30 секунд)

#### **4. Сравнение с резервным копированием**

| Характеристика | Резервное копирование | Репликация с Failover |
|---|---|---|
| **RTO (Recovery Time Objective)** | Часы - дни | Секунды - минуты |
| **RPO (Recovery Point Objective)** | До последней копии | Ноль (при синхронной) |
| **Автоматизация** | Ручное восстановление | Автоматическое переключение |
| **Стоимость** | Низкая | Высокая |
| **Сложность** | Низкая | Высокая |
| **Назначение** | Защита от потери данных | Непрерывная доступность |

**Вывод:** Для критичных финансовых систем рекомендуется **комбинированный подход**: репликация с автоматическим failover для обеспечения непрерывной работы + регулярное резервное копирование для защиты от логических ошибок и долгосрочного хранения данных.

---

## Задание 2. PostgreSQL

### 2.1. Примеры команд резервирования данных и восстановления БД (pg_dump/pg_restore)

#### **Резервное копирование с помощью pg_dump**

**1. Резервное копирование одной базы данных в формате SQL:**
```bash
pg_dump -U postgres -h localhost -d mydatabase > backup.sql
```

**2. Резервное копирование в пользовательском формате (custom format):**
```bash
pg_dump -U postgres -h localhost -d mydatabase -Fc > backup.dump
```
- Флаг `-Fc` создаёт сжатую резервную копию
- Этот формат наиболее гибкий для использования с pg_restore

**3. Резервное копирование в формате directory:**
```bash
pg_dump -U postgres -h localhost -d mydatabase -Fd -f backup_dir
```
- Поддерживает параллельное резервное копирование
- Удобен для больших баз данных

**4. Резервное копирование в формате tar:**
```bash
pg_dump -U postgres -h localhost -d mydatabase -Ft > backup.tar
```

**5. Резервное копирование только схемы (без данных):**
```bash
pg_dump -U postgres -h localhost -d mydatabase --schema-only > schema.sql
```

**6. Резервное копирование только данных (без схемы):**
```bash
pg_dump -U postgres -h localhost -d mydatabase --data-only > data.sql
```

**7. Резервное копирование всех баз данных кластера:**
```bash
pg_dumpall -U postgres -h localhost > cluster_backup.sql
```

**8. Резервное копирование с параллельной обработкой:**
```bash
pg_dump -U postgres -h localhost -d mydatabase -Fd -j 4 -f backup_dir
```
- Флаг `-j 4` использует 4 параллельных процесса для ускорения

#### **Восстановление с помощью pg_restore**

**1. Восстановление из custom format:**
```bash
pg_restore -U postgres -h localhost -d mydatabase -v backup.dump
```
- Флаг `-v` включает подробный вывод

**2. Восстановление с предварительной очисткой:**
```bash
pg_restore -U postgres -h localhost -d mydatabase -c backup.dump
```
- Флаг `-c` удаляет объекты перед восстановлением

**3. Создание базы данных и восстановление:**
```bash
pg_restore -U postgres -h localhost -C -d postgres backup.dump
```
- Флаг `-C` создаёт базу данных перед восстановлением

**4. Восстановление только определённой таблицы:**
```bash
pg_restore -U postgres -h localhost -d mydatabase -t users backup.dump
```

**5. Восстановление только схемы (без данных):**
```bash
pg_restore -U postgres -h localhost -d mydatabase --schema-only backup.dump
```

**6. Восстановление только данных (без схемы):**
```bash
pg_restore -U postgres -h localhost -d mydatabase --data-only backup.dump
```

**7. Параллельное восстановление:**
```bash
pg_restore -U postgres -h localhost -d mydatabase -j 4 backup_dir
```

**8. Восстановление из tar формата:**
```bash
pg_restore -U postgres -h localhost -d mydatabase -Ft backup.tar
```

**9. Восстановление из SQL-дампа (plain format):**
```bash
psql -U postgres -h localhost -d mydatabase < backup.sql
```

**10. Восстановление всех баз данных:**
```bash
psql -U postgres -h localhost < cluster_backup.sql
```

---

### 2.1.* Возможно ли автоматизировать этот процесс? Если да, то как?

**Ответ:** Да, процесс резервного копирования PostgreSQL можно и нужно автоматизировать.

#### **Способ 1: Использование cron (Linux/Unix)**

**Шаг 1. Создание скрипта резервного копирования**

Создайте файл `/usr/local/bin/postgres_backup.sh`:
```bash
#!/bin/bash

# Конфигурация
BACKUP_DIR="/var/backups/postgresql"
DB_USER="postgres"
DB_NAME="mydatabase"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# Создание директории для резервных копий
mkdir -p "$BACKUP_DIR"

# Резервное копирование
pg_dump -U "$DB_USER" -Fc "$DB_NAME" > "$BACKUP_DIR/backup_${DB_NAME}_${DATE}.dump"

# Проверка успешности
if [ $? -eq 0 ]; then
    echo "Backup completed successfully: backup_${DB_NAME}_${DATE}.dump"
    
    # Удаление старых резервных копий (старше 7 дней)
    find "$BACKUP_DIR" -name "backup_*.dump" -type f -mtime +$RETENTION_DAYS -delete
    
    # Опционально: отправка в облачное хранилище (AWS S3)
    # aws s3 cp "$BACKUP_DIR/backup_${DB_NAME}_${DATE}.dump" s3://my-bucket/postgresql-backups/
else
    echo "Backup failed!" >&2
    exit 1
fi
```

**Шаг 2. Настройка прав доступа**
```bash
chmod +x /usr/local/bin/postgres_backup.sh
```

**Шаг 3. Настройка аутентификации без пароля**

Создайте файл `~/.pgpass`:
```bash
echo "localhost:5432:mydatabase:postgres:your_password" > ~/.pgpass
chmod 600 ~/.pgpass
```

**Шаг 4. Настройка cron для автоматического выполнения**
```bash
# Открыть crontab для редактирования
crontab -e

# Ежедневное резервное копирование в 2:00 ночи
0 2 * * * /usr/local/bin/postgres_backup.sh >> /var/log/postgres_backup.log 2>&1

# Резервное копирование каждые 6 часов
0 */6 * * * /usr/local/bin/postgres_backup.sh >> /var/log/postgres_backup.log 2>&1

# Еженедельное резервное копирование (каждое воскресенье в 3:00)
0 3 * * 0 /usr/local/bin/postgres_backup.sh >> /var/log/postgres_backup.log 2>&1
```

#### **Способ 2: Расширенный скрипт с уведомлениями**

```bash
#!/bin/bash

# Конфигурация
BACKUP_DIR="/var/backups/postgresql"
DB_USER="postgres"
DB_NAMES=("database1" "database2" "database3")
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30
LOG_FILE="/var/log/postgres_backup.log"
EMAIL="admin@example.com"

# Функция логирования
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Создание директории
mkdir -p "$BACKUP_DIR"

# Резервное копирование каждой базы данных
for DB_NAME in "${DB_NAMES[@]}"; do
    log_message "Starting backup of database: $DB_NAME"
    
    BACKUP_FILE="$BACKUP_DIR/backup_${DB_NAME}_${DATE}.dump"
    
    # Выполнение резервного копирования
    pg_dump -U "$DB_USER" -Fc "$DB_NAME" > "$BACKUP_FILE"
    
    if [ $? -eq 0 ]; then
        # Получение размера файла
        SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
        log_message "Backup successful: $BACKUP_FILE (Size: $SIZE)"
    else
        log_message "ERROR: Backup failed for database: $DB_NAME"
        echo "Backup failed for $DB_NAME on $(hostname)" | mail -s "PostgreSQL Backup Failed" "$EMAIL"
        continue
    fi
done

# Резервное копирование глобальных объектов (ролей, tablespaces)
log_message "Backing up global objects"
pg_dumpall -U "$DB_USER" --globals-only > "$BACKUP_DIR/globals_${DATE}.sql"

# Удаление старых резервных копий
log_message "Cleaning up old backups (older than $RETENTION_DAYS days)"
find "$BACKUP_DIR" -name "backup_*.dump" -type f -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "globals_*.sql" -type f -mtime +$RETENTION_DAYS -delete

# Опционально: загрузка в S3
# aws s3 sync "$BACKUP_DIR" s3://my-bucket/postgresql-backups/ --exclude "*" --include "backup_*${DATE}*"

log_message "Backup process completed"
```

#### **Способ 3: Использование pgBackRest (профессиональный инструмент)**

**Установка:**
```bash
sudo apt-get install pgbackrest  # Debian/Ubuntu
sudo yum install pgbackrest       # RHEL/CentOS
```

**Конфигурация `/etc/pgbackrest.conf`:**
```ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
process-max=4

[mydatabase]
pg1-path=/var/lib/postgresql/14/main
```

**Создание резервной копии:**
```bash
# Полное резервное копирование
pgbackrest --stanza=mydatabase --type=full backup

# Инкрементное резервное копирование
pgbackrest --stanza=mydatabase --type=incr backup

# Дифференциальное резервное копирование
pgbackrest --stanza=mydatabase --type=diff backup
```

**Автоматизация через cron:**
```bash
# Полное резервное копирование раз в неделю
0 3 * * 0 pgbackrest --stanza=mydatabase --type=full backup

# Дифференциальное резервное копирование ежедневно
0 3 * * 1-6 pgbackrest --stanza=mydatabase --type=diff backup
```

#### **Способ 4: Использование Barman (Backup and Recovery Manager)**

**Установка:**
```bash
sudo apt-get install barman barman-cli
```

**Настройка в `/etc/barman.conf`:**
```ini
[mydatabase]
description = "Production Database"
ssh_command = ssh postgres@db-server
conninfo = host=db-server user=postgres dbname=mydatabase
backup_method = postgres
backup_options = concurrent_backup
retention_policy = RECOVERY WINDOW OF 7 DAYS
```

**Создание резервной копии:**
```bash
barman backup mydatabase
```

**Автоматизация через cron:**
```bash
# Ежедневное резервное копирование в 1:00
0 1 * * * barman backup all
```

#### **Способ 5: Continuous Archiving (WAL Archiving)**

**Настройка в `postgresql.conf`:**
```ini
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/lib/postgresql/wal_archive/%f && cp %p /var/lib/postgresql/wal_archive/%f'
```

**Скрипт базового резервного копирования:**
```bash
#!/bin/bash
pg_basebackup -U postgres -D /var/backups/postgresql/base_backup_$(date +%Y%m%d) -Fp -Xs -P
```

**Преимущества автоматизации:**
- Регулярные резервные копии без участия человека
- Снижение риска человеческой ошибки
- Возможность восстановления на различные моменты времени
- Централизованное управление и мониторинг
- Интеграция с системами оповещения при сбоях

---

## Задание 3. MySQL

### 3.1. Пример команды инкрементного резервного копирования базы данных MySQL

Инкрементное резервное копирование в MySQL реализуется через **Binary Log (Binlog)**. MySQL не имеет встроенной команды для инкрементного резервного копирования, но его можно реализовать через бинарные журналы.

#### **Подготовка: Включение Binary Log**

**Шаг 1. Настройка `my.cnf` или `my.ini`:**
```ini
[mysqld]
# Включение binary log
log_bin = /var/log/mysql/mysql-bin
server-id = 1
binlog_format = ROW
expire_logs_days = 10
max_binlog_size = 100M
```

**Шаг 2. Перезапуск MySQL:**
```bash
sudo systemctl restart mysql
```

**Шаг 3. Проверка статуса Binary Log:**
```sql
SHOW VARIABLES LIKE 'log_bin';
SHOW MASTER STATUS;
SHOW BINARY LOGS;
```

#### **Процесс инкрементного резервного копирования**

**Шаг 1. Создание полной резервной копии (базовая копия):**
```bash
# Полное резервное копирование с flush и записью позиции
mysqldump -u root -p \
  --all-databases \
  --single-transaction \
  --flush-logs \
  --master-data=2 \
  --delete-master-logs > /backups/full_backup_$(date +%Y%m%d).sql
```

**Опции:**
- `--single-transaction` — создаёт согласованную резервную копию без блокировки таблиц (для InnoDB)
- `--flush-logs` — создаёт новый binary log файл после резервного копирования
- `--master-data=2` — записывает позицию binary log в комментарий в дампе
- `--delete-master-logs` — удаляет старые binary log файлы после создания полной копии

**Шаг 2. Инкрементное резервное копирование (сохранение binary logs):**

**Вариант A. Копирование binary log файлов:**
```bash
#!/bin/bash

BACKUP_DIR="/backups/mysql/incremental"
DATE=$(date +%Y%m%d_%H%M%S)

# Создание директории
mkdir -p "$BACKUP_DIR"

# Сброс текущего binary log (создание нового файла)
mysql -u root -p -e "FLUSH BINARY LOGS;"

# Копирование всех binary log файлов, кроме текущего
for binlog in $(mysql -u root -p -e "SHOW BINARY LOGS;" | awk 'NR>1 {print $1}' | head -n -1); do
    cp /var/log/mysql/$binlog "$BACKUP_DIR/${binlog}_${DATE}"
done

# Удаление старых binary log файлов (опционально)
# mysql -u root -p -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);"
```

**Вариант B. Экспорт binary logs с помощью mysqlbinlog:**
```bash
# Инкрементная копия за определённый период времени
mysqlbinlog \
  --start-datetime="2025-10-20 00:00:00" \
  --stop-datetime="2025-10-20 23:59:59" \
  /var/log/mysql/mysql-bin.000001 > /backups/incremental_$(date +%Y%m%d).sql
```

**Вариант C. Инкрементная копия по позиции:**
```bash
# Получение текущей позиции
mysql -u root -p -e "SHOW MASTER STATUS;"

# Копирование изменений с определённой позиции
mysqlbinlog \
  --start-position=154 \
  --stop-position=4567 \
  /var/log/mysql/mysql-bin.000001 > /backups/incremental_pos.sql
```

#### **Полный скрипт автоматического инкрементного резервного копирования**

```bash
#!/bin/bash

# Конфигурация
MYSQL_USER="root"
MYSQL_PASS="your_password"
FULL_BACKUP_DIR="/backups/mysql/full"
INCR_BACKUP_DIR="/backups/mysql/incremental"
BINLOG_DIR="/var/log/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
DAY_OF_WEEK=$(date +%u)  # 1=Понедельник, 7=Воскресенье

# Создание директорий
mkdir -p "$FULL_BACKUP_DIR"
mkdir -p "$INCR_BACKUP_DIR"

# Если воскресенье — полное резервное копирование
if [ "$DAY_OF_WEEK" -eq 7 ]; then
    echo "Performing FULL backup..."
    
    mysqldump -u "$MYSQL_USER" -p"$MYSQL_PASS" \
        --all-databases \
        --single-transaction \
        --flush-logs \
        --master-data=2 \
        --delete-master-logs | gzip > "$FULL_BACKUP_DIR/full_backup_${DATE}.sql.gz"
    
    echo "Full backup completed: full_backup_${DATE}.sql.gz"
else
    # В остальные дни — инкрементное резервное копирование
    echo "Performing INCREMENTAL backup..."
    
    # Создание нового binary log файла
    mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" -e "FLUSH BINARY LOGS;"
    
    # Получение списка всех binary log файлов кроме текущего
    BINLOGS=$(mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" -Nse "SHOW BINARY LOGS;" | awk '{print $1}' | head -n -1)
    
    # Копирование binary log файлов в архив
    for binlog in $BINLOGS; do
        if [ -f "$BINLOG_DIR/$binlog" ]; then
            cp "$BINLOG_DIR/$binlog" "$INCR_BACKUP_DIR/${binlog}_${DATE}"
            echo "Copied: $binlog"
        fi
    done
    
    # Удаление binary log файлов, которые уже скопированы (старше 3 дней)
    mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" -e "PURGE BINARY LOGS BEFORE DATE_SUB(NOW(), INTERVAL 3 DAY);"
    
    echo "Incremental backup completed"
fi

# Удаление старых резервных копий (старше 30 дней)
find "$FULL_BACKUP_DIR" -name "full_backup_*.sql.gz" -type f -mtime +30 -delete
find "$INCR_BACKUP_DIR" -name "mysql-bin.*" -type f -mtime +30 -delete
```

#### **Восстановление из полной и инкрементных копий**

**Шаг 1. Восстановление полной резервной копии:**
```bash
# Распаковка и восстановление
gunzip < /backups/mysql/full/full_backup_20251020.sql.gz | mysql -u root -p
```

**Шаг 2. Применение инкрементных изменений (binary logs):**
```bash
# Восстановление всех binary log файлов по порядку
mysqlbinlog /backups/mysql/incremental/mysql-bin.000001 \
            /backups/mysql/incremental/mysql-bin.000002 \
            /backups/mysql/incremental/mysql-bin.000003 | mysql -u root -p
```

**Шаг 3. Восстановление до определённого момента времени (Point-in-Time Recovery):**
```bash
# Восстановление до конкретной даты/времени
mysqlbinlog \
  --stop-datetime="2025-10-20 14:00:00" \
  /backups/mysql/incremental/mysql-bin.* | mysql -u root -p
```

#### **Автоматизация через cron**

```bash
# Редактирование crontab
crontab -e

# Полное резервное копирование каждое воскресенье в 2:00
0 2 * * 0 /usr/local/bin/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1

# Инкрементное резервное копирование ежедневно в 2:00 (кроме воскресенья)
0 2 * * 1-6 /usr/local/bin/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1
```

---

### 3.1.* В каких случаях использование реплики будет давать преимущество по сравнению с обычным резервным копированием?

**Ответ:** Репликация даёт значительные преимущества в следующих сценариях:

#### **1. Высокая доступность (High Availability)**

**Преимущества репликации:**
- **Моментальное переключение** при отказе основного сервера (failover время: 30-120 секунд)
- **Нулевое или минимальное время простоя** (RTO ≈ 0-2 минуты)
- **Непрерывная работа приложений** без остановки сервиса

**Резервное копирование:**
- Время восстановления: часы или дни
- Требуется остановка приложений на время восстановления
- Возможная потеря данных с момента последней копии

**Когда использовать репликацию:**
- Критичные приложения с требованием 99.9% uptime
- Финансовые системы, e-commerce платформы
- SLA требует минимальное время простоя

---

#### **2. Распределение нагрузки чтения (Read Scaling)**

**Преимущества репликации:**
- **Read-only реплики** обрабатывают SELECT-запросы
- **Разгрузка мастера** — все записи на master, чтение на replicas
- **Горизонтальное масштабирование** — добавление новых реплик для увеличения пропускной способности

**Пример:**
```
Master (Write): 100% INSERT/UPDATE/DELETE
Replica 1 (Read): 33% SELECT
Replica 2 (Read): 33% SELECT
Replica 3 (Read): 33% SELECT
```

**Резервное копирование:**
- Не помогает в распределении нагрузки
- Копии находятся в "холодном" состоянии

**Когда использовать репликацию:**
- Приложения с высокой нагрузкой чтения (read-heavy workloads)
- Аналитические запросы и отчёты
- Разделение OLTP и OLAP нагрузки

---

#### **3. Снижение нагрузки при создании резервных копий**

**Преимущества репликации:**
- Резервное копирование выполняется **на реплике**, а не на продакшн-сервере
- **Нулевое влияние** на производительность основного сервера
- Возможность остановки реплики для создания консистентной копии

**Пример:**
```bash
# Резервное копирование на реплике (не влияет на master)
mysqldump -h replica-server -u backup_user -p --all-databases > backup.sql
```

**Резервное копирование на master:**
- Создаёт дополнительную нагрузку на I/O
- Может замедлить работу приложений
- Блокировки при резервном копировании MyISAM таблиц

**Когда использовать репликацию:**
- Большие базы данных (сотни GB - TB)
- Высоконагруженные системы 24/7
- Необходимость частых резервных копий без влияния на production

---

#### **4. Географическое распределение (Geo-Distribution)**

**Преимущества репликации:**
- **Реплики в разных регионах** для снижения latency
- Пользователи получают данные с ближайшего сервера
- Защита от региональных катастроф (災害対策)

**Пример:**
```
Master (USA): Записи
Replica (Europe): Чтение для европейских пользователей
Replica (Asia): Чтение для азиатских пользователей
```

**Резервное копирование:**
- Требует время на передачу копий между регионами
- Не обеспечивает быстрый доступ к данным

**Когда использовать репликацию:**
- Глобальные приложения с пользователями по всему миру
- Требования к низкой задержке для международных клиентов

---

#### **5. Тестирование и разработка**

**Преимущества репликации:**
- **Актуальная копия данных** для тестирования
- Возможность тестирования обновлений без влияния на production
- Dev/QA окружения с реальными данными

**Резервное копирование:**
- Данные устаревают между копиями
- Требуется время на восстановление для тестирования

**Когда использовать репликацию:**
- Continuous Integration/Continuous Deployment (CI/CD)
- Необходимость тестирования на актуальных данных
- Debugging production issues

---

#### **6. Защита от аппаратных сбоев**

**Преимущества репликации:**
- **Немедленное переключение** на реплику при сбое оборудования
- **Минимальная потеря данных** (RPO ≈ 0 при синхронной репликации)
- Автоматическое восстановление через failover

**Резервное копирование:**
- Требуется ручное восстановление
- Потеря данных с момента последней копии
- Длительное время восстановления

**Когда использовать репликацию:**
- Критичные системы с требованием RPO < 1 минута
- Финансовые транзакции, биржевые системы
- Медицинские системы с критичными данными

---

#### **7. Аналитика и отчётность**

**Преимущества репликации:**
- **Dedicated replica** для тяжёлых аналитических запросов
- Не влияет на производительность transactional workload
- Возможность создания специализированных индексов на реплике

**Пример:**
```
Master: OLTP (Online Transaction Processing)
Replica: OLAP (Online Analytical Processing) - сложные агрегации, отчёты
```

**Резервное копирование:**
- Восстановление копии требует времени и ресурсов
- Данные быстро устаревают

**Когда использовать репликацию:**
- Business Intelligence (BI) системы
- Data Warehousing
- Сложные отчёты и аналитика в реальном времени

---

#### **8. Защита от логических ошибок**

**Важное замечание:** В этом сценарии резервное копирование **лучше** репликации!

**Проблема с репликацией:**
- Ошибочные операции (DROP TABLE, DELETE без WHERE) **реплицируются** на все серверы
- Невозможно восстановить данные только из реплики
- Задержка репликации (replication lag) может дать небольшое окно для остановки

**Преимущества резервного копирования:**
- Возможность восстановления данных до момента ошибки
- Point-in-Time Recovery (PITR) для точного восстановления
- Защита от ransomware и вредоносного ПО

**Решение:** **Комбинированный подход**
- Репликация для высокой доступности
- Регулярное резервное копирование для защиты от логических ошибок
- PITR (Binary Log) для восстановления на точный момент времени

---

#### **Сравнительная таблица: Репликация vs Резервное копирование**

| Критерий | Репликация | Резервное копирование |
|---|---|---|
| **RTO (Recovery Time)** | Секунды - минуты | Часы - дни |
| **RPO (Recovery Point)** | Секунды (синхронная) | До последней копии |
| **Высокая доступность** | ✅ Да (автоматический failover) | ❌ Нет |
| **Распределение нагрузки** | ✅ Да (read replicas) | ❌ Нет |
| **Защита от логических ошибок** | ❌ Нет (ошибки реплицируются) | ✅ Да |
| **Защита от аппаратных сбоев** | ✅ Да (моментальное переключение) | ⚠️ Медленное восстановление |
| **Стоимость** | Высокая (дополнительное оборудование) | Низкая (только storage) |
| **Сложность** | Высокая | Низкая - средняя |
| **Влияние на production** | Минимальное | Может быть значительным |
| **Долгосрочное хранение** | ❌ Нет | ✅ Да (архивирование) |

---

#### **Рекомендации по комбинированному подходу**

**Оптимальная стратегия для критичных систем:**

1. **Репликация (Master-Slave или Master-Master):**
   - Для высокой доступности и автоматического failover
   - Для распределения нагрузки чтения
   - Для снижения latency в разных географических регионах

2. **Регулярное резервное копирование:**
   - Ежедневные полные копии
   - Почасовые инкрементные копии (Binary Log)
   - Долгосрочное хранение (недели/месяцы/годы)

3. **Point-in-Time Recovery (PITR):**
   - Архивирование Binary Logs
   - Возможность восстановления на любой момент времени
   - Защита от логических ошибок и ransomware

**Пример архитектуры:**
```
Production:
  ├─ Master (Primary)
  ├─ Replica 1 (Hot Standby для failover)
  ├─ Replica 2 (Read-only для аналитики)
  └─ Replica 3 (Backup Server)

Backup Strategy:
  ├─ Полное резервное копирование: Воскресенье 02:00 (на Replica 3)
  ├─ Инкрементное резервное копирование: Ежедневно 02:00
  ├─ Binary Log архивирование: Непрерывно
  └─ Offsite копии: В облачное хранилище (S3, GCS)
```

**Заключение:** Репликация и резервное копирование решают разные задачи и дополняют друг друга. Для критичных систем необходимо использовать оба метода совместно.

---

## Заключение

Резервное копирование баз данных — критически важный процесс для обеспечения надёжности и непрерывности работы информационных систем. Для финансовых компаний рекомендуется комбинированный подход:

1. **Ежедневные полные резервные копии** для восстановления за предыдущий день
2. **Инкрементное резервное копирование + PITR** для восстановления на любой момент времени
3. **Репликация с автоматическим failover** для обеспечения высокой доступности и моментального переключения
4. **Автоматизация** всех процессов резервного копирования через cron или специализированные инструменты

Правильная стратегия резервного копирования обеспечивает защиту данных, минимизирует время простоя и гарантирует возможность восстановления системы в случае любых инцидентов.
