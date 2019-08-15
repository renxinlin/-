一、基础规范

（1）必须使用InnoDB存储引擎

        解读：支持事务、行级锁、并发性能更好、CPU及内存缓存页优化使得资源利用率更高
        Myisam引擎使用场景：
         数据库端的并发数量不多（２０％写，８０％读）
         读操作比较多，而且都能很好的利用索引
         sql语句比较简单的应用
         轻松达到TB级数据量的数据仓库
        InonoDB引擎适用的场景
          数据库端的读写并发数量非常多
          写操作比较多，TB级数据量应用
          数据量较小，索引不好利用的应用比较多
          有外键、事务等需求的应用
（2）必须使用UTF8字符集

    解读：万国码，无需转码，无乱码风险，节省空间
（3）数据表、数据字段必须加入中文注释

    解读：请不要给后来维护的人挖坑
（4）禁止使用存储过程、视图、触发器、Event

    解读：高并发大数据的互联网业务，架构设计思路是“解放数据库CPU，将计算转移到服务层”，并发量大的情况下，这些功能很可能将数据库拖死，业务逻辑放到服务层具备更好的扩展性，能够轻易实现“增机器就加性能”。数据库擅长存储与索引，CPU计算还是上移吧
（5）禁止存储大文件或者大照片

    解读：为何要让数据库做它不擅长的事情？大文件和照片存储在文件系统，数据库里存URI
二、命名规范

（6）只允许使用内网域名，而不是ip连接数据库

（7）线上环境、开发环境、测试环境数据库内网域名遵循命名规范

    业务名称：xxx
    线上环境：dj.xxx.db
    开发环境：dj.xxx.rdb
    测试环境：dj.xxx.tdb
    从库在名称后加-s标识，备库在名称后加-ss标识
    线上从库：dj.xxx-s.db
    线上备库：dj.xxx-sss.db
（8）库名、表名、字段名：小写，下划线风格，不超过32个字符，必须见名知意，禁止拼音英文混用

（9）表名txxx，临时表请用tmpxxx, 非唯一索引名idx_字段名称字段名称，唯一索引名uniq字段名称字段名称

三、表设计规范

（10）单实例表数目必须小于500

（11）单表列数目必须小于30, 表数据量预期会超过5千万的，必须进行分表或分库拆分

    解读：
    a)如果使用md5（或者类似的hash算法）对表进行分区，表名后缀使用16进制，比如user_ff
    b)推荐使用CRC32求余（或者类似的算术算法）对表进行分区，表名后缀使用数字，数字必须从0开始并等宽，比如拆分100张表，后缀从00-99
    c)使用时间对表进行分区，表名后缀必须使用特定格式，比如按日拆分表user_20110209、按月进行分区user_201102
（12）表必须有主键，例如自增主键 (自增主键必须命名为id)

    解读：
    a）主键递增，数据行写入可以提高插入性能，可以避免page分裂，减少表碎片提升空间和内存的使用
    b）主键要选择较短的数据类型， Innodb引擎普通索引都会保存主键的值，较短的数据类型可以有效的减少索引的磁盘空间，提高索引的缓存效率
    c）无主键的表删除，在row模式的主从架构，会导致备库夯住
（13）禁止使用外键，如果有外键完整性约束，需要应用程序控制

    解读：外键会导致表与表之间耦合，update与delete操作都会涉及相关联的表，十分影响sql 的性能，甚至会造成死锁。高并发情况下容易造成数据库性能，大数据高并发业务场景数据库使用以性能优先
四、字段设计规范

（14）必须把字段定义为NOT NULL并且提供默认值

    解读：
    a）null的列使索引/索引统计/值比较都更加复杂，对MySQL来说更难优化
    b）null 这种类型MySQL内部需要进行特殊处理，增加数据库处理记录的复杂性；同等条件下，表中有较多空字段的时候，数据库的处理性能会降低很多
    c）null值需要更多的存储空，无论是表还是索引中每行中的null的列都需要额外的空间来标识
    d）对null 的处理时候，只能采用is null或is not null，而不能采用=、in、<、<>、!=、not in这些操作符号。如：where name!=’shenjian’，如果存在name为null值的记录，查询结果就不会包含name为null值的记录
（15）禁止使用TEXT、BLOB类型

    解读：会浪费更多的磁盘和内存空间，非必要的大量的大字段查询会淘汰掉热数据，导致内存命中率急剧降低，影响数据库性能。VARCHAR性能远远好于二者，请尽量使用VARCHAR。
（16）禁止使用小数存储货币

    解读：使用整数或者使用DECIMAL
（17）必须使用varchar(20)存储手机号

    解读：
    a）涉及到区号或者国家代号，可能出现+-()
    b）手机号会去做数学运算么？
    c）varchar可以支持模糊查询，例如：like“138%”
（18）禁止使用ENUM，可使用TINYINT代替

    解读：
    a）增加新的ENUM值要做DDL操作
    b）ENUM的内部实际存储就是整数，你以为自己定义的是字符串？
