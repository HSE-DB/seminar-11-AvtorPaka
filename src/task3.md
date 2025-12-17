## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*

```sql
Bitmap Heap Scan on test_cluster  (cost=5584.59..20226.93 rows=500667 width=39) (actual time=13.416..105.104 rows=500466 loops=1)
  Recheck Cond: (category = 'A'::text)
  Heap Blocks: exact=8334
  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5459.43 rows=500667 width=0) (actual time=12.307..12.308 rows=500466 loops=1)
        Index Cond: (category = 'A'::text)
Planning Time: 1.095 ms
Execution Time: 118.031 ms

```
    
*Объясните результат:*

Планировщик выбрал стратегию `Bitmap Heap Scan` с использованием индекса `test_cluster_cat_idx`. Поскольку `category` низко-селективен (всего 2 возможных значения, с вероятностью $\approx 50\%$ каждое), а данные в таблице распределены хаотично (не кластеризованы), искомые строки 'A' находятся практически на каждой физической странице. `Bitmap Scan` позволяет оптимизировать чтение, сгруппировав обращения к страницам, но из-за разброса данных СУБД всё равно вынуждена считывать и обрабатывать огромный объем блоков таблицы - 8334, что делает выполнение ресурсоемким.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*

```sql
workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
completed in 559 ms
```

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*

```sql
Bitmap Heap Scan on test_cluster  (cost=5584.59..20176.93 rows=500667 width=39) (actual time=9.649..57.931 rows=500466 loops=1)
  Recheck Cond: (category = 'A'::text)
  Heap Blocks: exact=4171
  ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5459.43 rows=500667 width=0) (actual time=9.089..9.090 rows=500466 loops=1)
        Index Cond: (category = 'A'::text)
Planning Time: 0.478 ms
Execution Time: 72.291 ms

```
    
*Объясните результат:*

Планировщик сохранил стратегию `Bitmap Heap Scan`, но эффективность выполнения кардинально выросла благодаря кластеризации данных. Все строки с категорией 'A' теперь расположены на диске последовательно, что позволило считывать только те физические страницы, которые содержат целевые данные. Это подтверждается `Heap Blocks`, которые сократились ровно вдвое (с 8334 до 4171), так как системе больше не нужно обращаться ко всем страницам таблицы ради сбора разбросанных записей.

6. Сравните производительность до и после кластеризации:
    
*Сравнение:*

Кластеризация обеспечила прирост производительности, сократив время выполнения запроса с 118 мс до 72 мс. Главный фактор ускорения — уменьшение объема операций ввода-вывода: количество считанных страниц данных `Heap Blocks` уменьшилось в два раза (с 8334 до 4171). До кластеризации искомые строки были разбросаны по всей таблице, вынуждая считывать почти каждый блок, тогда как после операции они оказались сгруппированы физически, что позволило читать данные последовательно вместо хаотичного доступа.