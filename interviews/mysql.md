### Mysql 面试技术知识点

#### 1、MyISAM和InnoDB对比

| 对比项   | MyISAM                                                 | InnoDB                                                       |
| -------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 主外键   | 不支持                                                 | 支持                                                         |
| 事务     | 不支持                                                 | 支持                                                         |
| 行表锁   | 表锁，即使操作一条数据也会锁住整个表，不适合高并发场景 | 行锁，操作时之所某一行，不多其它行有影响，适合高并发的场景   |
| 缓存     | 只缓存索引，部缓存真实的数据                           | 不仅缓存索引，还缓存真实的数据，对内存要求高，内存大小对性能有绝对性的影响 |
| 表空间   | 小                                                     | 大                                                           |
| 关注点   | 性能                                                   | 事务                                                         |
| 默认安装 | 是                                                     | 是                                                           |

#### 2、7种join方式演示

```sql
CREATE DATABASE db_test;
USE db_test;
CREATE TABLE `tbl_dept` (
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `deptName` VARCHAR(30) DEFAULT NULL,
  `locAdd` VARCHAR(40) DEFAULT NULL COMMENT '楼层',
  PRIMARY KEY (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;

CREATE TABLE `tbl_emp`(
  `id` INT(11) NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(20) DEFAULT NULL,
  `deptId` INT(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_dept_id`(`deptId`)
  #CONSTRAINT `fk_dept_id` FOREIGN KEY (`deptId`) REFERENCES `tbl_dept` (`id`)
) ENGINE=INNODB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;


INSERT INTO tbl_dept(deptName,locAdd) VALUES ('RD',11), ('HR',12), ('MK',13), ('MIS',14), ('FD',15);
INSERT INTO tbl_emp(name,deptId) VALUES('z3',1), ('z4',1), ('z5',1);
INSERT INTO tbl_emp(name,deptId) VALUES('w5',2), ('w6',2);
INSERT INTO tbl_emp(name,deptId) VALUES('s7',3);
INSERT INTO tbl_emp(name,deptId) VALUES('s8',4);
INSERT INTO tbl_emp(name,deptId) VALUES('s9',51);  # 故意错的51
```

* 1、inner join

```sql
select * from tbl_emp emp inner join tbl_dept dept on emp.deptId=dept.id;
```

* 2、left join

```sql
select * from tbl_emp emp left join tbl_dept dept on emp.deptId=dept.id;
```

* 3、left join excluding inner join

```sql
select * from tbl_emp emp left join tbl_dept dept on emp.deptId=dept.id where dept.id is NULL
```

* 4、right join

```sql
select * from tbl_emp emp right join tbl_dept dept on emp.deptId=dept.id
```

* 5、right join excluding inner join

```sql
select * from tbl_emp emp right join tbl_dept dept on emp.deptId=dept.id where emp.id is NULL;
```

* 6、full outer join(oracle 支持full outer join语法，但是mysql不支持，做如下变形)

```sql
select * from tbl_emp emp left join tbl_dept dept on emp.deptId=dept.id union select * from tbl_emp emp right join tbl_dept dept on emp.deptId=dept.id;
```

* 7、full outer join excluding inner join

```sql
select * from tbl_emp emp left join tbl_dept dept on emp.deptId=dept.id where dept.id is null union select * from tbl_emp emp right join tbl_dept dept on emp.deptId=dept.id where emp.id is null;
```

