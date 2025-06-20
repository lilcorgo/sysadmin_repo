# План восстановления инфраструктуры MediaWiki

Этот документ описывает шаги по восстановлению инфраструктуры MediaWiki на случай сбоев.

---

## 1. Введение

В этом плане описаны действия по:

- Восстановлению работы веб-серверов (MediaWiki)
- Переключению на реплику базы данных при отказе главной БД
- Восстановлению базы данных из резервной копии
- Восстановлению файлов MediaWiki из резервной копии
- Восстановлению балансировщика нагрузки

---

## 2. Термины и обозначения

- **antiochus** (158.160.105.240) — Master БД + MediaWiki
- **aquilia** (158.160.105.2) — Standby БД + MediaWiki  
- **aurelia** (46.21.245.72) — Load Balancer + Мониторинг + Backup Storage
- **Master** — основная база данных (antiochus)
- **Standby** — резервная реплика базы данных (aquilia)
- **Backup** — резервная копия данных, хранящаяся на aurelia в `/opt/backup/central/`

---

## 3. Восстановление веб-сервера (MediaWiki)

### 3.1. Симптомы
- Пользователи не могут открыть сайт
- Ошибка соединения или таймаут в браузере
- HTTP 502/503 ошибки

### 3.2. План действий

1. Подключиться по SSH к **aurelia**
2. Проверить статус балансировщика:
```bash
sudo systemctl status nginx
```
3. Если балансировщик работает, проверить доступность сайта:
```bash
curl -I http://localhost/
```
4. Проверить отдельные узлы:
```bash
curl -I http://158.160.105.240:8080/  # antiochus
curl -I http://158.160.105.2:8080/    # aquilia
```
5. Если один из узлов не отвечает, подключиться по SSH к этому узлу
6. Перезапустить сервисы MediaWiki:
```bash
sudo systemctl restart nginx
sudo systemctl restart php8.1-fpm
```
7. Проверить статус сервисов:
```bash
sudo systemctl status nginx php8.1-fpm
```
8. Повторить проверку доступности


---

## 4. Восстановление балансировщика нагрузки

### 4.1. Симптомы
- Полная недоступность сайта
- Все пользователи не могут попасть на сайт

### 4.2. План действий

1. Подключиться по SSH к **aurelia**
2. Проверить статус nginx:
```bash
sudo systemctl status nginx
```
3. Если nginx не работает, перезапустить:
```bash
sudo systemctl restart nginx
```
4. Проверить конфигурацию nginx:
```bash
sudo nginx -t
```
5. Если есть ошибки конфигурации, проверить файл:
```bash
sudo nano /etc/nginx/sites-available/mediawiki
```
6. Проверить upstream серверы в конфигурации
7. Перезапустить nginx после исправления:
```bash
sudo systemctl reload nginx
```
8. Проверить доступность:
```bash
curl -I http://localhost/
```

---

## 5. Автоматическое переключение на реплику базы данных

### 5.1. Симптомы
- Главная БД (antiochus) недоступна или не отвечает
- Ошибка 500 при обращении к сайту

### 5.2. План действий

1. Проверить доступность сайта через балансировщик на **aurelia**:
```bash
curl -I http://localhost/
```
2. Если сайт отвечает с кодом 200/301 — автоматический failover сработал
3. Подключиться по SSH к **aquilia** и проверить роль БД:
```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```
   - Если `false` — aquilia стала новой master
   - Если `true` — aquilia всё ещё standby
4. Проверить репликацию на **antiochus** (если доступен):
```bash
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
```
5. При необходимости вручную перевести **aquilia** в режим master:
```bash
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main
```

---

## 6. Восстановление базы данных из резервной копии

### 6.1. Симптомы
- База данных повреждена или удалена
- Ошибка 500 при обращении к сайту
- MediaWiki выдает ошибки подключения к БД

### 6.2. План действий

1. Найти последний бэкап на **aurelia**:
```bash
ls -t /opt/backup/central/db_antiochus_*.sql.gz | head -n1
```
2. Скопировать бэкап на **antiochus**:
```bash
scp /opt/backup/central/db_antiochus_*.sql.gz eugene@158.160.105.240:/tmp/
```
3. Подключиться по SSH к **antiochus**
4. Остановить подключения к БД и удалить старую:
```bash
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'my_wiki' AND pid != pg_backend_pid();"
sudo -u postgres dropdb my_wiki
```
5. Создать новую базу:
```bash
sudo -u postgres createdb -O wikiuser my_wiki
```
6. Восстановить из бэкапа:
```bash
gunzip -c /tmp/db_antiochus_*.sql.gz | sudo -u postgres psql my_wiki
```
7. Проверить количество таблиц:
```bash
sudo -u postgres psql my_wiki -c "SELECT count(*) FROM pg_tables WHERE schemaname = 'mediawiki';"
```
8. Перезапустить MediaWiki:
```bash
sudo systemctl restart nginx php8.1-fpm
```
9. Проверить сайт:
```bash
curl -I http://158.160.105.240:8080/
```


---

## 7. Восстановление файлов MediaWiki из резервной копии

### 7.1. Симптомы
- Файлы MediaWiki повреждены или удалены
- HTTP 404 ошибки
- Белые страницы или ошибки PHP

### 7.2. План действий

1. Найти последний бэкап файлов на **aurelia**:
```bash
ls -t /opt/backup/central/mediawiki_antiochus_*.tar.gz | head -n1
```
2. Скопировать бэкап на **antiochus**:
```bash
scp /opt/backup/central/mediawiki_antiochus_*.tar.gz eugene@158.160.105.240:/tmp/
```
3. Подключиться по SSH к **antiochus**
4. Создать резервную копию текущих файлов (если есть):
```bash
sudo mv /var/www/mediawiki /var/www/mediawiki.backup.$(date +%Y%m%d_%H%M) 2>/dev/null || true
```
5. Восстановить файлы из архива:
```bash
cd /var/www
sudo tar -xzf /tmp/mediawiki_antiochus_*.tar.gz
```
6. Установить правильные права доступа:
```bash
sudo chown -R www-data:www-data /var/www/mediawiki
sudo chmod 600 /var/www/mediawiki/LocalSettings.php
```
7. Перезапустить веб-сервер:
```bash
sudo systemctl restart nginx php8.1-fpm
```
8. Проверить сайт:
```bash
curl -I http://158.160.105.240:8080/
```




---

## 8. Контакты

### 8.1. Ответственные лица
- **Системный администратор**: Eugene Ditkovsky
- **Резервный контакт**: []

