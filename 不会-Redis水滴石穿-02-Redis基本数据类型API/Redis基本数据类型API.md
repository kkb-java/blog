# redis学习笔记（二、基本数据类型API）

## 一、Redis数据类型

官方命令大全网址：http://www.redis.cn/commands.html
Redis 中存储数据是通过 key-value 格式存储数据的，其中 value 可以定义五种数据类型：
String（字符类型） 

Hash（散列类型） 

List（列表类型） 

Set（集合类型） 

SortedSet（有序集合类型，简称zset）

### Redis key 简介

Redis key值是二进制安全的，这意味着可以用任何二进制序列作为key值，从形如”foo”的简单字符串到一个JPEG文件的内容都可以。空字符串也是有效key值。

#### 关于key的几条规则：

- 键值不建议太长，例如1024字节或者更大的键值，不仅消耗内存，而且在数据中检索这类键值的计算成本很高。
- 键值应该见名知意，比如用”user:1000:password”来代替”u:1000:pwd”，这样更容易阅读理解。
- 所有的键值都是字符串类型，其长度不能超过512MB
- 在 redis 中的命令语句中，命令忽略大小写，而 key 是不忽略大小写的。

#### key 的自动创建和删除

这适用于所有包括多个元素的 Redis 数据类型 – List ，Set, Sorted Set 和 Hash。

基本上，我们可以用三条规则来概括它的行为：

1. 当我们向一个聚合数据类型中添加元素时，如果目标键不存在，就在添加元素前创建空的聚合数据类型。
2. 当我们从聚合数据类型中移除元素时，如果值仍然是空的，键自动被销毁。
3. 对一个空的 key 调用一个只读的命令，比如 LLEN （返回 list 的长度），或者一个删除元素的命令，将总是产生同样的结果。该结果和对一个空的聚合类型做同个操作的结果是一样的。

### 1、Redis String

 String是Redis中最基本的数据类型，它可以包含任意类型的数据。

你可以用Redis字符串做许多有趣的事，例如你可以：

- 利用INCR命令簇（INCR, DECR, INCRBY）来把字符串当作原子计数器使用。
- 使用APPEND命令在字符串后添加内容。
- 将字符串作为GETRANGE和 SETRANGE的随机访问向量。
- 在小空间里编码大量数据，或者使用 GETBIT和 SETBIT创建一个Redis支持的Bloom过滤器。

#### 命令

##### 1.1 赋值 set

这里需要注意一点，如果key已经保存了一个值，那么这个操作会直接覆盖原来的值，当set命令执行成功之后，之前设置的过期时间都将失效。

- 语法

  ```java
  set key value
  ```

- 示例

  ```java
  127.0.0.1:6379> set key1 hello
  OK
  ```

##### 1.2 取值 get

- 语法

  ```java
  get key
  ```

- 示例

  ```java
  127.0.0.1:6379> get key1
  "hello"
  ```

##### 1.3 数值增减 incr/decr

需要注意的是

​	1、 当value为整数数据时，才能使用以下命令操作数值的增减。

​	2、 数值递增都是【原子】操作。

- 语法

  ```java
  incr key  //增
  decr key  //减
  incrby key increment   //增加指定的数
  decrby key decrement   //减少指定的数
  ```

- 示例

  ```java
  127.0.0.1:6379> set key.int  1
  OK
      
  //增    
  127.0.0.1:6379> incr key.int
  (integer) 2
  127.0.0.1:6379> get key.int
  "2"
      
  //减 
  127.0.0.1:6379> decr key.int
  (integer) 1
  127.0.0.1:6379> get key.int
  "1"
  
  //指定增
  127.0.0.1:6379> incrby key.int 3
  (integer) 4
  127.0.0.1:6379> get key.int
  "4"
    
  //指定减    
  127.0.0.1:6379> decrby key.int 3
  (integer) 1
  127.0.0.1:6379> get key.int
  "1"
      
  //因为key1为非数值，所以会报错
  127.0.0.1:6379> incr key1
  (error) ERR value is not an integer or out of range      
  ```

