## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*

```sql
Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.043..0.044 rows=1 loops=1)
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.029..0.030 rows=1 loops=1)
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
Planning Time: 0.681 ms
Execution Time: 0.098 ms

```
    
*Объясните результат:*
    
Планировщик использовал GIN индекс `t_books_fts_idx`. Поскольку выражение `to_tsvector` в запросе полностью совпадает с определением индекса, был выполнен `Bitmap Index Scan` для быстрого нахождения идентификаторов строк, содержащих лексему 'expert'. Это позволило точечно извлечь единственную нужную запись через `Bitmap Heap Scan` ,избежав медленного парсинга и сканирования всего текста таблицы.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*

```sql
Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.112..0.113 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 0.445 ms
Execution Time: 0.148 ms

```
     
*Объясните результат:*

Планировщик выбрал наиболее эффективную стратегию `Index Scan` с использованием индекса первичного ключа `t_lookup_pk`. Поскольку запрос выполняет поиск по точному совпадению уникального ключа, механизм B-Tree позволяет напрямую перейти к нужному листу индекса, получить указатель на физическую строку в heap и считать данные.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*

```sql
Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.351..0.353 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 1.363 ms
Execution Time: 0.537 ms

```
     
*Объясните результат:*

Планировщик использовал `Index Scan` по первичному ключу `t_lookup_clustered_pkey`. Несмотря на то, что таблица была кластеризована по этому индексу, для поиска одной конкретной записи механизм выполнения идентичен работе с обычной таблицей: поиск ключа в B-Tree и чтение соответствующей страницы данных. Преимущества кластеризации в данном случае не проявляются, так как они актуальны преимущественно для выборки диапазонов строк, где физическое соседство данных снижает количество операций ввода-вывода.

15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*

```sql
Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.057..0.057 rows=0 loops=1)
  Index Cond: ((item_value)::text = 'T_BOOKS'::text)
Planning Time: 0.495 ms
Execution Time: 0.089 ms

```
     
*Объясните результат:*

Планировщик выбрал стратегию `Index Scan` с использованием вторичного индекса `t_lookup_value_idx`. Это стандартный метод доступа для точечных запросов по индексируемому полю.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*

```sql
Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.451..0.452 rows=0 loops=1)
  Index Cond: ((item_value)::text = 'T_BOOKS'::text)
Planning Time: 0.433 ms
Execution Time: 0.551 ms

```
     
*Объясните результат:*

Планировщик выполнил `Index Scan` по индексу `t_lookup_clustered_value_idx`. Несмотря на то, что таблица кластеризована, физическая сортировка данных выполнена по первичному ключу `item_key`, а не по столбцу поиска `item_value`. Поскольку корреляция между этими колонками отсутствует, кластеризация не дает никаких преимуществ для этого запроса, и поиск выполняется через стандартный механизм вторичного индекса, аналогично некластеризованной таблице.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
*Сравнение:*

Производительность поиска по столбцу `item_value` в обеих таблицах структурно идентична. Кластеризация таблицы `t_lookup_clustered` выполнена по первичному ключу (`item_key`), поэтому физический порядок строк никак не коррелирует со значениями в столбце `item_value`. В обоих случаях планировщик использует вторичный B-Tree индекс, что приводит к случайному доступу к страницам кучи (heap) при извлечении данных. Преимущества кластеризации проявляются только при поиске по ключу кластеризации или коррелирующим с ним колонкам.