（19）每张表都应该有如下两个字段：

    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '记录创建时间',
    `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP  COMMENT '最后修改时间'
    解读： 可以直观看出什么时间创建的，最后修改是什么时间，使用相应的默认值保证就算不赋值，也能正确的更新时间。
（20）列的类型不能使用集合、枚举、位图类型

五、索引设计规范

（21）单表索引建议控制在5个以内

（22）单索引字段数不允许超过5个

    解读：字段超过5个时，实际已经起不到有效过滤数据的作用了
（23）禁止在更新十分频繁、区分度不高的属性上建立索引

    解读：
    a）更新会变更B+树，更新频繁的字段建立索引会大大降低数据库性能
    b）“性别”这种区分度不大的属性，建立索引是没有什么意义的，不能有效过滤数据，性能与全表扫描类似
（24）建立组合索引，必须把区分度高的字段放在前面

    解读：能够更加有效的过滤数据
六、SQL使用规范

（25）禁止使用SELECT *，只获取必要的字段，需要显示说明列属性

    解读：
    a）读取不需要的列会增加CPU、IO、NET消耗
    b）不能有效的利用覆盖索引
    c）使用SELECT *容易在增加或者删除字段后出现程序BUG
（26）禁止使用INSERT INTO t_xxx VALUES(xxx)，必须显示指定插入的列属性

    解读：容易在增加或者删除字段后出现程序BUG
（27）禁止使用属性隐式转换

    解读：SELECT uid FROM t_user WHERE phone=13812345678 会导致全表扫描，而不能命中phone索引，猜猜为什么？（这个线上问题不止出现过一次）
（28）禁止在WHERE条件的属性上使用函数或者表达式

    解读：SELECT uid FROM t_user WHERE from_unixtime(day)>='2017-02-15' 会导致全表扫描
    正确的写法是：SELECT uid FROM t_user WHERE day>= unix_timestamp('2017-02-15 00:00:00')
（29）禁止负向查询，以及%开头的模糊查询

    解读：
    a）负向查询条件：NOT、!=、<>、!<、!>、NOT IN、NOT LIKE等，会导致全表扫描
    b）%开头的模糊查询，会导致全表扫描
（30）禁止大表使用JOIN查询，禁止大表使用子查询

    解读：会产生临时表，消耗较多内存与CPU，极大影响数据库性能
（31）禁止使用OR条件，必须改为IN查询

    解读：旧版本Mysql的OR查询是不能命中索引的，即使能命中索引，为何要让数据库耗费更多的CPU帮助实施查询优化呢？
（32）where子句中限制条件排列最好为对应索引的最左前缀，将range(范围)限制条件调整到where子句的最后(最右)部分

    举例：select account_id from consume where expenditure>1.00 and account_payee =72478814;
    如上语句，expenditure>1.00是一个范围限制条件，可以调整其至account_payee条件之后。
    同时设计索引index_accountpayee_expenditure时考虑到这个问题，索引中列顺序为(account_payee,expenditure)。
（33）当where子句中有2个以上的范围限制条件时，只保留一个范围限制条件，并且必须放在最后，其它范围限制条件需要改写为等值条件或in（）

    举例：由于第一个范围限制条件后的范围条件对应的索引列无法在索引中被使用，因此将语句：
    select account_id from consume where account_payee between 51906734 and 51907000 and expenditure>0.00;
    改写为：
    select account_id from consume where account_payee in (51907000, 51906734, 51906740) and expenditure>0.00;
    就可以使用索引index_accountpayee_expenditure中包含的全部列信息(index index_accountpayee_expenditure(account_payee,expenditure))
（34） 位于select子句中的子查询，改写为外连接的查询

    举例：select a.account_id,a.balance,
    (select c.customer_name
    from customer as c
    where c. customer_id=ac.customer_id )as cust_name
    from account as a, cust_acct as ac
    where a. account_id=ac.account_id and a. account_id= 'a-001';
    改写为：
    select a.account_id,a.balance,c.customer_id
    from account as a, cust_acct as ac left join customer as c
    on c.customer_id = ac.customer_id
    where a.account_id=ac.account_id and a.account_id= 'a-001';
（35）位于where中带in的嵌套子查询，改写为表关联查询

举例： 和张三有在同一银行开户的账号的客户信息，并且按照customer_id递减排序，取前10个客户信息。
select c. customer_name,c.customer_id
from customer as c, cust_acct as ac, account as a
where c.customer_id=ac.customer_id and a.account_id=ac.account_id and a. account_bank in(
select a2.account_bank
from account as a2,cust_acct as ac2,customer as c2
where c2.customer_name='张三' and c2.customer_id=ac2.customer_id and ac2.account_id=a2.account_id
)
order by c.customer_id desc limit 0,10;
改写为：
select c.customer_name,c.customer_id
from customer as c,cust_acct as ac,account as a,
(
select distinct a2.account_bank
from account as a2,cust_acct as ac2,customer as c2
where c2.customer_name='张三' and c2.customer_id=ac2.customer_id and ac2.account_id=a2.account_id
) as tr
where tr. account_bank=a. account_bank
and c.customer_id=ac.customer_id and a.account_id=ac.account_id
order by c.customer_id desc limit 0,10;
    这两种写法的对比：
    1.  from子句支持limit子句，在索引合适的情况下，添加order by和limit在子查询中可减少文件排序。
    2.  带in的子查询和联表比起来，MySQL更喜欢联表。尤其是in()中子查询为空时，外层查询会逐行扫描对比，而此时联表查询会有较大优势。
（36）应用程序必须捕获SQL异常，并有相应处理
