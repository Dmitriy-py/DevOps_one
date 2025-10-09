# Создать несколько контейнеров (например, для веб-сервера и базы данных), разместить их в сети и запустить бесплатное обновление через Docker Compose.

1. PostgreSQL (база данных)
2. Приложение Flask (простое Python-приложение, взаимодействующее с БД)
3. Nginx (веб-сервер, выступающий в качестве обратного прокси для Flask-приложений)

Объединим их в сети и настроим бесплатное обновление через Docker Compose.

```yaml
.
├── docker-compose.yml
├── .env
├── nginx.conf
└── app/
    ├── Dockerfile
    ├── app.py
    └── requirements.txt
```

### Шаг 1: Конфигурация файлов

### `.env `

Этот сложный файл будет сохранять переменные окружения для наших сервисов, что позволит легко изменять их без редактирования docker-compose.ymlи жесткого кодирования чувствительной информации.

```yaml
env

# Database variables
POSTGRES_DB=mydatabase
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword

# Application database connection string
# Note: 'db' is the service name for PostgreSQL in docker-compose.yml
DATABASE_URL=postgresql://myuser:mypassword@db:5432/mydatabase
```

### ` nginx.conf `

Конфигурация Nginx для проксирования запросов к каждому Flask-приложению.

```yaml
nginx

worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen 80;
        server_name localhost;

        location / {
            # 'app' is the service name for our Flask application in docker-compose.yml
            # '8000' is the port our Flask app listens on
            proxy_pass http://app:8000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### ` app/requirements.txt `

Зависимости для нашего Flask-приложения.

```yaml
Flask==2.2.2
psycopg2-binary==2.9.5 # Для работы с PostgreSQL
gunicorn==20.1.0 # Production-ready WSGI HTTP Server for Python
```

### ` app/Dockerfile `

Dockerfile для сборки нашего Flask-приложения.
```yaml
dockerfile

# Используем официальный образ Python как базовый
FROM python:3.9-slim-buster

# Устанавливаем рабочую директорию
WORKDIR /app

# Копируем файл зависимостей и устанавливаем их
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Копируем остальной код приложения
COPY . .

# Открываем порт, на котором будет слушать Gunicorn
EXPOSE 8000

# Команда для запуска приложения с Gunicorn
# Приложение будет слушать на 0.0.0.0:8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### ` app/app.py `

Простое приложение Flask, основанное на PostgreSQL, создает таблицу items, добавляет элемент и отображает все элементы.

```python
питон

import os
from flask import Flask, jsonify
import psycopg2
import time

app = Flask(__name__)

# Функция для подключения к базе данных с повторными попытками
def get_db_connection():
    conn = None
    retries = 5
    while retries > 0:
        try:
            conn = psycopg2.connect(os.getenv('DATABASE_URL'))
            print("Successfully connected to the database!")
            return conn
        except psycopg2.OperationalError as e:
            print(f"Database connection failed: {e}. Retrying in 5 seconds...")
            retries -= 1
            time.sleep(5)
    raise Exception("Could not connect to the database after multiple retries.")

# Создание таблицы при первом запуске
def init_db():
    conn = get_db_connection()
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS items (
            id SERIAL PRIMARY KEY,
            name VARCHAR(100) NOT NULL
        );
    """)
    conn.commit()
    cur.close()
    conn.close()

# Инициализируем базу данных при запуске приложения
with app.app_context():
    init_db()

@app.route('/')
def hello():
    return jsonify({"message": "Hello from Flask! Connected to PostgreSQL."})

@app.route('/add_item/<name>')
def add_item(name):
    conn = None
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("INSERT INTO items (name) VALUES (%s) RETURNING id;", (name,))
        item_id = cur.fetchone()[0]
        conn.commit()
        return jsonify({"message": f"Item '{name}' added with ID: {item_id}"})
    except Exception as e:
        return jsonify({"error": str(e)}), 500
    finally:
        if conn:
            cur.close()
            conn.close()

@app.route('/items')
def get_items():
    conn = None
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT id, name FROM items;")
        items = cur.fetchall()
        return jsonify([{"id": item[0], "name": item[1]} for item in items])
    except Exception as e:
        return jsonify({"error": str(e)}), 500
    finally:
        if conn:
            cur.close()
            conn.close()

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0', port=8000)
```
### Шаг 2:docker-compose.yml

