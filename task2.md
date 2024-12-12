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
    "Bitmap Heap Scan on t_books  (cost=21.03..1367.60 rows=750 width=33) (actual time=0.026..0.028 rows=1 loops=1)"
    "  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
    "  Heap Blocks: exact=1"
    "  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.014..0.014 rows=1 loops=1)"
    "        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
    "Planning Time: 0.737 ms"
    "Execution Time: 0.056 ms"
    
    *Объясните результат:*
    Создан индекс GIN, с помощью функции to_tsvector можно найти названия, содержащие слово 'english'. Это позволяет эффективно оптимизировать поиск по таким словам.

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
     "Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.031..0.032 rows=1 loops=1)"
    "  Index Cond: ((item_key)::text = '0000000455'::text)"
    "Planning Time: 0.179 ms"
    "Execution Time: 0.059 ms"
     
     *Объясните результат:*
     При создании таблицы для первичного ключа автоматически был создан индекс. В результате запрос использовал этот индекс, что ускорило поиск конкретного значения.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     "Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.103..0.104 rows=1 loops=1)"
    "  Index Cond: ((item_key)::text = '0000000455'::text)"
    "Planning Time: 0.532 ms"
    "Execution Time: 0.125 ms"
         
     *Объясните результат:*
     При копировании таблицы был перенесен первичный ключ, который также использовался как индекс, что обеспечило ускорение поиска, аналогично предыдущему случаю.

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
     "Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.046..0.046 rows=0 loops=1)"
    "  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
    "Planning Time: 11.028 ms"
    "Execution Time: 0.086 ms"
     
     *Объясните результат:*
     Запрос использует индекс B-tree, что ускоряет поиск конкретного значения.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     "Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.061..0.061 rows=0 loops=1)"
    "  Index Cond: ((item_value)::text = 'T_BOOKS'::text)"
    "Planning Time: 0.661 ms"
    "Execution Time: 0.086 ms"
     
     *Объясните результат:*
     Аналогично предыдущему пункту, но для кластеризованной таблицы, индекс ускоряет поиск конкретного значения.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     Запрос на кластеризованной таблице работает быстрее, поскольку данные в ней хранятся на диске в отсортированном порядке, в отличие от обычной таблицы. Это позволяет запросу быстрее находить нужные записи, когда данные уже отсортированы.