##### 1.4 尾部追加 append

APPEND 命令，向键值的末尾追加 value 。
如果键不存在则将该键的值设置为 value ，即相当于 SET key value 。返回值是追加后字符串的总长度。

- 语法

  ```java
  append key value
  ```

- 示例

  ```java
  //key存在时的append
  127.0.0.1:6379> get key1
  "hello"
  
  127.0.0.1:6379> append key1 " world!"
  (integer) 12
  
  127.0.0.1:6379> get key1
  "hello world!"
  
  //key不存在时的append
  127.0.0.1:6379> get key2 
  (nil)
      
  127.0.0.1:6379> append key2 hello
  (integer) 5
      
  127.0.0.1:6379> get key2
  "hello"
  ```

##### 1.5 获取子串 getrange

 这个命令是被改成GETRANGE的，在小于2.0的Redis版本中叫SUBSTR。 返回key对应的字符串value的子串，这个子串是由start和end位移决定的（两者都在string内）。可以用负的位移来表示从string尾部开始数的下标，-1最后一个字符，-2就是倒数第二个，以此类推。 

- 语法

  ```java
  getrange key start end
  ```

- 示例

  ```java
  127.0.0.1:6379> get key1
  "hello world!"
      
  127.0.0.1:6379> getrange key1 0 4
  "hello"
      
  127.0.0.1:6379> getrange key1 -6 -1
  "world!"
  ```


### 2、Redis Hash

Hash 便于表示 *objects*，实际上，你可以放入一个 hash 域的数量实际上没有限制（除了可用内存以外）。所以，你可以在你的应用中以不同的方式使用 hash。 

Redis Hashes 是字符串字段和字符串值之间的映射，由field和关联的value组成的map，field和value都是字符串，它是完美的表示对象（eg:一个用户对象，包括姓名、密码、用户名、密码等属性）的数据类型。 

一个拥有少量（100个左右）字段的hash需要很少的空间来存储，所有你可以在一个小型的 Redis实例中存储上百万的对象。

尽管Hash主要用来表示对象，但它们也能够存储许多元素，所以你也可以用Hash来完成许多其他的任务。

一个hash最多可以包含(2^32)-1 个key-value键值对（超过40亿）。

#### 命令

##### 2.1 赋值  hset/hmset

- 语法

  ```Java
  //设置一个字段值
  //该命令命令不区分插入和更新操作，当执行插入操作时 HSET 命令返回 1 ，当执行更新操作时返回 0 
  hset key field value
  
  //设置多个字段值
  hmset key field value [field value ...]
  ```

- 示例

  ```java
  //设置一个字段值
  
  //插入，返回1
  127.0.0.1:6379> hset  user  username zhangsan
  (integer) 1
      
  //更新，返回0    
  127.0.0.1:6379> hset  user  username lisi
  (integer) 0
      
  //设置多个字段值
  127.0.0.1:6379> hmset user username zhangsan password 111 age 18
  OK
      
  127.0.0.1:6379> hmset user username zhangsan password 111 age 18 sex gender
  OK
  
  ```

##### 2.2 取值 hget/hmget

- 语法

  ```java
  //获取一个字段值
  hget key field
      
  //获取多个字段值
  hmget key [field ...]
  ```

- 示例

  ```java
  //获取一个字段值
  127.0.0.1:6379> hget user username
  "zhangsan"
      
  //获取多个字段值
  127.0.0.1:6379> hmget user username password  age  sex
  1) "zhangsan"
  2) "111"
  3) "18"
  4) "gender"
  ```

##### 2.3 只获取字段名/值 hkeys/hvals

- 语法

  ```java
  //只获取字段名
  hkeys key
      
  //只获取字段值    
  hvals key
  ```

- 示例

  ```java
  //只获取字段名
  127.0.0.1:6379> hkeys  user
  1) "username"
  2) "password"
  3) "age"
  4) "sex"
  
  //只获取字段值  
  127.0.0.1:6379> hvals user
  1) "zhangsan"
  2) "111"
  3) "18"
  4) "gender"
  127.0.0.1:6379>
  ```