Теперь, когда все вспомогательные файлы готовы, создаем основной docker-compose.yml.
```yaml
yaml

version: '3.8'

services:
  db:
    image: postgres:13-alpine # Используем стабильный образ PostgreSQL 13 (alpine для меньшего размера)
    restart: unless-stopped # Перезапускать контейнер, если он остановился, но не при явной остановке
    environment:
      POSTGRES_DB: ${POSTGRES_DB} # Из .env
      POSTGRES_USER: ${POSTGRES_USER} # Из .env
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD} # Из .env
    volumes:
      - db-data:/var/lib/postgresql/data # Персистентное хранение данных БД
    healthcheck: # Проверка здоровья для PostgreSQL
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: ./app # Собираем образ из Dockerfile в директории ./app
    restart: unless-stopped
    environment:
      DATABASE_URL: ${DATABASE_URL} # Из .env
    depends_on:
      db:
        condition: service_healthy # Зависит от того, что db сервис стал "здоровым"
    #ports:
    #  - "8000:8000" # Обычно Flask не выставляется наружу напрямую, Nginx будет проксировать

  nginx:
    image: nginx:stable-alpine # Стабильный образ Nginx (alpine для меньшего размера)
    restart: unless-stopped
    ports:
      - "80:80" # Выставляем порт 80 хоста на порт 80 контейнера Nginx
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro # Монтируем нашу конфигурацию Nginx
    depends_on:
      - app # Nginx зависит от того, что app сервис запущен

volumes:
  db-data: # Определение named volume для PostgreSQL
```

## Шаг 3: Запуск и проверка

1. Создайте все файлы и убедитесь, что структура проекта соответствует указанной выше.
2. Откройте терминал в корневой директории вашего проекта (где находится docker-compose.yml).
3. Выберите изображения и запустите контейнеры:
```yaml
docker-compose build
docker-compose up -d
```
   * docker-compose buildподобрать образ для сервиса app.
   * docker-compose up -dзапустите все сервисы в фоновом режиме.

4. Проверить статус контейнеров:
```yaml
docker-compose ps
```
Вы должны увидеть, что все контейнеры работают.

5. Проверьте приложение: откройте браузер и проверьте адрес http://localhost. Вы должны увидеть JSON-ответ: {"message":        "Hello from Flask! Connected to PostgreSQL."}.
   * добавить элемент:http://localhost/add_item/MyTestItem
   * Просмотрите элементы:http://localhost/items

### Шаг 4. Автоматическое обновление через Docker Compose

Сам Docker Compose у себя не имеет встроенного механизма для «автоматического» обновления образов в фоновом режиме, как это делает, например, утилита Watchtower. Он предназначен для определения и запуска многоконтейнерных Docker-приложений.

Однако вы можете вручную инициировать обновление или использовать внешние инструменты для автоматизации.

### 1.  Ручное обновление (рекомендуется для контролируемого обновления)

Чтобы обновить ваше приложение до последнего варианта образов, включите его docker-compose.yml(например, postgres:13-alpine, nginx:stable-alpine, или ваш собственный appобраз после изменений в его Dockerfile/коде):

1. Остановите и удалите текущие контейнеры:
```yaml
docker-compose down
```
  * db-dataтом остается нетронутым, так что ваши данные сохраняются.
2. загрузить (вытащить) последнюю версию различных образов:
```yaml
docker-compose pull
```
Это загрузит новые версии postgres:13-alpineи nginx:stable-alpine, если они доступны в Docker Hub.

