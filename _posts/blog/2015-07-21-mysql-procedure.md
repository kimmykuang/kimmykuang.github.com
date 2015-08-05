---
layout: post
title: 	删除关联表中重复数据的MySQL存储过程
category: blog
tag: mysql, mysql procedure, 存储过程
description: 一般的web应用数据库设计中多少都会有关联表的存在，用于记录表之间的关联关系等，良好地实践过程中不应该在关联表中存在大量重复数据，如果需要清除重复记录手动肯定不是一个好的选择，我自己写了一个mysql的存储过程来生成出删除所有重复数据的sql语句。
---

存储过程sql如下：

	DROP PROCEDURE IF EXISTS getDupIds;
	CREATE PROCEDURE getDupIds(IN tableName VARCHAR(100), IN pk VARCHAR(100), IN groupFields VARCHAR(100))
	BEGIN
     	DECLARE groupIds TEXT;
     	DECLARE _done INT DEFAULT 0;
     	DECLARE _cur CURSOR FOR
        SELECT * FROM check_dupid_view;
     	DECLARE CONTINUE HANDLER FOR SQLSTATE '02000' SET _done = 1;

     	CREATE TEMPORARY TABLE IF NOT EXISTS tmpTable(`ids` TEXT NOT NULL) ENGINE=MYISAM CHARSET=utf8;

     	SET @v_sql = CONCAT("CREATE VIEW check_dupid_view AS SELECT GROUP_CONCAT(CAST(", pk, " AS CHAR)) 
     	AS `group_id` FROM ", tableName, " GROUP BY ", groupFields, " HAVING COUNT(*) > 1");
     	PREPARE stmt FROM @v_sql;
     	EXECUTE stmt;
     	DEALLOCATE PREPARE stmt;

     	OPEN _cur;
          FETCH _cur INTO groupIds;
          WHILE(_done <> 1) DO
               SET groupIds = CONCAT("DELETE FROM `", tableName, "` WHERE `", pk, "` IN (", SUBSTRING(groupIds, INSTR(groupIds,',')+1), ");");
               INSERT INTO tmpTable(`ids`) VALUES (groupIds);
               FETCH _cur INTO groupIds;
          END WHILE;
     	CLOSE _cur;
     	SELECT * FROM tmpTable;
     	DROP TEMPORARY TABLE tmpTable;
     	DROP VIEW check_dupid_view;

	END;
	
调用方式：假设有一张分类关联表叫做```article_topic```，记录了每篇文章与分类的关系，表结构如下：

	id int(10) primary key auto_increment not null,
	article_id int(10) not null,
	topic_id int(10) not null,
	
那么调用存储过程删除重复记录：call getDupIds('article_topic', 'id', 'article_id, topic_id'); 第一个参数是表名，第二个参数是表的主键名，第三个参数是重复的过滤条件，就是放在group by条件语句后的字段。

调用后将生成类似```DELETE FROM `article_topic` WHERE `id` IN (1,2,3);```的sql语句，直接执行sql语句即可删除重复记录。

一般在处理删除重复纪录的时候，会用到```DELETE FROM `table_name` GROUP BY xx,xx,xx HAVING COUNT(*) > 1;```的sql，但是我遇到过同一条数据重复了n次，那么这条sql语句也要执行n－1次才能把所有重复项删除干净，效率不高。所以干脆先根据提供的三个参数把所有重复项找出来，接着用一个游标遍历结果集，用mysql自带的SUBSTRING函数，把所有重复项的id只保留第一个，其余的id都筛选出来，放到一个临时表里输出。

还有一点需要注意的是，mysql存储过程中如果有拼接sql语句的，需要对这个sql语句进行预处理，如line 12~16所示。