##### 2.4 获取字段数量/获取所有信息 hlen/hgetall

- 语法

  ```java
  //获取字段数量
  hlen key
  
  //获取所有的字段名、字段值
  hgetall key
  ```

- 示例

  ```java
  //获取字段数量
  127.0.0.1:6379> hlen  user
  (integer) 4
  
  //获取所有的字段名、字段值
  127.0.0.1:6379> hgetall user
  1) "username"
  2) "zhangsan"
  3) "password"
  4) "111"
  5) "age"
  6) "18"
  7) "sex"
  8) "gender"
  ```

##### 2.5 删除字段 HDEL

- 语法

  ```java
  //可以删除一个或多个字段
  //返回值是被删除字段的个数；如果删除的key或field不存在则返回0
  HDEL key field [field ...]
  ```

- 示例

  ```Java
  //删除一个字段，存在返回1，不存在返回0
  127.0.0.1:6379> hdel user age
  (integer) 1
  
  127.0.0.1:6379> hdel user age       
  (integer) 0
  
  127.0.0.1:6379> hdel user agexxxxx    //field不存在
  (integer) 0
      
  127.0.0.1:6379> hdel userrrr  user    //key不存在
  (integer) 0
      
  //删除多个字段
  127.0.0.1:6379> hdel user  username password
  (integer) 2
  ```

### 3、Redis List

Redis 的列表类型（ list 类型）可以存储一个有序的字符串列表，按照插入顺序排序，你可以添加一个元素到列表的头部（左边）或者尾部（右边）。常用的操作是向列表两端添加元素，或者获得列表的某一个片段， 当对一个空key执行其中某个命令时，将会创建一个新表 。

#### 命令

##### 3.1 赋值 lpush/rpush

将指定的值插入到存key 的列表的头部/尾部。如果 key 不存在，那么在进行 push 操作前会创建一个空列表，然后返回在 push 操作后的列表长度。如果 key 对应的值不是一个 list 的话，那么会返回一个错误。

- 语法

  ```java
  //从列表的头部插入，可以插入一个或多个值
  LPUSH key value [value ...] 
  
  //从列表的尾部插入，可以插入一个或多个值
  RPUSH key value [value ...]
  
  ```

- 示例

  ```java
  //从列表的头部插入,此时列表头部第一个元素是g，最后一个元素是a
  127.0.0.1:6379> lpush list1 a b c d e f g
  (integer) 7
  
  //从列表的尾部插入，此时列表第一个元素是a，最后一个元素是g
  127.0.0.1:6379> rpush list2 a b c d e f g
  (integer) 7
      
  ```

##### 3.2 范围遍历 lrange

返回存储在 key 的列表里指定范围内的元素。 start 和 end 偏移量都是基于0的下标，即list的第一个元素下标是0（list的表头），第二个元素下标是1，以此类推；

偏移量也可以是 负数，-1为列表的最后一个元素，-2为倒数第二个元素，以此类推;

当下标超过list范围的时候不会产生error。 如果start比list的尾部下标大的时候，会返回一个空列表。 如果stop比list的实际尾部大的时候，Redis会当它是最后一个元素的下标。 

- 语法

  ```java
  lrange key start stop
  ```

- 示例

  ```java
  127.0.0.1:6379> lpush list1 e d c b a  
  (integer) 5
      
  127.0.0.1:6379> rpush list1 f g h i j 
  (integer) 10
  
   //遍历list1，从第一个元素开始，直到最后一个
  127.0.0.1:6379> lrange list1 0 -1
   1) "a"
   2) "b"
   3) "c"
   4) "d"
   5) "e"
   6) "f"
   7) "g"
   8) "h"
   9) "i"
  10) "j"
  
  ```

##### 3.3 弹出值 lpop/rpop

移除并返回存于 key 对应的 list 队头/队尾的一个元素，当key不存在时，返回nil。 

- 语法

  ```java
  LPOP key
  RPOP key
  ```

