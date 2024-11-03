# Руководство по установке и настройке Hadoop и Hive с использованием PostgreSQL

## Предварительные требования
- Установленный и настроенный Hadoop (HDFS и YARN)
- Java (OpenJDK 11 или новее)
- SSH доступ ко всем узлам
- Привилегии sudo

## Шаг 1: Запуск сервисов Hadoop

Запустите HDFS и YARN:

```bash
start-dfs.sh
start-yarn.sh
```

## Шаг 2: Установка Hive

1. Скачайте и разархивируйте Hive:

```bash
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
tar -zxvf apache-hive-4.0.1-bin.tar.gz
```

2. Настройте переменные окружения. Добавьте следующие строки в `~/.bashrc`:

```bash
export HIVE_HOME=/home/hadoop/apache-hive-4.0.1-bin
export PATH=$PATH:$HIVE_HOME/bin
```

Примените изменения:

```bash
source ~/.bashrc
```

3. Создайте необходимые директории в HDFS:

```bash
hdfs dfs -mkdir /tmp
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
```

## Шаг 3: Установка и настройка PostgreSQL

1. Установите PostgreSQL:

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
```

2. Проверьте статус PostgreSQL:

```bash
sudo systemctl status postgresql
```

3. Настройте `pg_hba.conf`:

```
host    all             hive             192.168.1.23/32         md5
host    all             hive             192.168.1.24/32         md5
host    all             hive             192.168.1.25/32         md5
```

4. Настройте `postgresql.conf`:

```
listen_addresses = '*'
```

5. Установите клиент PostgreSQL на всех узлах:

```bash
sudo apt-get install postgresql-client
```

6. Создайте базу данных и пользователя для Hive:

```sql
sudo -u postgres psql
CREATE DATABASE metastore;
CREATE USER hive WITH PASSWORD 'hive';
GRANT ALL PRIVILEGES ON DATABASE metastore TO hive;
GRANT USAGE ON SCHEMA public TO hive;
GRANT CREATE ON SCHEMA public TO hive;
\q
```

## Шаг 4: Настройка Hive

1. Скачайте JDBC-драйвер для PostgreSQL:

```bash
wget https://jdbc.postgresql.org/download/postgresql-42.2.20.jar
mv postgresql-42.2.20.jar $HIVE_HOME/lib/
```

2. Настройте `hive-site.xml`:

```bash
touch hive-site.xml
# Скопируйте настройки из репозитория в этот файл

sudo mkdir /var/log/hive/operation_logs
# также нужно создать папку для логов
```

3. Настройте `hive-env.sh`:

```bash
cp hive-env.sh.template hive-env.sh
echo "HADOOP_HOME=/home/hadoop/hadoop-3.4.0" >> hive-env.sh
echo "export HIVE_HOME=/home/hadoop/apache-hive-4.0.1-bin" >> hive-env.sh
```

4. Инициализируйте схему метастора Hive:

```bash
schematool -initSchema -dbType postgres
```

## Шаг 5: Запуск сервисов Hive

1. Запустите Hive Metastore:

```bash
hive --service metastore &
```

2. Запустите Hive Server2:

```bash
hive --hiveconf hive.server2.enable.doAs=false \
--hiveconf hive.security.authorization.enabled=false \
--service hiveserver2 1>> /tmp/hs2.log 2>> /tmp/hs2.log &
```

## Шаг 6: Использование Hive

1. Запустите Beeline:

```bash
beeline -u jdbc:hive2://
```

2. Создайте тестовую таблицу:

```sql
CREATE TABLE IF NOT EXISTS customers (
    customer_id INT,
    name STRING,
    email STRING,
    registration_date DATE
)
COMMENT 'Table to store customer information'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;
```

3. Загрузите данные из CSV:

```bash
# Создайте файл с тестовыми данными
cat << EOF > user_data.csv
id,name,registration_date
1,John Doe,2024-10-01
2,Jane Smith,2024-10-02
3,Bob Johnson,2024-10-01
4,Alice Williams,2024-10-03
EOF

# Копируйте файл в HDFS
hdfs dfs -put user_data.csv /tmp
```

4. Создайте таблицу и загрузите данные:

```sql
CREATE TABLE IF NOT EXISTS users (
    id INT,
    name STRING,
    registration_date date
)
COMMENT 'Table to store user information'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

LOAD DATA INPATH '/tmp/user_data.csv' INTO TABLE users;

ANALYZE TABLE users COMPUTE STATISTICS;
```

5. Создайте партиционированную таблицу:

```sql
CREATE TABLE IF NOT EXISTS users_partitioned (
    id INT,
    name STRING
)
COMMENT 'Table to store user information, partitioned by registration date'
PARTITIONED BY (registration_date date)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

INSERT INTO TABLE users_partitioned PARTITION (registration_date)
SELECT id, name, registration_date FROM users;
```

# Настройка аутентификации и прокси для Hadoop UI

## Установка htpasswd (все действия воспроизводятся на jump-node)

```bash
sudo apt update
sudo apt install apache2-utils
```

## Создание файла паролей

```bash
# Создаем файл паролей и добавляем пользователя
sudo htpasswd -c /etc/nginx/.htpasswd hadoop
# Вам будет предложено ввести пароль для hadoop
# пароль: h@DooP$

# Выдаем нужные права
sudo chmod 644 /etc/nginx/.htpasswd
```

Примечание: Флаг `-c` создаёт новый файл. Если файл уже существует и вам нужно добавить ещё одного пользователя, используйте команду без `-c`:

```bash
sudo htpasswd /etc/nginx/.htpasswd new_user
```

## Настройка Nginx для NameNode UI

```bash
# Создаем файл для namenode
touch /etc/nginx/sites-available/nn

# Заполняем файл и прописываем аутентификацию по паролю
sudo vim /etc/nginx/sites-available/nn
```

Содержимое файла:

```nginx
server {
    listen 9870 default_server;
    root /var/www/html;
    index index.html index.htm index.nginx-debian.html;
    server_name _;

    location / {
        proxy_pass http://team-5-nn:9870;
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

```bash
# Добавляем UI namenode к доступным сайтам
sudo ln -s /etc/nginx/sites-available/nn /etc/nginx/sites-enabled/nn
```

## Настройка для других компонентов

Повторите аналогичные шаги для (в каждом файле в двух местах меняем порты):
- Secondary NameNode (порт 9868)
- YARN Resource Manager (порт 8088)
- History Server (порт 19888)

### Secondary NameNode

```bash
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/sn
sudo vim /etc/nginx/sites-available/sn
sudo ln -s /etc/nginx/sites-available/sn /etc/nginx/sites-enabled/sn
```

### YARN Resource Manager

```bash
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/ya
sudo vim /etc/nginx/sites-available/ya
sudo ln -s /etc/nginx/sites-available/ya /etc/nginx/sites-enabled/ya
```

### History Server

```bash
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/dh
sudo vim /etc/nginx/sites-available/dh
sudo ln -s /etc/nginx/sites-available/dh /etc/nginx/sites-enabled/dh
```

## Доступ к веб-интерфейсам

После настройки вы можете получить доступ к следующим веб-интерфейсам:

- NameNode UI: http://176.109.91.7:9870
- Secondary NameNode UI: http://176.109.91.7:9868
- YARN Resource Manager: http://176.109.91.7:8088
- History Server: http://176.109.91.7:19888
```
