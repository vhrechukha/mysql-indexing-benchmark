# Populate Table with 40 Million Rows

```
CREATE TABLE IF NOT EXISTS users (
   id BIGINT AUTO_INCREMENT PRIMARY KEY,
   name VARCHAR(255),
   date_of_birth DATE,
   email VARCHAR(255)
) ENGINE=InnoDB;

ALTER TABLE users DISABLE KEYS;

SET GLOBAL innodb_flush_log_at_trx_commit = 2;

CREATE PROCEDURE populate_users(IN start_i INT)
BEGIN
DECLARE i INT DEFAULT start_i;
DECLARE batch_size INT DEFAULT 1000;
DECLARE end_i INT;

    WHILE i <= 40000000 DO
        SET end_i = i + batch_size - 1;
        IF end_i > 40000000 THEN
            SET end_i = 40000000;
        END IF;
        
        INSERT INTO users (name, date_of_birth, email)
        SELECT CONCAT('User', seq_final.seq_final) AS name, '1980-01-01' AS date_of_birth, CONCAT('user', seq_final.seq_final, '@example.com') AS email
        FROM (
            SELECT i + seq_combined.seq AS seq_final
            FROM (
                SELECT 0 AS seq UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4 
                UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9
            ) AS seq_1
            CROSS JOIN (
                SELECT 0 AS seq UNION ALL SELECT 10 UNION ALL SELECT 20 UNION ALL SELECT 30 UNION ALL SELECT 40 
                UNION ALL SELECT 50 UNION ALL SELECT 60 UNION ALL SELECT 70 UNION ALL SELECT 80 UNION ALL SELECT 90
            ) AS seq_2
            CROSS JOIN (
                SELECT 0 AS seq UNION ALL SELECT 100 UNION ALL SELECT 200 UNION ALL SELECT 300 UNION ALL SELECT 400 
                UNION ALL SELECT 500 UNION ALL SELECT 600 UNION ALL SELECT 700 UNION ALL SELECT 800 UNION ALL SELECT 900
            ) AS seq_combined
        ) AS seq_final
        WHERE seq_final.seq_final BETWEEN i AND end_i;

        SET i = i + batch_size;
    END WHILE;
END;

CALL populate_users(0);

ALTER TABLE users ENABLE KEYS;
```
# Compare Indexes

When comparing the performance of queries with and without indexes, the `EXPLAIN` statement provides insight into how MySQL executes the query. Below are the key differences observed in the execution plan when querying a table with and without an index on the `date_of_birth` column.

## 1. Without Indexes

Query: `SELECT * FROM users WHERE date_of_birth = '1980-01-01';`

`"id","select_type","table","partitions","type","possible_keys","key","key_len","ref","rows","filtered","Extra"
1,SIMPLE,users,,ALL,,,,,35973608,10.0,Using where`

**Interpretation:**
- **Type:** `ALL` indicates a full table scan, checking every row in the table to find matches
- **Rows Examined:** MySQL estimates it needs to examine approximately 35.9 million rows

## 2. With BTREE Indexes

Query: `SELECT * FROM users WHERE date_of_birth = '1980-01-01';`

`"id","select_type","table","partitions","type","possible_keys","key","key_len","ref","rows","filtered","Extra"
1,SIMPLE,users,,ref,idx_dob_btree,idx_dob_btree,"4",const,17986804,100.0,`

**Interpretation:**
- **Type:** `ref` indicates that MySQL is using the `BTREE` index to directly find the rows where `date_of_birth` matches the specified value
- **Rows Examined:** MySQL estimates it needs to examine approximately 17.9 million rows

## 2. With HASH Indexes

Does not support HASH indexes.

# Check insert speed difference with different innodb_flush_log_at_trx_commit value and different ops per second

```
CREATE PROCEDURE insert_data(IN num_rows INT)
   BEGIN
      DECLARE i INT DEFAULT 1;
      WHILE i <= num_rows DO
      INSERT INTO users (name, date_of_birth, email)
      VALUES (CONCAT('User', i), '1980-01-01', CONCAT('user', i, '@example.com'));
      SET i = i + 1;
   END WHILE;
END;
```

## Execution Steps

1. Set the `innodb_flush_log_at_trx_commit` Value

   `SET GLOBAL innodb_flush_log_at_trx_commit = x;`

2. Prepare the Environment

   `TRUNCATE TABLE users;`

3. Enable Profiling

   ```
   SET profiling = 1;
   CALL insert_test_data(10000);
   SHOW PROFILE FOR QUERY 1;
   SHOW PROFILES;
   ```

## Summary and Comparison

1. With `innodb_flush_log_at_trx_commit = 1`:

- **Total time for 10,000 inserts ≈** 6.57325 seconds
- **Operations Per Second (OPS):** 1,521 inserts/second

2. With `innodb_flush_log_at_trx_commit = 2`:

- **Total time for 10,000 inserts ≈** 4.09714 seconds
- **Operations Per Second (OPS):** 2,441 inserts/second

3. With `innodb_flush_log_at_trx_commit = 3`:

- **Total time for 10,000 inserts ≈** 3.42821 seconds
- **Operations Per Second (OPS):** 2,918 inserts/second