- 示例

  ```java
  //此时列表元素为 a b c d e f g h i j 
  127.0.0.1:6379> lrange list1  0 -1
   1) "a"
   2) "b"
   3) "c"
   4) "d"
   5) "e"
   6) "f"
   7) "g"
   8) "h"
   9) "i"
  10) "j"
  
  //弹出列表头部元素a
  127.0.0.1:6379> lpop list1
  "a"
      
  //弹出列表尾部元素j
  127.0.0.1:6379> rpop list1
  "j"
      
  //弹出列表头部元素b
  127.0.0.1:6379> lpop list1
  "b"
      
  //弹出列表尾部元素i
  127.0.0.1:6379> rpop list1
  "i"
      
  //此时列表元素为 c d e f g h     
  127.0.0.1:6379> lrange list1 0 -1
  1) "c"
  2) "d"
  3) "e"
  4) "f"
  5) "g"
  6) "h"
  
  ```

##### 3.4 获取元素个数 LLEN

返回存储在 key 里的list的长度。 如果 key 不存在，那么就被看作是空list，并且返回长度为 0。 当存储在 key 里的值不是一个list的话，会返回error。 

- 语法

  ```java
  llen key
  ```

- 示例

  ```java
  127.0.0.1:6379> lrange list1 0 -1
  1) "c"
  2) "d"
  3) "e"
  4) "f"
  5) "g"
  6) "h"
  
  //返回list1的长度6
  127.0.0.1:6379> llen list1
  (integer) 6
      
      
  127.0.0.1:6379> del list1
  (integer) 1
      
  //此时list1不存在，视为空，返回长度0
  127.0.0.1:6379> llen list1
  (integer) 0
  
      
  //对非list进行llen操作会报错
  127.0.0.1:6379> set key1 hello
  OK
  127.0.0.1:6379> llen key1
  (error) WRONGTYPE Operation against a key holding the wrong kind of value
     
  
  ```

##### 3.5 阻塞式列表弹出  BLPOP/BRPOP

BLPOP/BRPOP是阻塞式列表的弹出原语。 它是命令 LPOP/RPOP的阻塞版本，这是因为当给定列表内没有任何元素可供弹出的时候， 连接将被 BLPOP/BRPOP命令阻塞。 当给定多个 key 参数时，按参数 key 的先后顺序依次检查各个列表，弹出第一个非空列表的头/尾元素。 

BLPOP/BRPOP分为阻塞行为和非阻塞行为，

- 非阻塞行为

  当 BLPOP/BRPOP被调用时，如果给定 key 内至少有一个非空列表，那么弹出遇到的第一个非空列表的头元素，并和被弹出元素所属的列表的名字 key 一起，组成结果返回给调用者。

- 阻塞行为

  如果所有给定 key 都不存在或包含空列表，那么 BLPOP/BRPOP命令将阻塞连接， 直到有另一个客户端对给定的这些 key 的任意一个执行 LPUSH或 RPUSH命令为止。

  一旦有新的数据出现在其中一个列表里，那么这个命令会解除阻塞状态，并且返回 key 和弹出的元素值。

  当 BLPOP/BRPOP命令引起客户端阻塞并且设置了一个非零的超时参数 timeout 的时候， 若经过了指定的 timeout 仍没有出现一个针对某一特定 key 的 push 操作，则客户端会解除阻塞状态并且返回一个 nil 的多组合值(multi-bulk value)。

  **timeout 参数表示的是一个指定阻塞的最大秒数的整型值。**当 timeout 为 0 是表示阻塞时间无限制。

- 语法

  ```java
  BLPOP key [key ...] timeout
  ```

