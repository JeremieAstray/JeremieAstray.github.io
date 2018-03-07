## SQL 语法学习笔记  
[<博客主页](https://jeremieastray.github.io)  
  

```
dml语句：（数据操作语句）

查询语句
select

1)
use mydata; 
desc emp;
desc dept;
select * from salgrade;
select * from dept;
2)
select now();
select sysdate();

select ename, sal*12 anuual_sal from emp;     改显示的列名
select ename, sal*12 "anuual sal" from emp; 
select ename, comm from emp;                   展示空值和0
select ename, sal*12 + comm from emp;          汇总（错误例子），任何含空值的表达式都不能计算
select ename||sal from emp;                    //oracle
select CONCAT(ename,sal) from emp;             //mysql
select CONCAT(ename,'sal') from emp;
select CONCAT(ename,'s''al') from emp;
3)
select distinct deptno from emp;               //不重复的显示deptno字段的内容
select distinct deptno , job from emp;         //不重复的显示字段对的内容
4）
select * from emp where deptno = 10;           //where 是过滤条件
select * from emp where ename = 'test';
select * from emp where ename != 'test';
select * from emp where sal >= 2000;
select * from emp where sal <> 2000;
select * from emp where ename > 'ee';
select * from emp where sal between 1000 and 2000;
select * from emp where sal >= 1000 and sal <= 2000; //等价于上条
select ename, sal, comm from emp where comm is null; //选空值 
select ename, sal, comm from emp where comm is not null;
select ename, sal, comm from emp where sal in (1000, 2000);//找到符合规则的
select ename, sal, comm from emp where ename in ('ee','qq');
5)
*日期处理
select ename, sal, hiredate from emp where hiredate > 1981-07-29; //日期需要特殊的字符串，select sysdate();可以查看
select ename, sal, hiredate from emp where deptno = 10 and sal > 1000;//两个过滤条件
select ename, sal, hiredate from emp where deptno = 10 or sal > 1000;
select ename, sal, hiredate from emp where sal not in (1000)；
6)
select ename from emp where ename like '%e%';	//"%"（通配符）表示0个或多个，"_"代表一个字母 （这句话表示找字符串含有e的值）
select ename from emp where ename like '_e%';	//（这句话表示找字符串第二位为e的值）
select ename from emp where ename like '%\%%';  //\为转义字符表示字符串中有一个%
select ename from emp where ename like '%$%%' escape '$'; //表示$符号位转义字符（也就是说转义字符可以自定义,默认"\"）
7)
排序
select * from dept order by deptno desc;        //降序排列
select empno, ename from emp order by empno asc ;//升序，asc可不写
select * from dept where deptno <> 10 order by deptno desc;//混合使用
select ename, sal, deptno from emp order by deptno asc, ename desc; //混合排序，先deptno排，后ename排，跟excel一样
select ename, sal*12 annual_sal from emp where ename not like '_e%' and sal > 1000 order by sal desc;、//一次总结
8)
常用sql函数
单行:
select lower(ename) from emp; //转换成小写
select ename from emp where lower(ename) like '_e%'; //用法多种多样
select ename from emp where ename like '_e%'or ename like '_E%'; //等价于上面的写法
select upper(ename) from emp; //相对应的转成大写
select substr(ename, 1, 2) from emp; //从第一个字符串开始截两个字符
select char(65) from dual;	//把ASCII码转成字符
select ascii('A') from dual;
select round(23.652) from dual;	//四舍五入
select round(23.452) from dual;
select round(23.652, 1) from dual; //精确到小数点后面多少位
select round(23.652, 2) from dual;
select round(23.652, -1) from dual;//精确到十位数
//oracle才可以用： select to_char(sal, '$99,999.9999') from emp; 
select ename, sal*12 + IFNULL(comm, 0) from emp; //出现null值也能计算的方法
9)
*多行函数：//对多个数值进行操作
select max(sal) from emp;
select min(sal) from emp;
select avg(sal) from emp;
select round(avg(sal),2) from emp;
select count(*) from emp; //有多少条记录
select count(*) from emp where sal = 3000 or sal = 5000; //脑残式混合使用
select sum(sal) from emp; //算总和
10)
select deptno, avg(sal) from emp group by deptno; //分组求平均
select deptno, max(sal) from emp group by deptno, job;    //两个字段组合分组

11）
select avg(sal), deptno from emp group by deptno having avg(sal) > 2000; //在前面的基础下求大于6500的，注意，这里不能用where,用where会错误
总结：
->select * from ? 	//取数据
->where ? <> ?		//过滤
->group by ?		//分组
->having ? <> ?;	//分组限制
select avg(sal), deptno from emp where sal > 2000 group by deptno having avg(sal) > 2000 order by avg(sal) desc; 

12)
子查询，多表联接

select ename , dname, grade from emp e, dept d,salgrade s where e.deptno = d.deptno and e.sal between s.local and s.hisal and job <> 'CLERK';
//1992的语法写法

select ename,dname from emp ,dept;		//1992
select ename,dname from emp cross join dept;	//1999
//交叉连接
select ename, dname from emp,dept where emp.deptno = dept.deptno;//1992
select ename, dname from emp join dept on(emp.deptno = dept.deptno);//1999
select ename, dname from emp join dept using(deptno);//1999简便写法，马士兵不推荐用
//等值连接

select ename,grade from emp e join salgrade s on(e.sal between s.local and s.hisal); //比要写在where要清楚一点

select ename,dname,grade from 
emp e join dept d on(e.deptno = d.deptno)
join salgrade s on (e.sal between s.local and s.hisal)
where ename not like '_A%';
//多表连接

select empno, ename,mgr,mgrname from emp join (select empno mgrempno,ename mgrname from emp) t on(emp.mgr = t.mgrempno) ;
select e1.empno, e1.ename, e1.mgr, e2.ename from emp e1,emp e2 where e1.mgr = e2.empno;//自连接
//两句话是同一个意思，下面的更高效，但上面更常用。这句话意思是求每个人的经理人
select e1.ename,e2.ename from emp e1 join emp e2 on(e1.mgr = e2.empno);
select e1.ename, e2.ename from emp e1 left join emp e2 on(e1.mgr = e2.empno);
//把额外的取出来，左外连接
select empno, ename,mgr,mgrname from emp left join (select empno mgrempno,ename mgrname from emp) t on(emp.mgr = t.mgrempno);

select ename,dname from emp e join dept d on(e.deptno = d.deptno);
select ename,dname from emp e right outer join dept d on(e.deptno = d.deptno);
//右外连接，把额外的拿出来

select ename ,dname from emp e full join dept d on(e.deptno = d.deptno);
//全外连接,很可惜，mysql不支持

13)
子查询，多表联接例子
select ename, sal from emp where sal = (select max(sal) from emp); //选sal最多的一组人
//避免了select ename, max(sal) from emp;这个语句不能对一堆人的查询的错误
select ename, sal from emp where sal > (select avg(sal) from emp); //选sal在平均之上的

select deptno,ename, avg_sal from emp join (select max(sal) max_sal , deptno from emp group by deptno) t on(emp.sal = t.max_sal and emp.deptno = t.deptno);	//求部门那些人薪水最高

select deptno, avg_sal, grade from (select deptno,avg(sal) avg_sal from emp group by deptno) t join salgrade s on (t.avg_sal between s.local and s.hisal);
//求部门平均薪水的等级

select deptno,ename, grade from emp join salgrade s on (emp.sal between s.local and s.hisal);
//每个人的薪水等级

select deptno,avg(grade) from(select deptno,ename, grade from emp join salgrade s on (emp.sal between s.local and s.hisal))t group by deptno;
//求部门平均的薪水等级

select ename from emp where empno in(select distinct mgr from emp);
//雇员中有哪些人是管理者,包括king。

select sal from emp where sal not in(select distinct e1.sal from emp e1 join emp e2 on(e1.sal < e2.sal));
//不用组函数求最高薪水

select max(avg_Sal),deptno from (select avg(sal) avg_sal, deptno from emp group by deptno) e1;
select deptno , avg_sal from (select avg(sal) avg_sal ,deptno from emp group by deptno)e3 where avg_sal = (select max(avg_sal) from (select avg(sal) avg_sal, deptno from emp group by deptno)e1);
//求部门平均薪水的最高的部门编号
//两个方法

select dname from dept where deptno =
(
  select deptno from 
    (select avg(sal) avg_sal ,deptno from emp group by deptno)e3 
      where avg_sal = 
        (select max(avg_sal) from 
	    (select avg(sal) avg_sal, deptno from emp group by deptno)e1
        )
);
//求平均薪水最高的部门的部门名称

select dname, t1.deptno , grade ,avg_sal from
  (
    select deptno, grade, avg_sal from 
      (select deptno, avg(sal) avg_sal from emp group by deptno) t
    join salgrade s on (t.avg_sal between s.local and s.hisal)
  ) t1
  join dept on (t1.deptno = dept.deptno)
  where t1.grade = 
  (
    select min(grade) from
    (
      select deptno, grade , avg_sal from
        (select deptno, avg(sal) avg_sal from emp group by deptno)t
      join salgrade s on (t.avg_sal between s.local and s.hisal)
    )t2
  );
//求平均薪水等级最低的部门的部门名称
create view t as 
select deptno, avg(sal) avg_sal from emp group by deptno;

create view v$_dept_avg_sal_info as
select deptno, grade , avg_sal from t
join salgrade s on (t.avg_sal between s.local and s.hisal); 
//mysql的bug,不能使用子查询

select dname, t1.deptno , grade ,avg_sal from
  v$_dept_avg_sal_info t1
  join dept on (t1.deptno = dept.deptno)
  where t1.grade = (select min(grade)from v$_dept_avg_sal_info);
//求平均薪水等级最低的部门的部门名称(使用视图)视图的小科普~

select ename from emp
where empno in (select distinct mgr from emp where mgr is not null) and
sal >
(
  select max(sal) from emp 
  where empno not in (select distinct mgr from emp where mgr is not null)
);
//比普通员工的最高薪水还要高的经理人名称

select ename,empno,sal from emp order by sal desc limit 0,5; //从第0行起取5个
//取薪水最高的前5名员工
select ename, sal from (select ename ,sal fron emp order by sal desc) 
where rownum <= 5;
//不知道行不行的方法select ename,empno,sal from emp order by sal desc where rownum <= 5;
//oracle写法，有一定限制（如where rownum > 10;>不能用，只能用<或者<=）马士兵说(oracle)脑子进水了....（比较隔的设定）,解决方法：select ename from(select rownum r,ename from emp) where r > 10;


面试题：
有3个表S,C,SC
S(SNO,SNAME)(学号,姓名)
C(CNO,CNAME.CTEACHER)(课号，课名，教师)
SC(SNO,CNO,SCGRADE)(学号，课号，成绩)

select sname from s join sc on(s.sno = sc.sno) join c(c.cno = sc.cno) where c.cteacher <> 'liming';
//找出没选过"liming"老师的所有学生姓名

select sname where sno in(select sno from sc where scgrade < 60 group by sno having count(*) >= 2);
//列出2门以上（含2门）不及格学生姓名及平均成绩

select sname from s where sno in(select sno from sc where cno = 1 and sno in(select sno from sc where cno = 2));
//学过1号课程又学过2号课程的所有学生的姓名




常见用户语句(oracle):
drop user xxx ;

create user guanhong identified by guanhong defualt tablespace users quota 10M on users
//创建新用户
insert into mysql.user(host,user,password) values ("localhost","guanhong",password("guanhong"));


插入语句：
insert 

desc dept;

create table emp2 as select * from emp;
create table dept2 as select * from dept;
create table salgrade2 as select * from salgrade;
create table avc2 as select * from avc;
create table emp3 as select * from emp;

insert into dept2 values(50,'game','bj');
insert into dept2(deptno,dname) values(60,'game2');
insert into dept2 select * from dept;

修改语句：
update

update emp2 set sal=sal*2, ename=concat(ename,'-') where deptno = 10;

删除语句：
delete

delete from emp2;
delete from dept2 where deptno <= 25;

ddl数据定义语句(使用后事务自动提交)：

create table tb (a varchar(10));
drop table tb;

create总汇：

create table xx（表名）(xx xxx,xx xxx);(字段名 类型)

create table class
(
id tinyint primary key,
name varchar(20) not null
);
create table student 
(
id int,
name varchar(255) not null,  
/* mySQL直接not null就是约束,不用constraint  */
sex tinyint,
age tinyint,
sdate bigint, 
grade tinyint default 1 , 
class tinyint , 
email varchar(50),
/*email varchar(50) unique */
constraint stu_class_fk foreign key(class) references class(id),
/*外键约束*/
constraint stu_id_pk primary key(id),
/*主键约束两种写法1）如上2）id int primary key;*/
constraint stu_name_email_uni unique(email,name)
/*对表约束,email和name的连接是唯一的*/
);
以下是测试：
insert into class(id,name) values(100,'1班');
insert into student(id, name,class,email) values(10,'a',100,'a');
/*存在100,所以插入成功*/
insert into student(id, name,class,email) values(1,'b',10,'b');
/*不存在10,所以插入失败*/
delete from class where id = 100;
/*违反约束条件，所以不能删除*/

修改表结构：

alter table student add(addr varchar(100));
//添加字段

alter table student drop addr;
//删除字段

alter table student modify addr varchar(150);
desc student;
alter table student modify addr varchar(50);
desc student;
//修改字段

alter table student drop foreign key stu_class_fk;
//删除约束条件

alter table student add constraint stu_class_fk foreign key(class) references class(id);
//添加约束条件

drop table student;
//删除表

数据字典表(view,约束，表):
show tables;
//查当前数据库的表名含有视图
SELECT table_name FROM information_schema.`VIEWS` where table_schema= 'mydata';
//查看视图

SELECT * FROM information_schema.`TABLE_CONSTRAINTS` where table_name = 'student';
//查看约束 

SELECT * FROM information_schema.`TRIGGERS`;
//查看触发器 

SELECT table_name FROM information_schema.`tables`;
//查看mysql总数据字典表

index(索引)：
create index idx_std_email on student (email);
//对字段的建立索引
//访问一个字段访问量大，访问效率低的时候可以用索引，但不要轻易建立（确实有用的时候才建）因为索引占用大量空间

create index idx_std_email on student (email,class);
//对两个字段的组合建立索引

drop index idx_std_email;
//删除索引

show index from student;
show keys from student;
//查看索引

view（视图）:
//视图就是一个字查询
//试图建立要付出代价，就是视图维护比较麻烦，比如主表改了，视图也要跟着改

create view v$_stu as select id,name,age from student;
//视图的保护作用：可以只公开表中的某些信息，不完全公开

desc v$_stu;
//描述一下视图

*视图可以用来更新数据，但是一般不用


oracle序列（建立唯一的不间断地序列，作主键）：

create table article
(
id number,
title varchar2(1024),
cont long
)
create sequence seq;/*创建序列*/
select seq.nextval from dual;
insert into article values(seq.nextval,'a','b');
insert into article values(seq.nextval,'a','b');
insert into article values(seq.nextval,'a','b');
insert into article values(seq.nextval,'a','b');

drop sequence seq/*删除序列*/

mysql自动递增:
create table article
(
id int NOT NULL auto_increment,
/*自动递增*/
title varchar(1024),
cont longtext,
primary key(id)
);


三范式：

范式：数据库设计的一些应当遵循的规则,姓范的兄弟制定出来的.............
很有用，应该具体问题具体分析，必要时应该打破

第一范式：不存在冗余数据
1)要有主键
2)列不可分（例：就是不能把学号，姓名什么的合成一个字段写info 2012_guanhong）
范式二：
当一张表有多个字段作为主键的时候，非主键的字段不能够依赖与部分主键(不能存在部分依赖)。
范式三：
每个非关键字列都独立于其他非关键字列，并依赖于关键字。

触发器(oracle,mysql待定):
create table emp2_log
(
uname varchar(20),
action varchar(10),
atime date
);

create or replace trigger trig
    after insert or delete or update on emp2 /*for each row*///每次更新一条就做一次操作
begin
    if inserting then
        insert into emp2_log values (user,'insert',sysdate);
    elsif updating then
        insert into emp2_log values (user,'update',sysdate);
    elsif deleting then
        insert into emp2_log values (user,'delete',sysdate);
    end if;
end;

树状结构展示：
drop table article;

create table article
(
id int primary key,
cont varchar(4000),
pid int,
idleaf int(1),
alevel int(2)
);

insert into article values (1,'蚂蚁大战大象',0,0,0);
insert into article values (2,'大象被打趴下了',1,0,1);
insert into article values (3,'蚂蚁也不好过',2,1,2);
insert into article values (4,'瞎说',2,0,2);
insert into article values (5,'没有瞎说',4,1,3);
insert into article values (6,'怎么可能',1,0,1);
insert into article values (7,'怎么没有可能',6,1,2);
insert into article values (8,'可能性很大的',6,1,2);
insert into article values (9,'大象进医院了',2,0,2);
insert into article values (10,'护士是蚂蚁',9,1,3);

蚂蚁大战大象
    大象被打趴下了
	蚂蚁也不好过
	瞎说
	    没有瞎说
	大象进医院了
	    护士是蚂蚁
    怎么可能
	怎么没有可能
	可能性很大的

oracle的函数(展示递归)：
create or replace procedure p(v_pid article.pid%type,v_level binary_integer) is
  cursor c is select * from article where pid = v_pid;
  v_preStr varchar(1024) := '';
begin 
  for i in 1..v_level loop
    v_preStr := v_preStr || '    ';
  end loop
  for v_article in c loop
    dbms_output.put_line(v_preStr || v_article.cont);
    if(v_article.isleaf = 0) then
      p(v_article.id , v_level + 1);
    end if;
  end loop;
end;
set serveroutput on;
exec p(0);



Transaction(事务)语句：


dcl语句:

```