3. Пересоберите ваш кастомный образ (если код изменился):
```yaml
docker-compose build --no-cache app
```
Используйте --no-cache, чтобы убедиться, что Docker не использует и действительно пересобирает образ с кэши последними изменениями.

4. Запустите все снова:
```yaml
docker-compose up -d
```
Docker Compose увидит новые образы и создаст новые контейнеры на их основе.

Примечание. Если вы обновляете только измененные сервисы без downвсего стека, вы можете использовать:
```yaml
docker-compose pull # Обновить внешние образы
docker-compose build --no-cache app # Пересобрать ваш app образ
docker-compose up -d --no-recreate --force-recreate db app nginx # Пересоздать контейнеры, используя новые образы
```
--force-recreateзаставит Compose пересоздать контейнеры, даже если их облик не изменился, но образы, на которых они основаны, обновились.

### 2. Автоматическое обновление с Watchtower(для полной автоматизации)

Watchtower- это отличный инструмент, который отслеживает работающие контейнеры и обновляет их, когда создаются новые версии их базовых образов. Вы можете добавить Watchtowerкак ещё один сервис в свой docker-compose.yml.

Добавьте Сторожевую башню к себе docker-compose.yml:

```yaml
yaml

version: '3.8'

services:
  # ... (Ваши существующие сервисы: db, app, nginx) ...

  watchtower:
    image: containrrr/watchtower
    restart: always # Watchtower должен работать постоянно
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # Необходимо для Watchtower, чтобы он мог взаимодействовать с Docker Daemon
    environment:
      # Настройки Watchtower (опционально)
      - WATCHTOWER_POLL_INTERVAL=300 # Проверять обновления каждые 300 секунд (5 минут)
      - WATCHTOWER_CLEANUP=true # Удалять старые образы после обновления
      #- WATCHTOWER_INCLUDE_STOPPED=true # Обновлять также и остановленные контейнеры
      #- WATCHTOWER_REVIVE_STOPPED=true # Запускать обновленные остановленные контейнеры

      # Если вы хотите, чтобы Watchtower обновлял только определенные сервисы,
      # укажите их имена (через запятую)
      #- WATCHTOWER_INCLUDE_RESTARTING=db,nginx,app
    command: --interval 300 # Проверять обновления каждые 5 минут (можно использовать WATCHTOWER_POLL_INTERVAL вместо этого)
```
### Обновленные инструкции для запуска Watchtower:

1. Сохраните изменения в docker-compose.yml.
2. Запустите стек:
```yaml
docker-compose up -d
```
Теперь Watchtowerбудет работать в фоновом режиме. Вскоре Docker Hub (и ваш локальный реестр для ваших build:образов) обнаружил наличие новых версий образов, используя ваши контейнеры. Если Watchtower найдет новый образ, он остановит старый контейнер, скачает новый образ, запустит новый контейнер на основе нового образа и, при необходимости, удалит старый образ.

### Важные замечания по Сторожевой башне:

  * Теги Образов: Watchtower лучше всего работает с тегами, которые могут изменяться, например latest, stable, 13-alpine.       Если вы используете очень специфический тег вроде postgres:13.5, Watchtower не найдет нового образа, пока вы не             измените тег docker-compose.ymlи не перезапустите сервис.
  * Собственные Образы (Сборка): Для ваших естественных образов (например, app), Watchtowerбудет искать новый образ в           локальном реестре Docker . Чтобы Сторожевая Башня «увидела» изменения в вашем приложении, вам нужно сначала                 адаптировать docker-compose build app(или просто docker build -t your_image_name:tag ./app) вне Сторожевой Башни, чтобы     обновить локальный образ. После этого Сторожевая Башня сможет помочь и развернуть этот новый локальный образ.
  * Производство: В производственной среде автоматические обновления могут быть опасными. Убедитесь, что у вас есть             стратегия отката, и тщательно тестируйте обновления перед их развертыванием. Рекомендуется использовать более               контролируемые пайплайны CI/CD.
  * Зависимость ( depends_on): Сторожевая башня уважает в зависимости от того. Когда он обновляет один контейнер, он будет      перезапускать зависимые контейнеры соответствующим образом.