- 示例

  ```java
  //会话1
  127.0.0.1:6379> lpush list2  2 4 6 
  (integer) 3
  127.0.0.1:6379> lpush list3 3 5 7
  (integer) 3
  
  //此时list1不存在，list2和list3非空
  127.0.0.1:6379> blpop list1 list2 list3 0
  1) "list2"
  2) "6"
  127.0.0.1:6379> blpop list1 list2 list3 0
  1) "list2"
  2) "4"
  127.0.0.1:6379> blpop list1 list2 list3 0
  1) "list2"
  2) "2"
  
  //此时list2全部弹出，则从list3开始
  127.0.0.1:6379> blpop list1 list2 list3 0
  1) "list3"
  2) "7"
  127.0.0.1:6379> blpop list1 list2 list3 0
  1) "list3"
  2) "5"
  127.0.0.1:6379> blpop list1 list2 list3 0
  1) "list3"
  2) "3"
  
  //此时list1 list2 list3 都没有值，进入阻塞状态
  127.0.0.1:6379> blpop list1 list2 list3 0
  
  //会话2
  //复制会话，在另一个窗口操作，对list1进行push操作
  127.0.0.1:6379> lpush list1 l1
  (integer) 1
  
  //会话1
  //此时因为会话2 对list1进行了push操作，此时会话1解除阻塞状态
  1) "list1"
  2) "l1"
  (43.17s)
  
  
  ```

#### List的常用案例

ist可被用来实现聊天系统。还可以作为不同进程间传递消息的队列。

你可以每次都以原先添加的顺序访问数据。这不需要任何SQL ORDER BY 操作，将会非常快，也会很容易扩展到百万级别元素的规模。

例如在评级系统中，比如社会化新闻网站 reddit.com，你可以把每个新提交的链接添加到一个list，用LRANGE可简单的对结果分页。

在博客引擎实现中，你可为每篇日志设置一个list，在该list中推入博客评论，等等。

### 4、Redis Set

 Redis集合是一个无序的字符串合集。你可以以**O(1)** 的时间复杂度（无论集合中有多少元素时间复杂度都为常量）完成 添加，删除以及测试元素是否存在的操作。

set集合有着不允许相同成员存在的优秀特性。向集合中多次添加同一元素，在集合中最终只会存在一个此元素。实际上这就意味着，在添加元素前，你并不需要事先进行检验此元素是否已经存在的操作。

一个Redis set它们支持一些服务端的命令从现有的集合出发去进行集合运算。 所以你可以在很短的时间内完成合并（union）,求交(intersection), 找出不同元素的操作。

一个集合最多可以包含(2^32)-1个元素（4294967295，每个集合超过40亿个元素）。

你可以用Redis集合做很多有趣的事，例如你可以：

- 用集合跟踪一个独特的事。想要知道所有访问某个博客文章的独立IP？只要每次都用SADD来处理一个页面访问。那么你可以肯定重复的IP是不会插入的。
- 使用SPOP或者SRANDMEMBER命令随机地获取元素。

#### 命令

##### 4.1 添加/删除元素 SADD/SREM

- SADD

  添加一个或多个指定的member元素到集合的 key中.指定的一个或者多个元素member 如果已经在集合key中存在则忽略.如果集合key 不存在，则新建集合key,并添加member元素到集合key中.

  返回新成功添加到集合里元素的数量，不包括已经存在于集合中的元素.如果key 的类型不是集合则返回错误.

  - 语法

    ```java
    SADD key member [member ...]
    ```

  - 示例

    ```java
    127.0.0.1:6379> sadd myset 1 2 3
    (integer) 3
    127.0.0.1:6379> smembers myset
    1) "1"
    2) "2"
    3) "3"
    127.0.0.1:6379> sadd myset 1
    (integer) 0
    127.0.0.1:6379> smembers myset
    1) "1"
    2) "2"
    3) "3"
    ```

- SREM

  在key集合中移除指定的元素. 如果指定的元素不是key集合中的元素则忽略 如果key集合不存在则被视为一个空的集合，该命令返回0.

  返回从集合中移除元素的个数，不包括不存在的成员.如果key的类型不是一个集合,则返回错误.

  - 语法

    ```java
    SREM key member [member ...]
    ```

  - 示例

    ```java
    127.0.0.1:6379> srem myset 1
    (integer) 1
    127.0.0.1:6379> srem myset 4
    (integer) 0
    127.0.0.1:6379> smembers myset
    1) "2"
    2) "3"
    
    ```

