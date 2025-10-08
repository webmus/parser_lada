# Ошибка MySQL 1100 при выборке из `dle_storage`

## Что происходит
В модуле публикации `addPost()` перед вставкой новости вызывается `ensurePostUrlHashConstraint()`. Метод пытается убедиться, что у таблицы `{prefix}_post` есть поле `url_hash` и уникальный индекс `idx_url_hash`. Если индекс создать не удаётся (в таблице остаются дубли), код ставит явную блокировку: `LOCK TABLES {prefix}_post WRITE`.【F:parsing-lada.xml†L4738-L4781】【F:parsing-lada.xml†L5067-L5084】

Пока блокировка активна, движок DLE выполняет служебный запрос `SELECT * FROM {prefix}_storage`. Так как таблица `dle_storage` не была перечислена в списке блокируемых, MySQL отвечает ошибкой `1100 Table 'dle_storage' was not locked with LOCK TABLES`, а публикация обрывается.

## Как исправить проблему
1. **Очистить дублирующиеся URL.**
   ```sql
   -- найти дубли
   SELECT url_hash, COUNT(*) AS cnt
   FROM dle_post
   WHERE url_hash != ''
   GROUP BY url_hash
   HAVING cnt > 1;

   -- удалить «лишние» записи вручную или оставить одну из каждой группы
   DELETE p1 FROM dle_post p1
   JOIN dle_post p2
     ON p1.url_hash = p2.url_hash AND p1.id > p2.id;
   ```
   Повторите `SELECT`, пока результат пустой.

2. **Дать функции создать индекс.** После очистки снова запустите парсер. `ensurePostUrlHashConstraint()` увидит, что дубликатов нет, и создаст уникальный индекс `idx_url_hash`. Как только индекс появится, путь с `LOCK TABLES` больше не используется, и ошибка пропадёт.【F:parsing-lada.xml†L4768-L4787】【F:parsing-lada.xml†L5067-L5079】

3. **Проверить, что индекс существует.**
   ```sql
   SHOW INDEX FROM dle_post WHERE Key_name = 'idx_url_hash';
   ```
   Если индекс на месте, блокировки не понадобятся, и запросы к `dle_storage` снова выполнятся штатно.

> Альтернатива: если по каким-то причинам нельзя чистить таблицу сейчас, временно можно убрать блокировку из кода, однако без уникального индекса вернётся проблема с дублирующими публикациями.
