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
   "Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.97 rows=1 width=33) (actual time=0.005..0.006 rows=1 loops=1)"
   "Planning Time: 0.811 ms"
   "Execution Time: 0.023 ms"
   
   *Объясните результат:*
   Запрос использует созданный индекс, потому что он был создан для категории, и BRIN-индекс эффективен при работе с большими таблицами.

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
   "Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=48.976..48.979 rows=0 loops=1)"
   "  Recheck Cond: ((category)::text = 'INDEX'::text)"
   "  Rows Removed by Index Recheck: 150003"
   "  Filter: ((author)::text = 'SYSTEM'::text)"
   "  Heap Blocks: lossy=1280"
   "  ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.135..0.136 rows=12800 loops=1)"
   "        Index Cond: ((category)::text = 'INDEX'::text)"
   "Planning Time: 0.528 ms"
   "Execution Time: 49.022 ms"
   
   *Объясните результат (обратите внимание на bitmap scan):*
   Запрос использует индекс по категории, что позволяет сразу определить отсутствие нужных категорий и значительно ускоряет выполнение запроса.

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   "Sort  (cost=3155.15..3155.16 rows=5 width=7) (actual time=97.181..97.184 rows=7 loops=1)"
   "  Sort Key: category"
   "  Sort Method: quicksort  Memory: 25kB"
   "  ->  HashAggregate  (cost=3155.04..3155.09 rows=5 width=7) (actual time=97.094..97.099 rows=7 loops=1)"
   "        Group Key: category"
   "        Batches: 1  Memory Usage: 24kB"
   "        ->  Seq Scan on t_books  (cost=0.00..2780.03 rows=150003 width=7) (actual time=0.026..31.706 rows=150003 loops=1)"
   "Planning Time: 5.628 ms"
   "Execution Time: 97.256 ms"
   
   *Объясните результат:*
   Запрос не использует созданный BRIN-индекс по категориям, поскольку он не помогает в упорядочивании записей по категории, так как BRIN-индекс основывается на физическом расположении данных. Поэтому запрос выполняет последовательное сканирование. Однако BRIN-индекс будет эффективно работать на уже отсортированных данных.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   "Aggregate  (cost=3155.08..3155.09 rows=1 width=8) (actual time=41.150..41.152 rows=1 loops=1)"
   "  ->  Seq Scan on t_books  (cost=0.00..3155.04 rows=15 width=0) (actual time=41.139..41.139 rows=0 loops=1)"
   "        Filter: ((author)::text ~~ 'S%'::text)"
   "        Rows Removed by Filter: 150003"
   "Planning Time: 2.316 ms"
   "Execution Time: 41.207 ms"
   
   *Объясните результат:*
   BRIN-индекс также не был использован, потому что он основан на построении диапазонов в хранении данных, а не на реальной систематизации, что ограничивает его эффективность в данном случае.

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
   "Aggregate  (cost=3531.92..3531.93 rows=1 width=8) (actual time=96.956..96.958 rows=1 loops=1)"
   "  ->  Seq Scan on t_books  (cost=0.00..3530.05 rows=750 width=0) (actual time=96.875..96.946 rows=1 loops=1)"
   "        Filter: (lower((title)::text) ~~ 'o%'::text)"
   "        Rows Removed by Filter: 150002"
   "Planning Time: 0.554 ms"
   "Execution Time: 97.008 ms"
   
   *Объясните результат:*
   Индекс не используется, потому что для сопоставления индекса с данными нужно сначала привести заголовки книг к нижнему регистру. В результате все равно выполняется последовательное сканирование.

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
   "Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=1.784..1.785 rows=0 loops=1)"
   "  Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "  Rows Removed by Index Recheck: 8821"
   "  Heap Blocks: lossy=128"
   "  ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.066..0.066 rows=1280 loops=1)"
   "        Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))"
   "Planning Time: 0.441 ms"
   "Execution Time: 1.814 ms"
   
   *Объясните результат:*
   Индекс по категории и автору позволил быстро найти нужную категорию в сочетании с автором, что значительно ускорило выполнение запроса.
