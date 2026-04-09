---
title: MySQL 五十道查询题目
slug: fifty-mysql-query-questions
tags: [MySQL]
date: 2021-07-01T21:59:31+08:00
---

MySQL 经典练习题及答案，常用 SQL 语句练习 50 题<!--more-->

## 表名和字段

- 1. 学生表  
  `Student(s_id, s_name, s_birth, s_sex)` — 学生编号，学生姓名，出生年月，学生性别
- 2. 课程表  
  `Course(c_id, c_name, t_id)` — 课程编号，课程名称，教师编号
- 3. 教师表  
  `Teacher(t_id, t_name)` — 教师编号，教师姓名
- 4. 成绩表  
  `Score(s_id, c_id, s_score)` — 学生编号，课程编号，分数

## 测试数据

```sql
# --1.学生表 
# Student(s_id,s_name,s_birth,s_sex) —学生编号,学生姓名, 出生年月,学生性别
CREATE TABLE `Student` (
    `s_id` VARCHAR(20),
    s_name VARCHAR(20) NOT NULL DEFAULT '',
    s_brith VARCHAR(20) NOT NULL DEFAULT '',
    s_sex VARCHAR(10) NOT NULL DEFAULT '',
    PRIMARY KEY(s_id)
);

# --2.课程表 
# Course(c_id,c_name,t_id) — 课程编号, 课程名称, 教师编号 
CREATE TABLE Course(
    c_id VARCHAR(20),
    c_name VARCHAR(20) NOT NULL DEFAULT '',
    t_id VARCHAR(20) NOT NULL,
    PRIMARY KEY(c_id)
);

/*
--3.教师表 
Teacher(t_id,t_name) —教师编号,教师姓名 
*/
CREATE TABLE Teacher(
    t_id VARCHAR(20),
    t_name VARCHAR(20) NOT NULL DEFAULT '',
    PRIMARY KEY(t_id)
);

/*
--4.成绩表 
Score(s_id,c_id,s_score) —学生编号,课程编号,分数
*/
CREATE TABLE Score(
    s_id VARCHAR(20),
    c_id VARCHAR(20) NOT NULL DEFAULT '',
    s_score INT(3),
    PRIMARY KEY(`s_id`, `c_id`)
);
```

## 插入数据

```sql
# --插入学生表测试数据
# ('01' , '赵雷' , '1990-01-01' , '男')
INSERT INTO Student VALUES ('01', '赵雷', '1990-01-01', '男');
INSERT INTO Student VALUES ('02', '钱电', '1990-12-21', '男');
INSERT INTO Student VALUES ('03', '孙风', '1990-05-20', '男');
INSERT INTO Student VALUES ('04', '李云', '1990-08-06', '男');
INSERT INTO Student VALUES ('05', '周梅', '1991-12-01', '女');
INSERT INTO Student VALUES ('06', '吴兰', '1992-03-01', '女');
INSERT INTO Student VALUES ('07', '郑竹', '1989-07-01', '女');
INSERT INTO Student VALUES ('08', '王菊', '1990-01-20', '女');

# --课程表测试数据
INSERT INTO Course VALUES ('01', '语文', '02');
INSERT INTO Course VALUES ('02', '数学', '01');
INSERT INTO Course VALUES ('03', '英语', '03');

# --教师表测试数据
INSERT INTO Teacher VALUES ('01', '张三');
INSERT INTO Teacher VALUES ('02', '李四');
INSERT INTO Teacher VALUES ('03', '王五');

# --成绩表测试数据
INSERT INTO Score VALUES ('01', '01', 80);
INSERT INTO Score VALUES ('01', '02', 90);
INSERT INTO Score VALUES ('01', '03', 99);
INSERT INTO Score VALUES ('02', '01', 70);
INSERT INTO Score VALUES ('02', '02', 60);
INSERT INTO Score VALUES ('02', '03', 80);
INSERT INTO Score VALUES ('03', '01', 80);
INSERT INTO Score VALUES ('03', '02', 80);
INSERT INTO Score VALUES ('03', '03', 80);
INSERT INTO Score VALUES ('04', '01', 50);
INSERT INTO Score VALUES ('04', '02', 30);
INSERT INTO Score VALUES ('04', '03', 20);
INSERT INTO Score VALUES ('05', '01', 76);
INSERT INTO Score VALUES ('05', '02', 87);
INSERT INTO Score VALUES ('06', '01', 31);
INSERT INTO Score VALUES ('06', '03', 34);
INSERT INTO Score VALUES ('07', '02', 89);
INSERT INTO Score VALUES ('07', '03', 98);
```

