# PostgreSQL deferrable initially deferred 造成的死锁问题

## 太长不看

PostgreSQL可以在约束（constraint）上使用`deferrable`特性，在执行插入/更新语句时，推迟检查约束。其中`deferrable initially deferred`推迟至`commit`语句执行时进行检查，而`deferrable initially immediate`推迟至该语句执行完成时进行检查。使用不当可能产生死锁问题。

## 现象

PostgreSQL日志出现死锁记录

```postgresql
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:ERROR:  deadlock detected
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:DETAIL:  Process 2699 waits for ShareLock on transaction 640992196; blocked by process 441.
	Process 441 waits for ShareLock on transaction 640992182; blocked by process 2699.
	Process 2699: COMMIT
	Process 441: update item set event_seq=$1 where id=$2
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:HINT:  See server log for query details.
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:CONTEXT:  while checking exclusion constraint on tuple (2654776,52) in relation "annotation"
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:STATEMENT:  COMMIT
```

## 环境

### 数据库
运行在AWS RDS db.r4.2xlarge实例上的postgres 9.5.10

### 表
两个实体表，`item`  1:N `annotation`。`annotation`表通过`item_id`关联`item`表，并有唯一性约束（item_id, type）。

```
                        Table "item"
    Column     |           Type           |       Modifiers        
---------------+--------------------------+------------------------
 id            | uuid                     | not null
 event_seq     | bigint                   | not null default 0
Indexes:
    "item$pk" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "annotation" CONSTRAINT "annotation$item$fk" FOREIGN KEY (item_id) REFERENCES item(id)
```

```
                    Table "annotation"
   Column    |           Type           |       Modifiers        
-------------+--------------------------+------------------------
 id          | uuid                     | not null
 item_id     | uuid                     | not null
 type        | character varying(128)   | not null
Indexes:
    "annotation$pk" PRIMARY KEY, btree (id)
    "annotation$item_id_type$uk" EXCLUDE USING btree (item_id, type) DEFERRABLE INITIALLY DEFERRED
Foreign-key constraints:
    "annotation$item$fk" FOREIGN KEY (item_id) REFERENCES item(id)

```
### 服务端
* org.springframework.boot:spring-boot-starter-web:1.4.1.RELEASE
* org.springframework.boot:spring-boot-starter-data-jpa:1.4.1.RELEASE
* org.postgresql:postgresql:9.4.1208

#### 实体定义
```java
@Entity
public class Item {

    @Id
    @Type(type = "pg-uuid")
    private UUID id;

    private long eventSeq;

    @OneToMany(mappedBy = "item", cascade = CascadeType.ALL, fetch = FetchType.EAGER, orphanRemoval = true)
    private List<Annotation> annotations = new ArrayList<>();
}
```

```java
@Entity
@Table(uniqueConstraints = {@UniqueConstraint(columnNames = {"type", "item_id"})})
public class Annotation {

    @Id
    @Type(type = "pg-uuid")
    private UUID id;

    @Column
    private String type;

    @ManyToOne()
    @JoinColumn(name = "item_id")
    private Item item;
}
```

#### JPA定义

```java
public interface ItemRepository extends JpaRepository<Item, UUID>, JpaSpecificationExecutor<Item> {}
```

#### 调用JPA保存
```java
@Component
public class ItemServiceImpl implements ItemService {
    @Autowired
    private ItemRepository repository;

    @Override
    public void saveItem(Item item) {
            repository.save(item);
    }
}
```

## 原因分析
### 名词解释
PostgreSQL比较各色，对很多概念给了自己的称呼。为了避免误导，稍微翻译下。
`relation`，就是数据表——`table`
`tuple`，就是数据表的一行——`row`

### 问题定位
```log
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:ERROR:  deadlock detected
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:DETAIL:  Process 2699 waits for ShareLock on transaction 640992196; blocked by process 441.
	Process 441 waits for ShareLock on transaction 640992182; blocked by process 2699.
	Process 2699: COMMIT
	Process 441: update item set event_seq=$1 where id=$2
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:HINT:  See server log for query details.
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:CONTEXT:  while checking exclusion constraint on tuple (2654776,52) in relation "annotation"
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:STATEMENT:  COMMIT
```

从日志文件中看，441与2699两个数据库线程死锁。但是刚看到的时候没法理解：一个线程`COMMIT`，一个线程`update item`，怎么会死锁呢？
我们知道，死锁是由于两个线程分别持有锁，同时需要申请对方所持有的锁。
但在代码中只有一个`.save(item)`，查看实际执行的SQL为
```sql
begin;
insert into annotation (id,item_id,type) values('fdbf4e61-dcf6-4b40-87e6-c81c0a14c001','cd0dbe92-764a-4a91-af5b-055d9387f4b6','note');
update item set event_seq=30000 where id='cd0dbe92-764a-4a91-af5b-055d9387f4b6';
commit;
```
即使多线程并发，执行顺序也保证了先锁`annotation`表，再锁`item`表，不会循环加锁。

没什么进展。重新再看看日志
```log
Process 441: update item set event_seq=$1 where id=$2
```
441线程比较清楚，update被阻塞了，这么说来2699已经持有了`item`表的锁。
可是2699被441阻塞了什么呢？
```postgresql
2018-07-02 06:45:15 UTC:${IP_ADDR}(41404):${ACCOUNT}@${DB}:[2699]:CONTEXT:  while checking exclusion constraint on tuple (2654776,52) in relation "annotation"
```
ummm....2699在`commit`的时候，需要“检查annotation表上的唯一约束”？
似乎跟`annotation`表上配置的
```postgresql
"annotation$item_id_type$uk" EXCLUDE USING btree (item_id, type) DEFERRABLE INITIALLY DEFERRED
```
能联系起来？

查找资料之后发现，`DEFERRABLE INITIALLY DEFERRED`的作用是将唯一约束检查推迟到`commit`时进行。这就能说通了：441提交时，需要检查`annotation`表的唯一性，也就需要对`annotation`表加锁。如果2699已经持有了`annotation`表的锁，441就得等待。所以这俩线程就互相持有对方需要的锁。

两个线程以如下顺序执行SQL，就会出现死锁：

```sql
--2699:
begin;
insert into annotation (id,item_id,type) values('adbf4e61-dcf6-4b40-87e6-c81c0a14c001','ad0dbe92-764a-4a91-af5b-055d9387f4b6','note');

--441:
begin;
insert into annotation (id,item_id,type) values('adbf4e61-dcf6-4b40-87e6-c81c0a14c002','ad0dbe92-764a-4a91-af5b-055d9387f4b6','note');

--2699:
update item set event_seq=30000 where id='cd0dbe92-764a-4a91-af5b-055d9387f4b6';

--441:
update item set event_seq=40000 where id='cd0dbe92-764a-4a91-af5b-055d9387f4b6';

--2699:
commit;
```

### 解决方案
1. 问题是由`deferrable initially deferred`导致的。如果不加上这个选项，2699线程在`insert into annotation`的时候就会阻塞，也就不会产生死锁问题。

2. 调整SQL语句顺序，先执行`update item`，再执行`insert into annotation`。先尝试对`item`表加锁。不过对于JPA框架来说不好调整。

3. 由于在我们的实际业务中，是由于完全相同的两个请求同时到达服务器导致的，所以最终解决方案是：放着不管：）。一个线程检测到死锁，抛异常回滚。另一个线程可以正常提交。对实际业务没有影响。


## 参考链接
[PostgreSQL文档](https://www.postgresql.org/docs/9.5/static/sql-set-constraints.html)
[stackoverflow上关于deferrable含义的讨论](https://stackoverflow.com/q/5300307/)
