# mysql批量修改utf8mb3为utf8mb4字符集

## 设置数据库字段集

```sql
ALTER DATABASE <DATABASE_NAME> CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
```

## 查询utf8mb3 字符的表，并生成修改sql语句

```sql
SELECT CONCAT('ALTER TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ' CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;') AS sql_statements
FROM information_schema.TABLES
WHERE TABLE_SCHEMA = '<database_name>'
AND TABLE_TYPE = 'BASE TABLE';

SELECT CONCAT('ALTER TABLE ', TABLE_SCHEMA, '.', TABLE_NAME, ' MODIFY ', COLUMN_NAME, ' ', DATA_TYPE, IF(ISNULL(CHARACTER_MAXIMUM_LENGTH), '', CONCAT('(', CHARACTER_MAXIMUM_LENGTH, ')')), ' CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci', IF(IS_NULLABLE = 'NO', ' NOT NULL', ''), ';') AS sql_statements
FROM information_schema.COLUMNS
WHERE TABLE_SCHEMA = '<database_name>'
AND CHARACTER_SET_NAME = 'utf8mb3';
```

查询结果类似如下：

```sql
ALTER TABLE <database_name>.active CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
ALTER TABLE <database_name>.article CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
...
```

查询结果copy 到sql 控制台，执行结果类似如下结果：

| Name             | Value          |
|------------------|----------------|
| Queries          | 20             |
| Updated Rows     | 822            |
| Execute time (ms)| 2881           |
| Fetch time (ms)  | 0              |
| Total time (ms)  | 2881           |
| Start time       | 2023-07-17 14:58:42.897 |
| Finish time      | 2023-07-17 14:58:45.788 |

