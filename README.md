# Домашнее задание к занятию 11 «TeamCity» - Юрочкин В.А.

## Репозиторий

Репозиторий с выполненным заданием:

```text
https://github.com/victoryurochkin/example-teamcity
```

В репозитории присутствуют:

```text
src/
pom.xml
.teamcity/
```

Каталог `.teamcity/` содержит мигрированную конфигурацию TeamCity в формате Kotlin DSL.

---

## 1. Общая схема стенда

Домашнее задание было выполнено на собственных ресурсах в среде виртуализации Proxmox.

<img width="3071" height="1440" alt="image" src="https://github.com/user-attachments/assets/9a06056d-60f2-42fe-8a59-73b77c56f08a" />


Вместо Yandex Cloud были подготовлены три виртуальные машины на Ubuntu Server 22.04:

| Назначение       | Hostname    |       IP-адрес | Роль                                          |
| ---------------- | ----------- | -------------: | --------------------------------------------- |
| TeamCity Server  | `tc-server` | `192.168.1.67` | Web-интерфейс TeamCity, хранение конфигураций |
| TeamCity Agent   | `tc-agent`  | `192.168.1.70` | Выполнение сборок Maven                       |
| Nexus Repository | `nexus`     | `192.168.1.71` | Maven-репозиторий для публикации артефактов   |

Все сервисы были развернуты в Docker-контейнерах.

Использованные сервисы:

```text
TeamCity Server: http://192.168.1.67:8111
Nexus Repository: http://192.168.1.71:8081
GitHub repository: https://github.com/victoryurochkin/example-teamcity
```

<img width="3071" height="1741" alt="image" src="https://github.com/user-attachments/assets/b19fd199-d5a0-42fb-a1b5-4e5f72904b95" />

<img width="3071" height="1745" alt="image" src="https://github.com/user-attachments/assets/f5bfb485-b763-4af4-9efc-14cfe97f0a86" />

---

## 2. Подготовка виртуальных машин

На всех виртуальных машинах была установлена Ubuntu Server 22.04.

Установлены базовые утилиты:

```bash
apt update
apt install -y curl wget git vim htop net-tools ca-certificates gnupg lsb-release
```

На всех трёх виртуальных машинах был установлен Docker и Docker Compose.

Проверка Docker:

```bash
docker --version
docker compose version
systemctl is-active docker
```

---

## 3. Разворачивание TeamCity Server

На сервере `tc-server` был подготовлен каталог для данных TeamCity:

```bash
mkdir -p /opt/teamcity/data
mkdir -p /opt/teamcity/logs

chown -R 1000:1000 /opt/teamcity
chmod -R u+rwX,g+rwX /opt/teamcity
```

TeamCity Server был запущен в Docker:

```bash
docker run -d \
  --name teamcity-server \
  --restart unless-stopped \
  -p 8111:8111 \
  -v /opt/teamcity/data:/data/teamcity_server/datadir \
  -v /opt/teamcity/logs:/opt/teamcity/logs \
  jetbrains/teamcity-server:latest
```

Проверка контейнера:

```bash
docker ps
docker logs --tail=150 teamcity-server
curl -I http://127.0.0.1:8111
```

TeamCity Server стал доступен по адресу:

```text
http://192.168.1.67:8111
```

В web-интерфейсе TeamCity была выполнена первоначальная настройка:

1. Принята лицензия.
2. Создан пользователь администратора.
3. Выполнен вход в web-интерфейс.
4. В настройках сервера указан корректный Server URL:

```text
http://192.168.1.67:8111
```

---

## 4. Разворачивание Nexus Repository

На сервере `nexus` был подготовлен каталог для данных Nexus:

```bash
mkdir -p /opt/nexus-data
chown -R 200:200 /opt/nexus-data
```

Nexus был запущен в Docker:

```bash
docker run -d \
  --name nexus \
  --restart unless-stopped \
  -p 8081:8081 \
  -v /opt/nexus-data:/nexus-data \
  sonatype/nexus3:latest
```

Проверка контейнера:

```bash
docker ps
docker logs --tail=100 nexus
```

Nexus стал доступен по адресу:

```text
http://192.168.1.71:8081
```

Первичная авторизация выполнялась под пользователем `admin`. Начальный пароль был получен из файла внутри контейнера/volume Nexus:

```bash
cat /opt/nexus-data/admin.password
```

В Nexus использовались стандартные Maven-репозитории:

```text
maven-releases
maven-snapshots
maven-public
```

Для публикации snapshot-артефактов использовался репозиторий:

```text
http://192.168.1.71:8081/repository/maven-snapshots/
```

Для TeamCity был создан пользователь Nexus, предназначенный для публикации Maven-артефактов. Данные этого пользователя были добавлены в Maven `settings.xml`, который затем был загружен в TeamCity.