##### 4.2  遍历 smembers

​	返回key集合所有的元素.

​    该命令的作用与使用一个参数的SINTER命令作用相同.

- 语法

  ```java
  SMEMBERS key
  ```

- 示例

  ```java
  127.0.0.1:6379> sadd myset 1 2 3
  (integer) 3
  127.0.0.1:6379> smembers myset
  1) "1"
  2) "2"
  3) "3"
  ```

##### 4.3 交集 sinter

​	 返回指定所有的集合的成员的交集（结果集成员的列表）. 

 	如果key不存在则被认为是一个空的集合,当给定的集合为空的时候,结果也为空.(一个集合为空，结果一直为空). 

- 语法

  ```java
  SINTER key [key ...]
  ```

- 示例

  ```java
  127.0.0.1:6379> sadd myset1 1 3 4 
  (integer) 3
  127.0.0.1:6379> sadd myset2 2 3 4 
  (integer) 3
  
  //返回myset1和myset2的交集（3，4）
  127.0.0.1:6379> sinter myset1 myset2
  1) "3"
  2) "4"
  
  //当sinter的参数为一个时，相当于smembers，遍历
  127.0.0.1:6379> sinter myset1
  1) "1"
  2) "3"
  3) "4"
  
  //myset3集合为空，此时结果为空
  127.0.0.1:6379> sinter myset1 myset2 myset3
  (empty list or set)
  
  ```

##### 4.4 并集 sunion

​	返回给定的多个集合的并集中的所有成员.

​	不存在的key可以认为是空的集合.

- 语法

  ```java
  SUNION key [key ...]
  ```

- 示例

  ```java
  127.0.0.1:6379> smembers myset1
  1) "1"
  2) "3"
  3) "4"
  127.0.0.1:6379> smembers myset2
  1) "2"
  2) "3"
  3) "4"
  127.0.0.1:6379> sunion myset1 myset2
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  
  //单个key的时候，相当于遍历
  127.0.0.1:6379> sunion myset1 
  1) "1"
  2) "3"
  3) "4"
  
  //myset3为空，结果为myset1和myset2的并集
  127.0.0.1:6379> sunion myset1 myset2 myset3
  1) "1"
  2) "2"
  3) "3"
  4) "4"
  
  ```

##### 4.5 删除并获取 spop

​	从存储在key的集合中移除并返回一个或多个随机元素。

​	此操作与SRANDMEMBER类似，它从一个集合中返回一个或多个随机元素，但不删除元素。

​	返回被删除的元素，当key不存在时返回nil。

- 语法

  ```java
  SPOP key [count]
  ```

  

- 示例

  ```java
  127.0.0.1:6379> smembers myset1 
  1) "1"
  2) "3"
  3) "4"
  127.0.0.1:6379> spop myset1
  "4"
  127.0.0.1:6379> spop myset1
  "3"
  127.0.0.1:6379> spop myset1
  "1"
  
  //集合为空的时候返回nil
  127.0.0.1:6379> spop myset1
  (nil)
  
  127.0.0.1:6379> smembers  myset2
  1) "2"
  2) "3"
  3) "4"
  
  //从集合中移除并返回多个元素
  127.0.0.1:6379> spop myset2 3
  1) "2"
  2) "3"
  3) "4"
  127.0.0.1:6379> spop myset2 
  (nil)
  
  ```

### 5、Redis Sorted set

​	排序集是一种数据类型，类似于集合和哈希之间的混合。像集合一样，排序集合由唯一的，非重复的字符串元         

​	素组成，因此从某种意义上说，排序集合也是一个集合。

​	它们的差别是，每个有序集合的成员都关联着一个评分，用于把有序集合中的成员按最低分到最高分排列。

​	此外，已排序集合中的元素是按顺序进行的。它们按照以下规则排序：

​	数据结构的特殊性）。它们按照以下规则排序：

​			如果A和B是两个分数不同的元素，

​			如果A.score是> B.score，则A>B。