### 3. Обновление с Cron Job (простая автоматизация)

Вы можете настроить функцию cronна вашей хостовой системе, которая периодически запускает команды обновлений Docker Compose:
```yaml
# Откройте crontab для редактирования
crontab -e
```
Добавьте этот текст, чтобы запускать обновление каждый день каждые 3 часа ночи (например):
```yaml
0 3 * * * cd /path/to/your/project && docker-compose pull && docker-compose build --no-cache app && docker-compose up -d --no-recreate --force-recreate db app nginx >> /var/log/docker-update.log 2>&1
```
  * Замените /path/to/your/projectфактический путь к вашей директории docker-compose.yml.
  * Это сначала переходит в директорию, затем вытягивает новые образы, пересобирает ваш appобраз команды и, наконец,            перезапускает контейнеры проекта, используя новые образы.
  * Вывод перенаправляется в лог-файл /var/log/docker-update.logдля отладки.

### Заключение

Для большинства случаев разработки и тестирования ручное обновление docker-compose pull && docker-compose up -dдоступно и безопасно. Для более полной автоматизации, особенно для «всегда включенных» систем Watchtower— отличное решение, которое можно легко интегрировать в ваш стек. Для особо важных производственных систем рассмотрите более сложные пайплайны CI/CD, которые включают тестирование перед развертыванием.


# Разработка простого CI/CD Pipeline с использованием Jenkins
## Задача: Настроить Jenkins Pipeline для автоматической сборки и деплоя приложения из GitHub. Необходимо реализовать базовый процесс интеграции и доставки, который включает сборку, тестирование и деплой на тестовый сервер.

### Цель:
Настроить Jenkins Pipeline, который будет:

Отслеживать изменения в GitHub репозитории.
Клонировать репозиторий.
Собирать Java-приложение с помощью Maven.
Запускать юнит-тесты.
Архивировать собранный артефакт (JAR файл).
Деплоить артефакт на удаленный тестовый сервер (например, копировать его по SSH).


### Предварительные требования:
Jenkins Server: Установленный и работающий Jenkins.
(Рекомендуется Ubuntu/Debian)
Java Development Kit (JDK): Установлен на Jenkins сервере.
Maven: Установлен на Jenkins сервере.
Git: Установлен на Jenkins сервере.
GitHub Репозиторий: С простым Java Maven проектом.
Тестовый сервер: Удаленная машина (даже VM или Docker-контейнер), доступная по SSH с Jenkins сервера. На ней должен быть установлен Java Runtime Environment (JRE) для запуска приложения.
SSH Key Pair: Сгенерированная пара ключей SSH (public/private) для доступа с Jenkins на тестовый сервер без пароля. Публичный ключ должен быть на тестовом сервере (~/.ssh/authorized_keys), приватный ключ будет добавлен в Jenkins.

## Часть 1: Подготовка окружения и приложения

1. Настройка Jenkins Server:
   
Зайдите в Jenkins -> Manage Jenkins -> Manage Plugins.
На вкладке Available, найдите и установите:
   * GitHub Integration (или GitHub plugin если используете старую версию)
   * Pipeline (обычно уже установлен)
   * SSH Agent Plugin (для выполнения SSH команд с использованием ключей)
   * Maven Integration plugin (для удобной настройки Maven, хотя для Pipeline это не строго обязательно)

Перезапустите Jenkins при необходимости.

* Настройка инструментов (JDK, Maven):
* Зайдите в Jenkins -> Manage Jenkins -> Global Tool Configuration.
* JDK:
  * Нажмите Add Maven.
  * Снимите галочку Install automatically.
  * Укажите Name (например, Maven_3.8.6) и MAVEN_HOME путь (например, /opt/apache-maven-3.8.6).

### 2. Подготовка GitHub репозитория с простым Java Maven проектом:

Создайте новый публичный репозиторий на GitHub (например, simple-java-app-ci-cd). В корне репозитория создайте следующую структуру файлов:

