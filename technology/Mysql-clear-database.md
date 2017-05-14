# Mysql 清空数据库

最近项目中需要一个清空数据库数据的操作。在网上一直没能找到让我很满意的方法，所以自己动手写了一个。

- 方法特点： 简单易用，只需要执行一条 `SQL`，而不需要其他命令

该方法基于 `TRUNCATE TABLE` 命令。
1. 定义例程： `PROCEDURE clear_database`
1. 执行清空命令： `call clear_database();`

```sql
-- 销毁之前的定义
DROP PROCEDURE IF EXISTS clear_database;

-- 清空数据库的存储例程， 客户端直接使用 `call clear_database();` 即可。
DELIMITER $
CREATE PROCEDURE clear_database()
  BEGIN
    -- 用来临时存放表名
    DECLARE tname VARCHAR(50);

    -- 结果集遍历结束标志
    DECLARE done INT DEFAULT FALSE;

    -- 定义游标
    DECLARE cur CURSOR FOR
      -- 查询当前数据库中的所有表的表名
      -- 可以对该 sql 进行修改，去掉需要保留的表。
      SELECT TABLE_NAME
      FROM INFORMATION_SCHEMA.TABLES
      WHERE TABLE_SCHEMA = (SELECT database());

    -- 当游标到达结果集尾的时候设置 done 为 true
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;


    SET @tt = 'TRUNCATE TABLE ';

    -- 关闭外键检测
    SET FOREIGN_KEY_CHECKS = 0;

    OPEN cur;
    read_loop: LOOP
      -- 读取一条记录到 tname
      FETCH cur INTO tname;

      -- 拼接实际执行的语句
      SET @sqlexe = concat(@tt, tname);

      -- 如果到达了结果集的尾部，跳出循环
      IF done
      THEN
        LEAVE read_loop;
      END IF;

      -- 定义 PREPARE
      PREPARE TRUNCATE_TABLE FROM @sqlexe;

      -- 执行清空语句
      EXECUTE TRUNCATE_TABLE;
    END LOOP;
    CLOSE cur;

    -- 销毁 PREPARE
    DEALLOCATE PREPARE TRUNCATE_TABLE;

    -- 重新开启外键检测
    SET FOREIGN_KEY_CHECKS = 1;
  END $

DELIMITER ;
```

- 该定义无需修改即可用于其他数据库。
- 若有特殊需要可以定制其中的 `SELECT` 语句，只选出想要清空的表即可。


## 参考

- [MySQL 5.7 Reference Manual](https://dev.mysql.com/doc/refman/5.7/en/)
