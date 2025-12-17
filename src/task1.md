# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*

```sql
Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.063..0.064 rows=0 loops=1)
  Recheck Cond: (category IS NULL)
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.059..0.059 rows=0 loops=1)
        Index Cond: (category IS NULL)
Planning Time: 1.461 ms
Execution Time: 0.096 ms
```
   
*Объясните результат:*

Планировщик выбрал стратегию `Bitmap Scan` с использованием `BRIN` индекса `t_books_brin_cat_idx`. Поскольку `BRIN` индексы хранят суммарную информацию о диапазонах блоков, включая наличие `NULL` значений, `Bitmap Index Scan` смог быстро определить конкретные диапазоны физических страниц, где могут находиться искомые записи. Затем `Bitmap Heap Scan` считал только эти отобранные страницы для финальной проверки данных.

6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*

```sql
Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=14.256..14.256 rows=0 loops=1)
  Recheck Cond: ((category)::text = 'INDEX'::text)
  Rows Removed by Index Recheck: 150000
  Filter: ((author)::text = 'SYSTEM'::text)
  Heap Blocks: lossy=1224
  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.088..0.088 rows=12240 loops=1)
        Index Cond: ((category)::text = 'INDEX'::text)
Planning Time: 0.184 ms
Execution Time: 14.309 ms
```
   
*Объясните результат (обратите внимание на bitmap scan):*

Планировщик выбрал стратегию `Bitmap Heap Scan` с использованием BRIN индекса `t_books_brin_cat_idx`, однако выполнение оказалось неэффективным из-за физической неупорядоченности данных. Поскольку значения `category` распределены по таблице случайным образом, диапазон min/max практически каждого блока содержал искомое значение или перекрывал его. Это привело к тому, что индекс вернул ссылки на все физические страницы таблицы (`Heap Blocks: lossy=1224`), заставив механизм выполнения перечитывать их целиком и отбрасывать все 150000 строк на этапе `Recheck`, что по затратам эквивалентно полному сканированию таблицы.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*

```sql
Sort  (cost=3099.11..3099.12 rows=5 width=7) (actual time=33.821..33.822 rows=6 loops=1)
  Sort Key: category
  Sort Method: quicksort  Memory: 25kB
  ->  HashAggregate  (cost=3099.00..3099.05 rows=5 width=7) (actual time=33.636..33.637 rows=6 loops=1)
        Group Key: category
        Batches: 1  Memory Usage: 24kB
        ->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=7) (actual time=0.031..13.685 rows=150000 loops=1)
Planning Time: 0.494 ms
Execution Time: 34.017 ms
```
   
*Объясните результат:*

Планировщик выполнил полное сканирование таблицы `Seq Scan` с последующей агрегацией `HashAggregate` и сортировкой `Sort`, полностью проигнорировав существующий BRIN индекс `t_books_brin_cat_idx`. Это обусловлено природой BRIN индексов: они хранят лишь обобщенную информацию (минимум/максимум) для диапазонов страниц и не содержат упорядоченного списка всех значений, необходимого для эффективного выполнения операций `DISTINCT` или `ORDER BY`. Поэтому для извлечения уникальных категорий СУБД была вынуждена прочитать все строки таблицы.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*

```sql
Aggregate  (cost=3099.04..3099.05 rows=1 width=8) (actual time=18.420..18.421 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3099.00 rows=15 width=0) (actual time=18.416..18.417 rows=0 loops=1)
        Filter: ((author)::text ~~ 'S%'::text)
        Rows Removed by Filter: 150000
Planning Time: 1.349 ms
Execution Time: 18.680 ms
```
   
*Объясните результат:*

Планировщик выбрал стратегию `Seq Scan`, проигнорировав BRIN индекс `t_books_brin_author_idx`. Эффективность BRIN индексов напрямую зависит от физической упорядоченности данных, однако значения `author` распределены случайно. Из-за высокой энтропии диапазоны min/max для каждого блока индекса перекрывают друг друга и не позволяют исключить значимые участки таблицы из сканирования, поэтому планировщик счел полное чтение таблицы более дешевой операцией.

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*

```sql
Aggregate  (cost=3475.88..3475.89 rows=1 width=8) (actual time=33.114..33.115 rows=1 loops=1)
  ->  Seq Scan on t_books  (cost=0.00..3474.00 rows=750 width=0) (actual time=33.107..33.108 rows=1 loops=1)
        Filter: (lower((title)::text) ~~ 'o%'::text)
        Rows Removed by Filter: 149999
Planning Time: 0.127 ms
Execution Time: 33.145 ms
```
   
*Объясните результат:*
  
  Планировщик проигнорировал функциональный индекс `t_books_lower_title_idx` и выбрал полное сканирование, так как стандартные B-Tree индексы в PostgreSQL не поддерживают оптимизацию запросов с оператором `LIKE`.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*

```sql
Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=0.722..0.723 rows=0 loops=1)
  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
  Rows Removed by Index Recheck: 8810
  Heap Blocks: lossy=72
  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.034..0.034 rows=720 loops=1)
        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
Planning Time: 0.308 ms
Execution Time: 0.763 ms
```
   
*Объясните результат:*

Планировщик успешно применил составной BRIN индекс `t_books_brin_cat_auth_idx`, выполнив `Bitmap Heap Scan`. Благодаря использованию условий сразу по двум колонкам, индекс позволил сузить область поиска до 72 физических страниц, что значительно меньше полного объема таблицы. Тем не менее, из-за специфики BRIN (хранение диапазонов значений) и отсутствия физической сортировки данных, точность поиска осталась на уровне блоков (`lossy`), что потребовало считывания этих страниц и отсеивания 8810 лишних строк на этапе `Recheck`.