### ` pom.xml: `
```yaml
xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example.app</groupId>
    <artifactId>simple-java-app</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-api</artifactId>
            <version>5.8.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>5.8.1</            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>${maven.compiler.source}</source>
                    <target>${maven.compiler.target}</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <version>3.2.0</version>
                <configuration>
                    <archive>
                        <manifest>
                            <addClasspath>true</addClasspath>
                            <mainClass>com.example.app.App</mainClass>
                        </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.22.2</version>
            </plugin>
        </plugins>
    </build>
</project>
```
### `src/main/java/com/example/app/App.java:`
```java
java
package com.example.app;

import java.time.LocalDateTime;

public class App {
    public static void main(String[] args) {
        System.out.println("Hello from simple-java-app!");
        System.out.println("Current time: " + LocalDateTime.now());
        System.out.println("Version: " + App.class.getPackage().getImplementationVersion());
    }

    public String getGreeting() {
        return "Hello World!";
    }
}
```
### `src/test/java/com/example/app/AppTest.java:`
```java
java
package com.example.app;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertEquals;

public class AppTest {
    @Test
    void appHasAGreeting() {
        App classUnderTest = new App();
        assertEquals("Hello World!", classUnderTest.getGreeting(), "app should have a 'Hello World!' greeting");
    }
}
```

### 3. Настройка тестового сервера:

### `Установите JRE:`
```yaml
sudo apt update
sudo apt install openjdk-11-jre -y
```

### `Создайте директорию для деплоя:`
```yaml
mkdir -p /opt/simple-java-app
```
### `Настройте SSH доступ с Jenkins:`

На Jenkins сервере сгенерируйте SSH ключ (если его нет):
```yaml
ssh-keygen -t rsa -b 4096 -C "jenkins-deploy-key" -f ~/.ssh/jenkins-deploy-key
```
Скопируйте публичный ключ (~/.ssh/jenkins-deploy-key.pub) на тестовый сервер в ~/.ssh/authorized_keys для пользователя, под которым будет происходить деплой (например, jenkinsuser или root для простоты, но лучше создавать отдельного пользователя).
```yaml
# На Jenkins сервере:
ssh-copy-id -i ~/.ssh/jenkins-deploy-key.pub jenkinsuser@your_test_server_ip
# или вручную:
# scp ~/.ssh/jenkins-deploy-key.pub jenkinsuser@your_test_server_ip:/tmp/
# Затем на тестовом сервере:
# cat /tmp/jenkins-deploy-key.pub >> ~/.ssh/authorized_keys
# chmod 600 ~/.ssh/authorized_keys
```
Убедитесь, что вы можете зайти с Jenkins сервера на тестовый по SSH без пароля:

```yaml
ssh -i ~/.ssh/jenkins-deploy-key jenkinsuser@your_test_server_ip "hostname"
```
### * Добавьте приватный SSH ключ в Jenkins:

На Jenkins -> Manage Jenkins -> Manage Credentials.
Нажмите (global) -> Add Credentials.
Выберите Kind: SSH Username with private key.
Scope: Global.
ID: jenkins-deploy-key (запомните этот ID).
Description: SSH key for deployment to test server.
Username: jenkinsuser (пользователь на тестовом сервере).
Private Key: Выберите Enter directly и вставьте содержимое вашего приватного ключа (~/.ssh/jenkins-deploy-key).
Нажмите Create.

## Часть 2: Создание Jenkins Pipeline

1. Создание Jenkinsfile:

В корне вашего GitHub репозитория simple-java-app-ci-cd создайте файл с именем Jenkinsfile.