## 练习题和答案

```sql
-- 1. 查询"01"课程比"02"课程成绩高的学生的信息及课程分数
SELECT a.*, b.s_score AS score01, c.s_score AS score02 
FROM student a 
JOIN score b ON a.s_id = b.s_id AND b.c_id = '01'
LEFT JOIN score c ON a.s_id = c.s_id AND c.c_id = '02' OR c.c_id IS NULL 
WHERE b.s_score > c.s_score;

-- 2. 查询"01"课程比"02"课程成绩低的学生的信息及课程分数
SELECT a.*, b.s_score AS 01_score, c.s_score AS 02_score 
FROM student a 
LEFT JOIN score b ON a.s_id = b.s_id AND b.c_id = '01' OR b.c_id IS NULL 
JOIN score c ON a.s_id = c.s_id AND c.c_id = '02' 
WHERE b.s_score < c.s_score;

-- 3. 查询平均成绩大于等于60分的同学的学生编号和学生姓名和平均成绩
SELECT b.s_id, b.s_name, ROUND(AVG(a.s_score), 2) AS avg_score 
FROM student b 
JOIN score a ON b.s_id = a.s_id
GROUP BY b.s_id, b.s_name 
HAVING ROUND(AVG(a.s_score), 2) >= 60;

-- 4. 查询平均成绩小于60分的同学的学生编号和学生姓名和平均成绩
-- (包括有成绩的和无成绩的)
SELECT b.s_id, b.s_name, ROUND(AVG(a.s_score), 2) AS avg_score 
FROM student b 
LEFT JOIN score a ON b.s_id = a.s_id
GROUP BY b.s_id, b.s_name 
HAVING ROUND(AVG(a.s_score), 2) < 60
UNION
SELECT a.s_id, a.s_name, 0 AS avg_score 
FROM student a 
WHERE a.s_id NOT IN (
    SELECT DISTINCT s_id FROM score
);

-- 5. 查询所有同学的学生编号、学生姓名、选课总数、所有课程的总成绩
SELECT a.s_id, a.s_name, COUNT(b.c_id) AS sum_course, SUM(b.s_score) AS sum_score 
FROM student a 
LEFT JOIN score b ON a.s_id = b.s_id
GROUP BY a.s_id, a.s_name;

-- 6. 查询"李"姓老师的数量
SELECT COUNT(t_id) FROM teacher WHERE t_name LIKE '李%';

-- 7. 查询学过"张三"老师授课的同学的信息
SELECT a.* 
FROM student a 
JOIN score b ON a.s_id = b.s_id 
WHERE b.c_id IN (
    SELECT c_id FROM course 
    WHERE t_id = (
        SELECT t_id FROM teacher WHERE t_name = '张三'
    )
);

-- 8. 查询没学过"张三"老师授课的同学的信息
SELECT * 
FROM student c 
WHERE c.s_id NOT IN (
    SELECT a.s_id 
    FROM student a 
    JOIN score b ON a.s_id = b.s_id 
    WHERE b.c_id IN (
        SELECT c_id FROM course 
        WHERE t_id = (
            SELECT t_id FROM teacher WHERE t_name = '张三'
        )
    )
);

-- 9. 查询学过编号为"01"并且也学过编号为"02"的课程的同学的信息
SELECT a.* 
FROM student a, score b, score c 
WHERE a.s_id = b.s_id AND a.s_id = c.s_id AND b.c_id = '01' AND c.c_id = '02';

-- 10. 查询学过编号为"01"但是没有学过编号为"02"的课程的同学的信息
SELECT a.* 
FROM student a 
WHERE a.s_id IN (SELECT s_id FROM score WHERE c_id = '01') 
  AND a.s_id NOT IN (SELECT s_id FROM score WHERE c_id = '02');

-- 11. 查询没有学全所有课程的同学的信息
SELECT s.* 
FROM student s 
WHERE s.s_id IN (
    SELECT s_id FROM score 
    WHERE s_id NOT IN (
        SELECT a.s_id 
        FROM score a 
        JOIN score b ON a.s_id = b.s_id AND b.c_id = '02'
        JOIN score c ON a.s_id = c.s_id AND c.c_id = '03'
        WHERE a.c_id = '01'
    )
);

-- 12. 查询至少有一门课与学号为"01"的同学所学相同的同学的信息
SELECT * 
FROM student 
WHERE s_id IN (
    SELECT DISTINCT a.s_id 
    FROM score a 
    WHERE a.c_id IN (SELECT c_id FROM score WHERE s_id = '01')
);

-- 13. 查询和"01"号的同学学习的课程完全相同的其他同学的信息
SELECT a.* 
FROM student a 
WHERE a.s_id IN (
    SELECT DISTINCT s_id 
    FROM score 
    WHERE s_id != '01' AND c_id IN (SELECT c_id FROM score WHERE s_id = '01')
    GROUP BY s_id 
    HAVING COUNT(1) = (SELECT COUNT(1) FROM score WHERE s_id = '01')
);

-- 14. 查询没学过"张三"老师讲授的任一门课程的学生姓名
SELECT a.s_name 
FROM student a 
WHERE a.s_id NOT IN (
    SELECT s_id 
    FROM score 
    WHERE c_id = (
        SELECT c_id 
        FROM course 
        WHERE t_id = (
            SELECT t_id FROM teacher WHERE t_name = '张三'
        )
    )
    GROUP BY s_id
);

-- 15. 查询两门及其以上不及格课程的同学的学号，姓名及其平均成绩
SELECT a.s_id, a.s_name, ROUND(AVG(b.s_score)) 
FROM student a 
LEFT JOIN score b ON a.s_id = b.s_id
WHERE a.s_id IN (
    SELECT s_id FROM score WHERE s_score < 60 
    GROUP BY s_id HAVING COUNT(1) >= 2
)
GROUP BY a.s_id, a.s_name;

-- 16. 检索"01"课程分数小于60，按分数降序排列的学生信息
SELECT a.*, b.c_id, b.s_score 
FROM student a, score b 
WHERE a.s_id = b.s_id AND b.c_id = '01' AND b.s_score < 60 
ORDER BY b.s_score DESC;

-- 17. 按平均成绩从高到低显示所有学生的所有课程的成绩以及平均成绩
SELECT a.s_id,
    (SELECT s_score FROM score WHERE s_id = a.s_id AND c_id = '01') AS 语文,
    (SELECT s_score FROM score WHERE s_id = a.s_id AND c_id = '02') AS 数学,
    (SELECT s_score FROM score WHERE s_id = a.s_id AND c_id = '03') AS 英语,
    ROUND(AVG(s_score), 2) AS 平均分 
FROM score a  
GROUP BY a.s_id 
ORDER BY 平均分 DESC;

-- 18. 查询各科成绩最高分、最低分和平均分：
-- 课程ID，课程name，最高分，最低分，平均分，及格率，中等率，优良率，优秀率
-- 及格为>=60，中等为：70-80，优良为：80-90，优秀为：>=90
SELECT a.c_id, b.c_name, MAX(s_score), MIN(s_score), ROUND(AVG(s_score), 2),
    ROUND(100 * (SUM(CASE WHEN a.s_score >= 60 THEN 1 ELSE 0 END) / SUM(CASE WHEN a.s_score THEN 1 ELSE 0 END)), 2) AS 及格率,
    ROUND(100 * (SUM(CASE WHEN a.s_score >= 70 AND a.s_score <= 80 THEN 1 ELSE 0 END) / SUM(CASE WHEN a.s_score THEN 1 ELSE 0 END)), 2) AS 中等率,
    ROUND(100 * (SUM(CASE WHEN a.s_score >= 80 AND a.s_score <= 90 THEN 1 ELSE 0 END) / SUM(CASE WHEN a.s_score THEN 1 ELSE 0 END)), 2) AS 优良率,
    ROUND(100 * (SUM(CASE WHEN a.s_score >= 90 THEN 1 ELSE 0 END) / SUM(CASE WHEN a.s_score THEN 1 ELSE 0 END)), 2) AS 优秀率
FROM score a 
LEFT JOIN course b ON a.c_id = b.c_id 
GROUP BY a.c_id, b.c_name;

-- 19. 按各科成绩进行排序，并显示排名（实现不完全）
-- MySQL 没有 rank 函数
SELECT a.s_id, a.c_id,
    @i := @i + 1 AS i保留排名,
    @k := (CASE WHEN @score = a.s_score THEN @k ELSE @i END) AS rank不保留排名,
    @score := a.s_score AS score
FROM (
    SELECT s_id, c_id, s_score 
    FROM score 
    WHERE c_id = '01' 
    GROUP BY s_id, c_id, s_score 
    ORDER BY s_score DESC
) a, (SELECT @k := 0, @i := 0, @score := 0) s
UNION
SELECT a.s_id, a.c_id,
    @i := @i + 1 AS i,
    @k := (CASE WHEN @score = a.s_score THEN @k ELSE @i END) AS rank,
    @score := a.s_score AS score
FROM (
    SELECT s_id, c_id, s_score 
    FROM score 
    WHERE c_id = '02' 
    GROUP BY s_id, c_id, s_score 
    ORDER BY s_score DESC
) a, (SELECT @k := 0, @i := 0, @score := 0) s
UNION
SELECT a.s_id, a.c_id,
    @i := @i + 1 AS i,
    @k := (CASE WHEN @score = a.s_score THEN @k ELSE @i END) AS rank,
    @score := a.s_score AS score
FROM (
    SELECT s_id, c_id, s_score 
    FROM score 
    WHERE c_id = '03' 
    GROUP BY s_id, c_id, s_score 
    ORDER BY s_score DESC
) a, (SELECT @k := 0, @i := 0, @score := 0) s;

-- 20. 查询学生的总成绩并进行排名
SELECT a.s_id,
    @i := @i + 1 AS i,
    @k := (CASE WHEN @score = a.sum_score THEN @k ELSE @i END) AS rank,
    @score := a.sum_score AS score
FROM (
    SELECT s_id, SUM(s_score) AS sum_score 
    FROM score 
    GROUP BY s_id 
    ORDER BY sum_score DESC
) a, (SELECT @k := 0, @i := 0, @score := 0) s;

-- 21. 查询不同老师所教不同课程平均分从高到低显示
SELECT a.t_id, c.t_name, a.c_id, ROUND(AVG(s_score), 2) AS avg_score 
FROM course a
LEFT JOIN score b ON a.c_id = b.c_id 
LEFT JOIN teacher c ON a.t_id = c.t_id
GROUP BY a.c_id, a.t_id, c.t_name 
ORDER BY avg_score DESC;

-- 22. 查询所有课程的成绩第2名到第3名的学生信息及该课程成绩
SELECT d.*, c.排名, c.s_score, c.c_id 
FROM (
    SELECT a.s_id, a.s_score, a.c_id, @i := @i + 1 AS 排名 
    FROM score a, (SELECT @i := 0) s 
    WHERE a.c_id = '01'    
) c
LEFT JOIN student d ON c.s_id = d.s_id
WHERE 排名 BETWEEN 2 AND 3
UNION
SELECT d.*, c.排名, c.s_score, c.c_id 
FROM (
    SELECT a.s_id, a.s_score, a.c_id, @j := @j + 1 AS 排名 
    FROM score a, (SELECT @j := 0) s 
    WHERE a.c_id = '02'    
) c
LEFT JOIN student d ON c.s_id = d.s_id
WHERE 排名 BETWEEN 2 AND 3
UNION
SELECT d.*, c.排名, c.s_score, c.c_id 
FROM (
    SELECT a.s_id, a.s_score, a.c_id, @k := @k + 1 AS 排名 
    FROM score a, (SELECT @k := 0) s 
    WHERE a.c_id = '03'    
) c
LEFT JOIN student d ON c.s_id = d.s_id
WHERE 排名 BETWEEN 2 AND 3;

-- 23. 统计各科成绩各分数段人数：课程编号，课程名称，[100-85]，[85-70]，[70-60]，[0-60]及所占百分比
SELECT DISTINCT f.c_name, a.c_id, 
    b.`85-100`, b.百分比, 
    c.`70-85`, c.百分比, 
    d.`60-70`, d.百分比, 
    e.`0-60`, e.百分比 
FROM score a
LEFT JOIN (
    SELECT c_id, 
        SUM(CASE WHEN s_score > 85 AND s_score <= 100 THEN 1 ELSE 0 END) AS `85-100`,
        ROUND(100 * (SUM(CASE WHEN s_score > 85 AND s_score <= 100 THEN 1 ELSE 0 END) / COUNT(*)), 2) AS 百分比
    FROM score 
    GROUP BY c_id
) b ON a.c_id = b.c_id
LEFT JOIN (
    SELECT c_id, 
        SUM(CASE WHEN s_score > 70 AND s_score <= 85 THEN 1 ELSE 0 END) AS `70-85`,
        ROUND(100 * (SUM(CASE WHEN s_score > 70 AND s_score <= 85 THEN 1 ELSE 0 END) / COUNT(*)), 2) AS 百分比
    FROM score 
    GROUP BY c_id
) c ON a.c_id = c.c_id
LEFT JOIN (
    SELECT c_id, 
        SUM(CASE WHEN s_score > 60 AND s_score <= 70 THEN 1 ELSE 0 END) AS `60-70`,
        ROUND(100 * (SUM(CASE WHEN s_score > 60 AND s_score <= 70 THEN 1 ELSE 0 END) / COUNT(*)), 2) AS 百分比
    FROM score 
    GROUP BY c_id
) d ON a.c_id = d.c_id
LEFT JOIN (
    SELECT c_id, 
        SUM(CASE WHEN s_score >= 0 AND s_score <= 60 THEN 1 ELSE 0 END) AS `0-60`,
        ROUND(100 * (SUM(CASE WHEN s_score >= 0 AND s_score <= 60 THEN 1 ELSE 0 END) / COUNT(*)), 2) AS 百分比
    FROM score 
    GROUP BY c_id
) e ON a.c_id = e.c_id
LEFT JOIN course f ON a.c_id = f.c_id;

-- 24. 查询学生平均成绩及其名次
SELECT a.s_id,
    @i := @i + 1 AS '不保留空缺排名',
    @k := (CASE WHEN @avg_score = a.avg_s THEN @k ELSE @i END) AS '保留空缺排名',
    @avg_score := avg_s AS '平均分'
FROM (
    SELECT s_id, ROUND(AVG(s_score), 2) AS avg_s 
    FROM score 
    GROUP BY s_id
) a, (SELECT @avg_score := 0, @i := 0, @k := 0) b;

-- 25. 查询各科成绩前三名的记录
SELECT a.s_id, a.c_id, a.s_score 
FROM score a 
LEFT JOIN score b ON a.c_id = b.c_id AND a.s_score < b.s_score
GROUP BY a.s_id, a.c_id, a.s_score 
HAVING COUNT(b.s_id) < 3
ORDER BY a.c_id, a.s_score DESC;

-- 26. 查询每门课程被选修的学生数
SELECT c_id, COUNT(s_id) 
FROM score a 
GROUP BY c_id;

-- 27. 查询出只有两门课程的全部学生的学号和姓名
SELECT s_id, s_name 
FROM student 
WHERE s_id IN (
    SELECT s_id FROM score 
    GROUP BY s_id HAVING COUNT(c_id) = 2
);

-- 28. 查询男生、女生人数
SELECT s_sex, COUNT(s_sex) AS 人数 
FROM student 
GROUP BY s_sex;

-- 29. 查询名字中含有"风"字的学生信息
SELECT * 
FROM student 
WHERE s_name LIKE '%风%';

-- 30. 查询同名同性学生名单，并统计同名人数
SELECT a.s_name, a.s_sex, COUNT(*) 
FROM student a 
JOIN student b ON a.s_id != b.s_id AND a.s_name = b.s_name AND a.s_sex = b.s_sex
GROUP BY a.s_name, a.s_sex;

-- 31. 查询1990年出生的学生名单
SELECT s_name 
FROM student 
WHERE s_birth LIKE '1990%';

-- 32. 查询每门课程的平均成绩，结果按平均成绩降序排列，平均成绩相同时，按课程编号升序排列
SELECT c_id, ROUND(AVG(s_score), 2) AS avg_score 
FROM score 
GROUP BY c_id 
ORDER BY avg_score DESC, c_id ASC;

-- 33. 查询平均成绩大于等于85的所有学生的学号、姓名和平均成绩
SELECT a.s_id, b.s_name, ROUND(AVG(a.s_score), 2) AS avg_score 
FROM score a
LEFT JOIN student b ON a.s_id = b.s_id 
GROUP BY a.s_id 
HAVING avg_score >= 85;

-- 34. 查询课程名称为"数学"，且分数低于60的学生姓名和分数
SELECT a.s_name, b.s_score 
FROM score b 
LEFT JOIN student a ON a.s_id = b.s_id 
WHERE b.c_id = (
    SELECT c_id FROM course WHERE c_name = '数学'
) AND b.s_score < 60;

-- 35. 查询所有学生的课程及分数情况
SELECT a.s_id, a.s_name,
    SUM(CASE c.c_name WHEN '语文' THEN b.s_score ELSE 0 END) AS '语文',
    SUM(CASE c.c_name WHEN '数学' THEN b.s_score ELSE 0 END) AS '数学',
    SUM(CASE c.c_name WHEN '英语' THEN b.s_score ELSE 0 END) AS '英语',
    SUM(b.s_score) AS '总分'
FROM student a 
LEFT JOIN score b ON a.s_id = b.s_id 
LEFT JOIN course c ON b.c_id = c.c_id 
GROUP BY a.s_id, a.s_name;

-- 36. 查询任何一门课程成绩在70分以上的姓名、课程名称和分数
SELECT a.s_name, b.c_name, c.s_score 
FROM course b 
LEFT JOIN score c ON b.c_id = c.c_id
LEFT JOIN student a ON a.s_id = c.s_id 
WHERE c.s_score >= 70;

-- 37. 查询不及格的课程
SELECT a.s_id, a.c_id, b.c_name, a.s_score 
FROM score a 
LEFT JOIN course b ON a.c_id = b.c_id
WHERE a.s_score < 60;

-- 38. 查询课程编号为01且课程成绩在80分以上的学生的学号和姓名
SELECT a.s_id, b.s_name 
FROM score a 
LEFT JOIN student b ON a.s_id = b.s_id
WHERE a.c_id = '01' AND a.s_score > 80;

-- 39. 求每门课程的学生人数
SELECT c_id, COUNT(*) AS student_count 
FROM score 
GROUP BY c_id;

-- 40. 查询选修"张三"老师所授课程的学生中，成绩最高的学生信息及其成绩
-- 查询老师ID
SELECT c_id 
FROM course c, teacher d 
WHERE c.t_id = d.t_id AND d.t_name = '张三';
-- 查询最高分（可能有相同分数）
SELECT MAX(s_score) 
FROM score 
WHERE c_id = '02';
-- 查询信息
SELECT a.*, b.s_score, b.c_id, c.c_name 
FROM student a
LEFT JOIN score b ON a.s_id = b.s_id
LEFT JOIN course c ON b.c_id = c.c_id
WHERE b.c_id = (
    SELECT c_id FROM course c, teacher d WHERE c.t_id = d.t_id AND d.t_name = '张三'
)
AND b.s_score IN (
    SELECT MAX(s_score) FROM score WHERE c_id = '02'
);

-- 41. 查询不同课程成绩相同的学生的学生编号、课程编号、学生成绩
SELECT DISTINCT b.s_id, b.c_id, b.s_score 
FROM score a, score b 
WHERE a.c_id != b.c_id AND a.s_score = b.s_score;

-- 42. 查询每门功成绩最好的前两名
-- 牛逼的写法
SELECT a.s_id, a.c_id, a.s_score 
FROM score a
WHERE (
    SELECT COUNT(1) 
    FROM score b 
    WHERE b.c_id = a.c_id AND b.s_score >= a.s_score
) <= 2 
ORDER BY a.c_id;

-- 43. 统计每门课程的学生选修人数（超过5人的课程才统计）
-- 要求输出课程号和选修人数，查询结果按人数降序排列，若人数相同，按课程号升序排列
SELECT c_id, COUNT(*) AS total 
FROM score 
GROUP BY c_id 
HAVING total > 5 
ORDER BY total DESC, c_id ASC;

-- 44. 检索至少选修两门课程的学生学号
SELECT s_id, COUNT(*) AS sel 
FROM score 
GROUP BY s_id 
HAVING sel >= 2;

-- 45. 查询选修了全部课程的学生信息
SELECT * 
FROM student 
WHERE s_id IN (
    SELECT s_id 
    FROM score 
    GROUP BY s_id 
    HAVING COUNT(*) = (SELECT COUNT(*) FROM course)
);

-- 46. 查询各学生的年龄
-- 按照出生日期来算，当前月日 < 出生年月的月日则，年龄减一
SELECT s_id, s_name, s_birth, s_sex,
    (DATE_FORMAT(NOW(), '%Y') - DATE_FORMAT(s_birth, '%Y') - 
    (CASE WHEN DATE_FORMAT(NOW(), '%m%d') > DATE_FORMAT(s_birth, '%m%d') THEN 0 ELSE 1 END)) AS age
FROM student;

-- 47. 查询本周过生日的学生
SELECT * 
FROM student 
WHERE WEEK(DATE_FORMAT(NOW(), '%Y%m%d')) = WEEK(s_birth)
   OR YEARWEEK(s_birth) = YEARWEEK(DATE_FORMAT(NOW(), '%Y%m%d'));

-- 48. 查询下周过生日的学生
SELECT * 
FROM student 
WHERE WEEK(DATE_FORMAT(NOW(), '%Y%m%d')) + 1 = WEEK(s_birth);

-- 49. 查询本月过生日的学生
SELECT * 
FROM student 
WHERE MONTH(NOW()) = MONTH(s_birth);

-- 50. 查询下月过生日的学生
SELECT * 
FROM student 
WHERE MONTH(NOW()) + 1 = MONTH(s_birth)
   OR (MONTH(NOW()) = 12 AND MONTH(s_birth) = 1);
```