---

## 5. Разворачивание TeamCity Agent

На сервере `tc-agent` был подготовлен каталог для агента:

```bash
mkdir -p /opt/teamcity-agent/conf
mkdir -p /opt/teamcity-agent/work
mkdir -p /opt/teamcity-agent/temp
mkdir -p /opt/teamcity-agent/system

chown -R 1000:1000 /opt/teamcity-agent
chmod -R u+rwX,g+rwX /opt/teamcity-agent
```

Так как в стандартном образе TeamCity Agent отсутствовал Maven, был подготовлен собственный Docker-образ агента с Maven и JDK.

Каталог для Dockerfile:

```bash
mkdir -p /opt/teamcity-agent-image
cd /opt/teamcity-agent-image
```

Dockerfile:

```Dockerfile
FROM jetbrains/teamcity-agent:latest

USER root

RUN apt-get update && \
    apt-get install -y maven openjdk-17-jdk && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

USER buildagent
```

Сборка образа:

```bash
docker build -t local/teamcity-agent-maven:latest .
```

Запуск агента:

```bash
docker run -d \
  --name teamcity-agent \
  --restart unless-stopped \
  -e SERVER_URL="http://192.168.1.67:8111" \
  -v /opt/teamcity-agent/conf:/data/teamcity_agent/conf \
  -v /opt/teamcity-agent/work:/opt/buildagent/work \
  -v /opt/teamcity-agent/temp:/opt/buildagent/temp \
  -v /opt/teamcity-agent/system:/opt/buildagent/system \
  local/teamcity-agent-maven:latest
```

Проверка установленного ПО внутри контейнера агента:

```bash
docker exec -it teamcity-agent bash -lc '
java -version
git --version
mvn -version
'
```

На агенте были доступны:

```text
Java
Git
Maven
```

После запуска агент появился в TeamCity в разделе:

```text
Agents → Unauthorized
```

Агент был авторизован через web-интерфейс TeamCity:

```text
Agents → Unauthorized → tc-agent → Authorize
```

После авторизации агент появился в пуле `Default` со статусом `Idle`.

---

<img width="2559" height="261" alt="image" src="https://github.com/user-attachments/assets/73331211-27ef-4875-8510-ea1c395b7c11" />

<img width="2558" height="199" alt="image" src="https://github.com/user-attachments/assets/4f3d438c-e8b1-4555-a228-200cb060c924" />

<img width="2553" height="248" alt="image" src="https://github.com/user-attachments/assets/3ca7ffb4-d1f4-45bd-beb8-97118c1cd190" />

---

## 6. Подготовка GitHub-репозитория

Был создан fork исходного репозитория:

```text
https://github.com/aragastmatb/example-teamcity
```

Fork-репозиторий:

```text
https://github.com/victoryurochkin/example-teamcity
```

В дальнейшем TeamCity был настроен на работу именно с fork-репозиторием:

```text
https://github.com/victoryurochkin/example-teamcity.git
```

---

## 7. Настройка `pom.xml`

В `pom.xml` были настроены параметры Maven-проекта и публикация артефактов в Nexus.

Основные параметры проекта:

```xml
<groupId>org.netology</groupId>
<artifactId>plaindoll</artifactId>
<packaging>jar</packaging>
<version>0.0.2-SNAPSHOT</version>
```

Для публикации snapshot-артефактов был настроен Nexus:

```xml
<distributionManagement>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://192.168.1.71:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

Также в проекте используется `maven-shade-plugin`, поэтому в результате сборки формируются два `.jar`-файла:

```text
original-plaindoll-0.0.2-SNAPSHOT.jar
plaindoll-0.0.2-SNAPSHOT.jar
```

---

## 8. Настройка Maven `settings.xml` в TeamCity

Для выполнения команды:

```bash
mvn clean deploy
```

TeamCity должен иметь доступ к Nexus.

Для этого был подготовлен Maven `settings.xml` с данными подключения к Nexus.

Пример структуры файла:

```xml
<settings>
    <servers>
        <server>
            <id>nexus-snapshots</id>
            <username>teamcity</username>
            <password>********</password>
        </server>
    </servers>