### `Jenkinsfile:`
```groovy
groovy
pipeline {
    // Агент, где будет выполняться пайплайн. 'any' означает, что он будет выполняться на любом доступном агенте.
    agent any

    // Переменные окружения, которые будут доступны на протяжении всего пайплайна
    environment {
        // Указываем ID credentials, которые мы добавили в Jenkins для SSH доступа к тестовому серверу
        SSH_CREDENTIALS_ID = 'jenkins-deploy-key'
        // IP или hostname вашего тестового сервера
        TEST_SERVER_HOST = 'your_test_server_ip_or_hostname'
        // Путь на тестовом сервере, куда будет деплоиться приложение
        REMOTE_DEPLOY_DIR = '/opt/simple-java-app'
        // Имя артефакта (JAR файла), который мы ожидаем получить после сборки Maven
        ARTIFACT_NAME = 'simple-java-app-1.0.0-SNAPSHOT.jar'
        // Имя сконфигурированного Maven в Jenkins Global Tool Configuration
        M2_HOME_NAME = 'Maven_3.8.6' // Замените на имя, которое вы указали в Jenkins
        // Имя сконфигурированного JDK в Jenkins Global Tool Configuration
        JAVA_HOME_NAME = 'JDK_11'    // Замените на имя, которое вы указали в Jenkins
    }

    // Стадии выполнения пайплайна
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                // Клонирование репозитория. Если репозиторий публичный, credentialsId не нужен.
                // git branch: 'main', url: 'https://github.com/your-github-user/simple-java-app-ci-cd.git'
                git branch: 'main', url: 'https://github.com/YOUR_GITHUB_USERNAME/simple-java-app-ci-cd.git'
            }
        }

        stage('Build') {
            steps {
                echo 'Building application with Maven...'
                // Используем сконфигурированные JDK и Maven
                withTools(jdk: "${JAVA_HOME_NAME}", maven: "${M2_HOME_NAME}") {
                    sh 'mvn clean package -DskipTests' // Собираем проект, пропуская тесты на этапе сборки (они будут отдельной стадией)
                }
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'
                withTools(jdk: "${JAVA_HOME_NAME}", maven: "${M2_HOME_NAME}") {
                    sh 'mvn test' // Запускаем юнит-тесты
                }
            }
            post {
                // Всегда публиковать отчеты по тестам, даже если тесты упали
                always {
                    junit '**/target/surefire-reports/*.xml' // Публикуем результаты тестов JUnit
                }
            }
        }

        stage('Archive Artifacts') {
            steps {
                echo 'Archiving built artifact...'
                // Архивируем JAR файл, чтобы его можно было скачать из Jenkins
                archiveArtifacts artifacts: "target/${ARTIFACT_NAME}", fingerprint: true
            }
        }

        stage('Deploy to Test Server') {
            steps {
                echo "Deploying artifact to ${TEST_SERVER_HOST}:${REMOTE_DEPLOY_DIR}..."
                // Используем ssh-agent для безопасного использования SSH ключа
                sshagent(credentials: [SSH_CREDENTIALS_ID]) {
                    // Копируем JAR файл на удаленный сервер
                    sh "scp target/${ARTIFACT_NAME} ${env.JENKINS_SSH_SSH_USERNAME}@${TEST_SERVER_HOST}:${REMOTE_DEPLOY_DIR}/"

                    // Выполняем удаленную команду: останавливаем старую версию, запускаем новую
                    // Это простая реализация. В реальном мире здесь может быть systemd сервис, Docker контейнер и т.д.
                    sh "ssh ${env.JENKINS_SSH_SSH_USERNAME}@${TEST_SERVER_HOST} " +
                       "\"pkill -f ${ARTIFACT_NAME} || true; " + // Останавливаем старый процесс, если он есть
                       "nohup java -jar ${REMOTE_DEPLOY_DIR}/${ARTIFACT_NAME} > ${REMOTE_DEPLOY_DIR}/app.log 2>&1 &\"" // Запускаем новую версию в фоне
                }
                echo "Deployment to ${TEST_SERVER_HOST} completed. Check logs on remote server for details."
            }
        }
    }

    // Действия, выполняемые после завершения всего пайплайна
    post {
        always {
            echo 'Pipeline finished.'
            // Здесь можно добавить уведомления (Slack, Email и т.д.)
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```