​			如果A和B的分数完全相同，则A字符串在字典上大于B字符串，则A>B。 A和B字符串不能相等，因为排序			后的集合只有唯一的元素，即相同分数的成员按照字典规则相对排序 。

​	使用有序集合，你可以非常快地（**O(log(N))**）完成添加，删除和更新元素的操作。 因为元素是在插入时就排好

​	序的，所以很快地通过评分(score)或者 位次(position)获得一个范围的元素。 访问有序集合的中间元素同样也

​	是非常快的，因此你可以使用有序集合作为一个没用重复成员的智能列表。 在这个列表中， 你可以轻易地访问

​	任何你需要的东西: 有序的元素，快速的存在性测试，快速访问集合中间元素！

​	简而言之，使用有序集合你可以很好地完成很多在其他数据库中难以实现的任务。

​	比如：

- 在一个巨型在线游戏中建立一个排行榜，每当有新的记录产生时，使用ZADD来更新它。你可以用ZRANGE轻

  松地获取排名靠前的用户， 你也可以提供一个用户名，然后用ZRANK获取他在排行榜中的名次。 同时使用

  ZRANK和ZRANGE你可以获得与指定用户有相同分数的用户名单。 所有这些操作都非常迅速。

- 有序集合通常用来索引存储在Redis中的数据。 例如：如果你有很多的hash来表示用户，那么你可以使用一个

  有序集合，这个集合的年龄字段用来当作评分，用户ID当作值。用ZRANGEBYSCORE可以简单快速地检索到

  给定年龄段的所有用户。

#### 命令

##### 5.1 添加元素 zadd

​	将所有指定成员添加到键为key的有序集合（sorted set）里面。 添加时可以指定多个分数/成员（score/member）对。 如果指定添加的成员已经是有序集合里面的成员，则会更新改成员的分数（scrore）并	更新到正确的排序位置。

​	如果key不存在，将会创建一个新的有序集合（sorted set）并将分数/成员（score/member）对添加到有序集	合，就像原来存在一个空的有序集合一样。如果key存在，但是类型不是有序集合，将会返回一个错误应答。

​	分数值是一个双精度的浮点型数字字符串。+inf和inf都是有效值。

- 语法

  ```java
  ZADD key [NX|XX] [CH] [INCR] score member [score member ...]
  ```

  - **XX**: 仅仅更新存在的成员，不添加新成员。

  - **NX**: 不更新存在的成员。只添加新成员。

  - **CH**: 修改返回值为发生变化的成员总数，原始是返回新添加成员的总数 (CH 是 *changed* 的意思)。更改的元素是**新添加的成员**，已经存在的成员**更新分数**。 所以在命令中指定的成员有相同的分数将不被计算在内。注：在通常情况下，ZADD返回值只计算新添加成员的数量。

  - **INCR**: 当ZADD指定这个选项时，成员的操作就等同ZINCRBY命令，对成员的分数进行递增操作。

    

- 示例

  ```java
  27.0.0.1:6379> zadd myzset1 1 a 2 b 3 c 4 d
  (integer) 4
  127.0.0.1:6379> zrange myzset1 0 -1
  1) "a"
  2) "b"
  3) "c"
  4) "d"
  
  //zrange获取集合中指定范围的元素，可以参考前面lrange的用法
  127.0.0.1:6379> zadd myzset2 1 a 2 b 1 c 2 d
  (integer) 4
  127.0.0.1:6379> zrange myzset2 0 -1
  1) "a"
  2) "c"
  3) "b"
  4) "d"
  
  //通过添加WITHSCORES，同时打印出成员的分数
  127.0.0.1:6379> zrange myzset2 0 -1 WITHSCORES
  1) "a"
  2) "1"
  3) "c"
  4) "1"
  5) "b"
  6) "2"
  7) "d"
  8) "2"
  
  
  ```

##### 5.2 zcount

​	获得指定分数范围内的元素个数 ， 返回有序集key中，score值在min和max之间(默认包括score值等于min或	max)的成员。 

- 语法

  ```java
  ZCOUNT key min max
  ```