</settings>
```

Пароль в репозиторий не добавлялся.

Файл был загружен в TeamCity:

```text
Project Settings → Maven Settings → Upload settings file
```

После загрузки в build steps был выбран:

```text
User settings selection: settings.xml
```

---

## 9. Создание проекта в TeamCity

В TeamCity был создан проект:

```text
example-teamcity
```

Проект был создан на основе GitHub-репозитория:

```text
https://github.com/victoryurochkin/example-teamcity.git
```

Параметры VCS Root:

```text
Type of VCS: Git
Fetch URL: https://github.com/victoryurochkin/example-teamcity.git
Default branch: refs/heads/master
Branch specification: +:refs/heads/*
Authentication method: Password / personal access token
Username: victoryurochkin
```

Для доступа к GitHub использовался Personal Access Token. Сам токен в репозиторий не сохранялся.

---

## 10. Autodetect Maven build step

После подключения репозитория TeamCity автоматически определил Maven-проект и предложил Maven build step.

Autodetect обнаружил:

```text
Path to POM: pom.xml
Goals: clean test
```

Первоначальный Maven step был сохранён и затем изменён в соответствии с условиями задания.

---

## 11. Настройка условий сборки

По заданию требовалось:

```text
master → mvn clean deploy
остальные ветки → mvn clean test
```

Для этого в build configuration были настроены два Maven build step.

### Step 1 — Maven clean test

```text
Step name: Maven clean test
Runner: Maven
Goals: clean test
Path to POM file: pom.xml
User settings selection: settings.xml
```

Условие выполнения:

```text
teamcity.build.branch.is_default equals false
```

Это означает, что шаг выполняется только в ветках, отличных от default branch.

Например:

```text
feature/add_reply → mvn clean test
```

### Step 2 — Maven clean deploy

```text
Step name: Maven clean deploy
Runner: Maven
Goals: clean deploy
Path to POM file: pom.xml
User settings selection: settings.xml
```

Условие выполнения:

```text
teamcity.build.branch.is_default equals true
```

Это означает, что шаг выполняется только в default branch, то есть в `master`.

Например:

```text
master → mvn clean deploy
```

Итоговая логика:

| Ветка               | Выполняемый шаг    |
| ------------------- | ------------------ |
| `master`            | `mvn clean deploy` |
| `feature/add_reply` | `mvn clean test`   |

---

## 12. Проверка сборки master и публикации в Nexus

После настройки build steps была запущена сборка ветки `master`.

Для `master` сработала следующая логика:

```text
Maven clean test — skipped
Maven clean deploy — executed
```

Сборка завершилась успешно:

```text
BUILD SUCCESS
```

Артефакт был опубликован в Nexus:

```text
maven-snapshots/org/netology/plaindoll/0.0.2-SNAPSHOT/
```

Пример опубликованного артефакта:

```text
plaindoll-0.0.2-20260622.145956-5.jar
```

---

## 13. Миграция build configuration в репозиторий

Для выполнения пункта задания:

```text
Мигрируйте build configuration в репозиторий.
```

В TeamCity были включены Versioned Settings.

Раздел:

```text
Project Settings → Versioned Settings
```

Параметры:

```text
Synchronization enabled
Settings format: Kotlin
Settings path in VCS: .teamcity
Allow editing project settings via UI: enabled
Store passwords and API tokens outside of VCS: enabled
When build starts: always use current settings
```

После включения Versioned Settings TeamCity закоммитил конфигурацию в GitHub.

В репозитории появился каталог:

```text
.teamcity/
```

Таким образом, конфигурация TeamCity была мигрирована в репозиторий в формате Kotlin DSL.

После изменения настроек build configuration, в том числе artifact rules, TeamCity также обновил `.teamcity/` в репозитории.

---

## 14. Создание ветки `feature/add_reply`

В GitHub была создана отдельная ветка:

```text
feature/add_reply
```

Ветка была создана от `master`.

---

## 15. Добавление нового метода в `Welcomer`

В файл:

```text
src/main/java/plaindoll/Welcomer.java
```

был добавлен новый метод:

```java
public String sayReply() {
    return "A brave hunter always finds a way.";
}
```

Метод возвращает строку, содержащую слово:

```text
hunter
```

Это соответствует условию задания.

---

## 16. Добавление теста для нового метода

В файл:

```text
src/test/java/plaindoll/WelcomerTest.java
```

был добавлен новый тест:

```java
@Test
public void welcomerSaysReplyWithHunter() {
    assertThat(welcomer.sayReply(), containsString("hunter"));
}
```

После добавления теста общее количество тестов стало:

```text
6
```

---

## 17. Проверка автоматической сборки feature-ветки

После push изменений в ветку:

```text
feature/add_reply
```

TeamCity автоматически запустил сборку этой ветки.

Ожидаемое поведение для feature-ветки:

```text
Maven clean test — executed
Maven clean deploy — skipped
```

Сборка feature-ветки завершилась успешно:

```text
Tests passed: 6
```

Это подтвердило, что условие для не-default веток работает корректно.

---

## 18. Merge `feature/add_reply` в `master`

После успешной сборки feature-ветки был создан Pull Request:

```text
feature/add_reply → master
```

Pull Request:

```text
Add hunter reply
```

PR был успешно смержен в `master`.

После merge TeamCity автоматически запустил новую сборку ветки `master`.

Сборка `master` после merge прошла успешно:

```text
Tests passed: 6
```

Для `master` снова отработала логика:

```text
Maven clean test — skipped
Maven clean deploy — executed
```

---

## 19. Проверка отсутствия `.jar` в TeamCity Artifacts до настройки artifact rules

После сборки `master` была проверена вкладка:

```text
Build → Artifacts
```

На этом этапе `.jar`-файл ещё не публиковался в TeamCity Artifacts, так как artifact rules ещё не были настроены.

Это соответствует пункту задания:

```text
Убедитесь, что нет собранного артефакта в сборке по ветке master.
```

---

## 20. Настройка публикации `.jar` в TeamCity Artifacts

В build configuration были настроены artifact rules.

Раздел:

```text
Build → Settings → General
```

Поле:

```text
Artifact paths
```

Значение:

```text
target/*.jar => jars
```

Это правило означает:

```text
взять все .jar-файлы из target/
и опубликовать их в TeamCity Artifacts в каталог jars/
```

После сохранения настройки TeamCity обновил конфигурацию в `.teamcity/`.

---

## 21. Повторная сборка master и проверка артефактов

После настройки artifact rules была повторно запущена сборка `master`.

Сборка завершилась успешно:

```text
Build #7
Branch: master
Status: Success
Tests passed: 6
```

В build log подтверждено:

```text
BUILD SUCCESS
```

Также подтверждена публикация `.jar` как TeamCity Artifacts:

```text
Collecting files to publish: [target/*.jar => jars]
Publishing 2 files using [WebPublisher]: target/*.jar => jars
```

На вкладке:

```text
Build #7 → Artifacts
```

появилась папка:

```text
jars
```

Внутри опубликованы файлы:

```text
original-plaindoll-0.0.2-SNAPSHOT.jar
plaindoll-0.0.2-SNAPSHOT.jar
```

Два файла появились из-за использования `maven-shade-plugin`.


<img width="3071" height="1752" alt="image" src="https://github.com/user-attachments/assets/6e8688c3-5d2e-4246-87a6-6b0792cdcaa7" />

---

## 22. Итоговая проверка

В результате выполнены все основные пункты задания:

| Пункт                                         | Статус    |
| --------------------------------------------- | --------- |
| TeamCity Server развернут                     | Выполнено |
| TeamCity Agent развернут                      | Выполнено |
| Agent авторизован                             | Выполнено |
| Nexus Repository развернут                    | Выполнено |
| Fork репозитория создан                       | Выполнено |
| Проект в TeamCity создан из fork              | Выполнено |
| Maven autodetect выполнен                     | Выполнено |
| `master → mvn clean deploy` настроено         | Выполнено |
| `feature/* → mvn clean test` настроено        | Выполнено |
| Maven `settings.xml` загружен в TeamCity      | Выполнено |
| `pom.xml` настроен на Nexus                   | Выполнено |
| Артефакт опубликован в Nexus                  | Выполнено |
| Build configuration мигрирована в репозиторий | Выполнено |
| Ветка `feature/add_reply` создана             | Выполнено |
| Метод с `hunter` добавлен                     | Выполнено |
| Тест на `hunter` добавлен                     | Выполнено |
| Feature-сборка запустилась автоматически      | Выполнено |
| Feature-сборка прошла успешно                 | Выполнено |
| Ветка смержена в `master`                     | Выполнено |
| Master-сборка после merge прошла успешно      | Выполнено |
| `.jar` добавлен в TeamCity Artifacts          | Выполнено |
| Конфигурация сохранена в `.teamcity/`         | Выполнено |

---

## 23. Ссылки

Репозиторий:

```text
https://github.com/victoryurochkin/example-teamcity
```

Каталог с конфигурацией TeamCity:

```text
https://github.com/victoryurochkin/example-teamcity/tree/master/.teamcity
```

---

## Итог

Домашнее задание выполнено.

В рамках работы был развёрнут полный CI/CD-стенд на собственных виртуальных машинах в Proxmox:

```text
GitHub → TeamCity → Maven → Nexus → TeamCity Artifacts
```

Ветка `master` выполняет полноценную сборку и публикацию артефакта в Nexus через:

```text
mvn clean deploy
```

Ветки разработки выполняют только тесты через:

```text
mvn clean test
```

Конфигурация TeamCity мигрирована в GitHub в каталог:

```text
.teamcity/
```

Финальная сборка `master` успешно прошла, Maven-тесты завершились без ошибок, артефакты `.jar` опубликованы как в Nexus, так и в TeamCity Artifacts.
