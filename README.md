# Домашнее задание «TeamCity»

[Условие задания](https://github.com/netology-code/mnt-homeworks/blob/MNT-video/09-ci-05-teamcity/README.md).

## Стенд

В Yandex Cloud используются три виртуальные машины:

| VM | Ресурсы | Назначение |
| --- | --- | --- |
| `teamcity-server` | 2 vCPU, 4 ГБ RAM | `jetbrains/teamcity-server`, TeamCity 2026.2.1 |
| `teamcity-agent` | 2 vCPU, 4 ГБ RAM | `jetbrains/teamcity-agent:2026.2.1`, авторизованный агент |
| `nexus` | 2 vCPU, 4 ГБ RAM | Nexus Repository, установленный Ansible playbook |

**Отклонение от подготовки к заданию:** серверу выделены 2 vCPU вместо 4, поскольку квота `compute.instanceCores.count.ru-central1-d` составляет 2 vCPU. Сборки и публикация проверены на этой конфигурации.

Агент обращается к серверу по внутреннему адресу `http://10.10.2.21:8111`, Maven — к Nexus по `http://10.10.0.22:8081`. Для доступа сервера к GitHub используется внутренний CONNECT-прокси на VM Nexus: прямое соединение из зоны сервера давало периодические тайм-ауты. Прокси разрешает только `github.com:443`.

## Настройки сборки

В TeamCity создан проект `Netology TeamCity` и конфигурация `Build` на основе этого fork. Запущен Auto-detect build steps; Maven-шаги затем настроены вручную.

| Ветка | Условие шага | Maven goals |
| --- | --- | --- |
| `master` | `teamcity.build.branch` equals `master` | `clean deploy` |
| Остальные | `teamcity.build.branch` does not equal `master` | `clean test` |

VCS root отслеживает `refs/heads/master` и `+:refs/heads/(*)`. VCS-trigger запускает сборку автоматически при изменениях во всех ветках.

Настройки сохранены в репозитории:

- [Проект и Versioned Settings](.teamcity/Netology/project-config.xml).
- [Maven-шаги, условия, trigger и правило артефактов](.teamcity/Netology/buildTypes/Netology_Build.xml).
- [VCS root](.teamcity/Netology/vcsRoots/Netology_ExampleTeamcity.xml).

Versioned Settings включён в режиме загрузки XML из Git (`useFromVCS`). Изменения конфигурации выполняются коммитами в `.teamcity`; редактирование через UI отключено. [Сверка XML с действующими настройками сервера](evidence/config-match.json) подтверждает совпадение всех трёх файлов.

Maven settings с учётными данными Nexus загружен в проект TeamCity под именем `settings.xml` и выбран в обоих Maven-шагах. В Git хранится только [пример без секретов](settings.xml.example). Для повторного развёртывания нужно заполнить в нём учётные данные и загрузить файл как `settings.xml` через Project Settings → Maven Settings.

В [pom.xml](pom.xml) настроены `maven-public`, `maven-releases` и `maven-snapshots` Nexus. Используется версия `0.0.3-SNAPSHOT`: повторные сборки master публикуют новые timestamped snapshot-артефакты без перезаписи release-версии.

## Изменение приложения и merge

В ветке [feature/add_reply](https://github.com/YurtaevaSofia/example-teamcity/tree/feature/add_reply) добавлены:

- Метод [`Welcomer.sayReply()`](src/main/java/plaindoll/Welcomer.java), возвращающий реплику `May the moon guide your way, good hunter.`.
- Тест [`welcomerReplyContainsHunter()`](src/test/java/plaindoll/WelcomerTest.java), проверяющий наличие слова `hunter`.

После успешной автоматической сборки ветка внесена в master отдельным [merge-коммитом](https://github.com/YurtaevaSofia/example-teamcity/commit/037719c0c1484fe8ae834aba4dfa41783c73111c).

## Результаты проверок

Подтверждения сохранены из TeamCity REST API 1 ноября 2026 года. Номера соответствуют сборкам конфигурации `Netology_Build`.

| Сборка | Проверка | Результат | Подтверждение |
| --- | --- | --- | --- |
| №4 | Первая успешная сборка master: `clean test` | 5 тестов, SUCCESS | [JSON](evidence/build-4.json) |
| №6 | Master: `clean deploy` | 5 тестов, SUCCESS; JAR опубликован в Nexus | [JSON](evidence/build-6.json) |
| №8 | Автоматическая сборка `feature/add_reply` | VCS-trigger, 6 тестов, SUCCESS; deploy пропущен | [JSON](evidence/build-8.json) |
| №9 | Автоматический master после merge, до настройки артефактов | 6 тестов, SUCCESS; публичных артефактов нет | [JSON](evidence/build-9.json) |
| №11 | Повторный master после настройки артефактов | 6 тестов, SUCCESS; JAR сохранён в TeamCity | [JSON](evidence/build-11.json) |

Правило артефактов:

```text
target/plaindoll-*.jar => jars
```

В сборке №11 сохранён `jars/plaindoll-0.0.3-SNAPSHOT.jar`. Скачанный из TeamCity JAR проверен командой `java -jar`; приложение вывело приветствие и прощание без ошибок. Результат запуска включён в JSON сборки №11.

Публикация в `maven-snapshots` Nexus подтверждена [списком компонентов и путей JAR](evidence/nexus.json). Например, первая успешная публикация создала:

```text
org/netology/plaindoll/0.0.3-SNAPSHOT/plaindoll-0.0.3-20261101.152215-1.jar
```

Пароли Nexus и токен TeamCity в репозиторий не включены.