- 示例

  ```java
  127.0.0.1:6379> zrange myzset1 0 -1 withscores
   1) "a"
   2) "1"
   3) "b"
   4) "2"
   5) "c"
   6) "3"
   7) "d"
   8) "4"
   9) "e"
  10) "5"
  11) "f"
  12) "6"
  13) "g"
  14) "7"
  127.0.0.1:6379> zcount myzset1 2 7
  (integer) 6
  127.0.0.1:6379> zcount myzset1 6 7
  (integer) 2
  
  ```

##### 5.3 获取元素排名 zrank/zrevrank

- zrank

  ​	返回有序集key中成员member的排名。其中有序集成员按score值递增(从小到大)顺序排列。排名以0为	  	底，也就是说，score值最小的成员排名为0。

  ​	使用ZREVRANK命令可以获得成员按score值递减(从大到小)排列的排名。

  ​	返回值：

  ​		如果member是有序集key的成员，返回integer-reply：member的排名。

  ​		如果member不是有序集key的成员，返回bulk-string-reply: nil。

  - 语法

    ```java
    ZRANK key member
    ```

  - 示例

    ```java
    127.0.0.1:6379> zrange myzset1 0 -1 withscores
     1) "a"
     2) "1"
     3) "b"
     4) "2"
     5) "c"
     6) "3"
     7) "d"
     8) "4"
     9) "e"
    10) "5"
    11) "f"
    12) "6"
    13) "g"
    14) "7"
    127.0.0.1:6379> zrank myzset1 c
    (integer) 2
    127.0.0.1:6379> zrank myzset1 a
    (integer) 0
    127.0.0.1:6379> zrank myzset1 g
    (integer) 6
    127.0.0.1:6379> zrank myzset1 m
    (nil)
    
    ```

- zrevrank

  ​	返回有序集key中成员member的排名，其中有序集成员按score值从大到小排列。排名以0为底，也就是	            	说，score值最大的成员排名为0。

  ​	返回值

  ​		如果member是有序集key的成员，返回integer-reply:member的排名。

  ​		如果member不是有序集key的成员，返回bulk-string-reply: nil。

  - 语法

    ```java
    ZREVRANK key member
    ```

  - 示例

    ```java
    127.0.0.1:6379> zrange myzset1 0 -1 withscores
     1) "a"
     2) "1"
     3) "b"
     4) "2"
     5) "c"
     6) "3"
     7) "d"
     8) "4"
     9) "e"
    10) "5"
    11) "f"
    12) "6"
    13) "g"
    14) "7"
    127.0.0.1:6379> zrevrank myzset1 c
    (integer) 4
    127.0.0.1:6379> zrevrank myzset1 a
    (integer) 6
    127.0.0.1:6379> zrevrank myzset1 g
    (integer) 0
    127.0.0.1:6379> zrevrank myzset1 m
    (nil)
    
    ```

##### 5.4 删除元素 zrem

​	当key存在，但是其不是有序集合类型，就返回一个错误。

​	返回的是从有序集合中删除的成员个数，不包括不存在的成员。

- 语法

  ```java
  ZREM key member [member ...]
  ```

- 示例

  ```java
  127.0.0.1:6379> zrange myzset1 0 -1 withscores
   1) "a"
   2) "1"
   3) "b"
   4) "2"
   5) "c"
   6) "3"
   7) "d"
   8) "4"
   9) "e"
  10) "5"
  11) "f"
  12) "6"
  13) "g"
  14) "7"
  
  127.0.0.1:6379> zrem  myzset1 a b c
  (integer) 3
  
  127.0.0.1:6379> zrange myzset1 0 -1 withscores
  1) "d"
  2) "4"
  3) "e"
  4) "5"
  5) "f"
  6) "6"
  7) "g"
  8) "7"
  
  ```

以上就是关于Redis 5种基本数据类型的常用API，在下一篇会介绍Redis内部存储细节以及Redis对象类型的内部编码，链接：null！

## 每日一皮

 我只能中午和你聊天，不是因为我冷漠我无情，而是因为我怕你早晚会爱上我！