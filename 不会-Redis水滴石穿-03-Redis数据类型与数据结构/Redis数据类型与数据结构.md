# redis学习笔记（三、数据类型与内存编码）

上一篇我们介绍了关于Redis 5种基本数据类型常用的API（redis学习笔记二（基本数据类型API）），今天我们来探索下关于Redis 的内部存储结构。

## 一、Redis 内存模型

使用缓存对提高系统性能有很多好处，但是不合理的使用缓存可能非但不能提高系统的性能，还会成为系统的累赘，甚至风险。
比如频繁修改的数据，这种数据如果缓存起来，由于频繁修改，应用还来不及读取就已经失效或更新，徒增系统负担。一般说来，数据的读写比在 2:1 以上，缓存才有意义。

而Redis是目前最火爆的内存数据库之一，通过在内存中读写数据，大大提高了读写速度，可以说Redis是实现网站高并发不可或缺的一部分。

我们在使用Redis时，最常接触到的5种对象类型：字符串、哈希、列表、集合、有序集合。丰富的类型是Redis相对于Memcached等的一大优势。在了解Redis 5种对象类型用法和特点的基础上，进一步了解Redis的内存模型，对Redis的使用会有很大帮助，比如：

- 估算Redis内存使用量，内存的使用成本仍然相对较高，使用内存不能无所顾忌，根据需求合理的评估Redis的内存使用量，选择合适的机器配置，可以在满足需求的情况下节约成本；
- 优化内存占用，了解Redis内存模型可以选择更合适的数据类型和编码，更好的利用Redis内存；
- 分析解决问题，当Redis出现阻塞、内存占用等问题时，可以快速发现导致问题的原因，便于分析解决问题。

本文主要介绍以Redis5.0为例的Redis的内存模型，包括：Redis占用内存的情况、内存划分、内存分配器、简单动态字符串（SDS）、RedisObject、不同的对象类型在内存中的编码方式等。然后在此基础上分享几个Redis内存模型的应用。

### 1、Redis的内存统计

可以通过命令查看Redis使用内存的情况，在客户端通过redis-cli连接服务器后，通过info 命令可以查看内存使用情况：

![1599036753554](.\typora-user-images\1599036753554.png)

实际返回的信息有很多，这里只是截了其中一部分，我们只介绍返回结果中比较重要的几个信息：

-  **used_memory**

  ​	Redis分配器分配的内存总量（单位是字节），包括使用的虚拟内存；

  ​	used_memory_human将used_memory的信息显示的更加人性化些。

-  **used_memory_rss**

  ​	Redis进程占据操作系统的内存（单位是字节），与top及ps命令看到的值是一致的；除了分配器分配的内	存之外，used_memory_rss还包括进程运行本身需要的内存、内存碎片等，但是不包括虚拟内存。 

  ​	**used_memory和used_memory_rss的区别：**

  ​		used_memory是从Redis角度得到的量，而used_memory_rss是从操作系统角度得到的量。

  ​		二者之所以有所不同，一方面是因为内存碎片和Redis进程运行需要占用内存，使得前者可能比后者小；		另一方面虚拟内存的存在，使得前者可能比后者大。

  ​		在实际应用中，Redis的数据量会比较大，此时进程运行占用的内存与Redis数据量和内存碎片相 比，会		小得多；因此used_memory_rss和used_memory的比例，便形成了衡量Redis内存碎片率的参数 ——                              		mem_fragmentation_ratio。    		

- **mem_fragmentation_ratio**

  ​	内存碎片比率，该值是used_memory_rss / used_memory的比值。

  ​	mem_fragmentation_ratio一般大于1，且该值越大，说明内存碎片比例越大。

  ​	如果mem_fragmentation_ratio<1，说明Redis使用了虚拟内存，由于虚拟内存的媒介是磁盘，比内存速度	要慢很多，当这种情况出现时，应该及时排查，如果内存不足应该及时处理。

  ​	在服务刚启动时，因为还没有向Redis中存入数据，Redis 进程本身运行的内存使得used_memory_rss 比    	used_memory大得多，所以mem_fragmentation_ratioRedis的值可能会很大。	

  ​	另外如果使用的Redis是windows版本的话，这时需要注意下使用的内存分配器以及Redis的版本，因为服	务刚启动的时候该值可能小于1，原因是它认为启动程序是不该占用真正的内存的，所以会使用虚拟内存，	把真正的内存留给数据。

-  **mem_allocator**

  ​	Redis使用的内存分配器，在编译时指定，可以是 libc 、jemalloc或者tcmalloc，默认是jemalloc。

```java
小贴士：
    info命令可以显示redis服务器的许多信息，包括服务器基本信息、CPU、内存、持久化、客户端连接信息等；
    memory是参数，表示只显示内存相关的信息。
```

### 2、Redis的内存划分

Redis作为内存数据库，在内存中存储的内容主要是数据（键值对）。

Redis的内存占用主要可以划分为以下几个部分：

- 数据

  ​	对数据库来说，数据是最主要的部分，数据占用的内存会统计在used_memory中。

  ​	Redis使用键值对存储数据，包括5种类型：字符串、哈希、列表、集合、有序集合。

  ​	这5种类型是Redis对外提供的。实际上，Redis内部，每种类型可能有2种或更多的内部编码实现。此外，	Redis在存储对象时，并不是直接将数据扔进内存，而是会对对象进行各种包装：如RedisObject、SDS等。

- 进程

  ​	Redis进程本身运行也需要占用一定的内存，大约几兆，比如代码、常量池等。

  ​	在实际生产环境中与Redis数据占用的内存相比可以忽略不计了。这部分内存不是由jemalloc分配，所以不	会计算在used_memory当中。

   

  ```java
  小贴士：
  	Redis创建的子进程运行也会占用内存，比如Redis执行RDB、AOF重写时创建的子进程。
      这部分内存不属于Redis进程，所以也不会统计在used_memory中。
  
  ```

- 缓冲内存

  ​	缓冲内存包括客户端缓冲区、复制积压缓冲区、AOF 缓冲区等，其中，

  - ​	客户端缓冲区存储客户端连接的输入输出缓冲；

  - ​	复制积压缓冲区用于部分复制功能；
  - ​    AOF 缓冲区用于在进行 AOF 重写时，保存最近的写入命令。

  ​    这部分内存由 jemalloc 分配，因此会统计在 used_memory 中。

- 内存碎片

  ​	内存碎片是Redis在分配、回收物理内存过程中产生的，内存碎片的产生与对数据进行的操作、数据的特点	以及使用的内存分配器等都有关系。例如，如果对数据进行频繁修改，数据之间的大小存在差异，可能会导	致Redis释放的空间在物理内存中并没有释放，但Redis又无法有效利用，这就形成了内存碎片。内存碎片也	不会统计在used_memory中。

   

  ```java
  小提示：
  	如果Redis服务器中的内存碎片已经很大，可以通过安全重启的方式减小内存碎片。
  	因为重启之后，Redis从备份文件中恢复数据，在内存中进行重排，为每个数据重新选择合适的内存单元，减小	   内存碎片。
  	
  	另外如果内存分配器设计合理，也可以减少内存碎片的产生，jemalloc在控制内存碎片方面做的就很好。
  ```

## 二、Reids 内部存储细节

在正式学习Redis的对象类型与内存编码之前，我们需要先了解一些概念和存储关系，比如Redis的K和V是如何存储的、 我们常提到的内存分配器jemalloc、简单动态字符串（SDS）、RedisObject等。

这里有一个非常经典的图（一定要记住它），简单的描述了在执行set hello world时，所涉及到的数据模型，下面我们结合这个图简单的介绍下， 

![1599436274487](.\typora-user-images\1599436274487.png)

**（1）dictEntry：** 首先Redis是Key-Value数据库，每个键值对都会有一个dictEntry进行存储，里面包含指向Key和Value的指针；next（hash冲突的时候才会用到）指向下一个dictEntry，与本Key-Value无关。

 

**（2）Key：** Key（“hello”）并不是直接以字符串的形式存储，而是存储在SDS结构中。

 

**（3）redisObject：**可以看到Value(“world”)既不是直接以字符串存储，也不是像Key一样直接存储在SDS中，而是存储在redisObject中。实际上，不论Value是5种类型的哪一种，都是通过RedisObject来存储的；其中type字段指明了Value对象的类型，ptr字段则指向对象所在的地址。具体的值是通过SDS进行存储。

经过上面的分析，我们对K，V的存储已经有了初步的认识，下面我们详细讨论下其中比较关键的redisObject、使用的内存分配器、SDS。

-  **jemalloc** 

  无论是DictEntry对象，还是RedisObject、SDS对象等，存储的时候都需要内存分配器分配内存。Redis在编译时便会指定内存分配器；内存分配器可以是 libc 、jemalloc或者tcmalloc，jemalloc作为Redis的默认内存分配器，在减小内存碎片方面做的相对较好。

  jemalloc在64位系统中，将内存空间划分为小、大、巨大三个范围；每个范围内又划分了许多小的内存块单位；当Redis存储数据时，会选择大小最合适的内存块进行存储。

  以DictEntry对象为例，有3个指针组成，在64位机器下占24个字节，jemalloc会为它分配32字节大小的内存单元。 

  jemalloc划分的内存单元如图所示：

  ![1599437000592](.\typora-user-images\1599437000592.png)

​	   

​	    例如，如果需要存储大小为130字节的对象，jemalloc会将其放入160字节的内存单元中。 

- **redisObject** 
  上面提到过关于Redis有5种数据类型，无论是哪种类型，都不会直接存储，而是通过redisObject对象进行包装， 一个redisObject对象的大小为16字节 。
  redisObject对象非常重要，Redis对象的类型、内部编码、内存回收、共享对象等功能，都需要 redisObject支持，下面是关于redisObject结构体的定义，Redis中的每个对象都是由如下结构表示，我们来详细的分析一下，看看它是如何作用的，

  ```c
  {    
      unsigned type:4;//类型，即五种对象类型    
      unsigned encoding:4;//编码    
      void *ptr;//指向底层实现数据结构的指针      
      int refcount;//引用计数      
      unsigned lru:24;//记录最后一次被命令程序访问的时间    
      //... 
  }robj;
  ```

  - **type**

    ​	type字段表示对象的类型，占4个bit；即Redis的五种数据类型，STRING(字符串)、LIST (列表)、 	        	HASH(哈希)、SET(集合)、ZSET(有序集合)。
    ​    我们可以通过type命令获取对象的类型，其内部便是通过redisObject的type属性获取的，示例如图，

    ![1599438201785](.\typora-user-images\1599438201785.png)

    

  - **encoding** 

    ​	encoding表示对象内部使用的编码，占4个bit。 对于Redis支持的每种类型，都有至少两种内部编码。

    ​	比如字符串，就有int、embstr、raw三种编码。通过encoding属性，可以实现根据不同的场景来为对	象设置不同的编码，大大提高了Redis 的灵活性和效率。以列表对象为例，有压缩列表和双向链表两种	编码方式；如果列表中的元素较少， Redis倾向于使用压缩列表进行存储，因为压缩列表占用的内存更	少，而且可以比双向链表更快载入；当列表对象元素较多时，压缩列表就会转化为更适合存储大量元素	的双向链表。
    ​	通过object encoding命令，可以查看对象采用的编码方式，示例如图：

    ​		![1599438905390](.\typora-user-images\1599438905390.png)

  - **ptr** 

    ​	ptr指针指向具体的数据，如前面的例子，set hello world，ptr指向包含字符串world的SDS。

  - **refcount** 

    ​	refcount记录该对象被引用的次数，主要用于对象的引用计数和内存回收。类型为整型，描述如下，

    ​		创建新对象时，refcount初始化为1；

    ​		当有程序使用该对象时，refcount加1；

    ​		当对象不再被一个程序使用时，refcount减1；

    ​		当refcount变为0时，对象占用的内存会被释放。

    ​	**共享对象**

    ​		Redis为了节省内存，当有一些对象重复出现时，新的程序不会创建新的对象，而是仍然使用原来的		对象。这个被重复使用的对象即refcount>1时，称为共享对象。 

    ​		共享对象池是指Redis内部维护[0-9999]的整数对象池。由于创建大量的整数类型redisObject存在内		存开销，每个redisObject内部结构至少占16字节，甚至超过了整数自身空间消耗。所以Redis内存维		护一个[0-9999]的整数对象池，用于节约内存。

    ​		目前共享对象仅支持整数值的字符串对象，因为共享对象虽然会降低内存消耗，但是判断两个对象是		否相等却需要消耗额外的时间，
    ​			对于整数值， 判断操作复杂度为O(1)；

    ​			对于普通字符串，判断复杂度为O(n)；

    ​			而对于哈希、列表、集合和有序集合， 判断的复杂度为O(n^2)。 

    ​		虽然共享对象只能是整数值的字符串对象，但是5种类型都可能会使用到共享对象，比如list、hash、		set、zset内部元素也可以使用整数对象池。因此开发中在满足需求的前提下，尽量使用整数对象以		节省内存。

    ​		Redis服务器在初始化时，会创建10000个字符串对象，值分别是0~9999的整数值；当Redis需要使		用值为0~9999的字符串对象时，可以直接使用这些共享对象，通过object refcount查看时，能够返		回指定key所对应的value被引用的次数，在0-9999之间的整数，都是共享内存的，所以返回值是同		一个数“2147483647”，示例如图，

    ​			![1599441962862](.\typora-user-images\1599441962862.png)

  - **lru**

    ​	lru记录的是对象最后一次被程序访问的时间，占据的比特数不同的版本有所不同，4.0版本占24bit。
    ​	通过对比lru时间与当前时间，可以计算某个对象的闲置时间；object idletime该命令返回指定key对应	的value自被存储之后空闲的时间，以秒为单位(没有读写操作的请求) ，示例如图，

    ​			![1599442791807](.\typora-user-images\1599442791807.png)

    ​	

    ```java
    小贴士：
    	object idletime命令的一个特殊之处在于它不改变对象的lru值。
    	lru值除了通过object idletime命令打印之外，还与Redis的内存回收有关系，如果Redis打开了		maxmemory选项，且内存回收算法选择的是volatile-lru或allkeys—lru，那么当Redis内存占用超过	  maxmemory指定的值时，Redis会优先选择空转时间最长的对象进行释放。
    ```

-  **SDS**

  ​	 Redis没有采用原生C语言的字符串类型而是自己实现了字符串结构，简单动态字符串（simple dynamic 	    	 string，SDS）。 

  ​	Redis在存储对象时，一律使用SDS代替C字符串。比如set hello world命令，hello和world都是以SDS的形	式存储的。只有在字符串不会改变的情况下，如打印日志时，才会使用C字符串。

  - **3.2 之前**

    ```c
    struct sdshdr{    
        //记录buf数组中已使用字节的数量，即SDS中保存字符串的长度  
    	int len;    
        //记录 buf 数组中未使用字节的数量    
        int free;    
        //字节数组，用于保存字符串    
        char buf[]; 
    }
    ```

  - **3.2之后**

    ​	

    ```c
    typedef char *sds;      
    struct __attribute__ ((__packed__)) sdshdr5 {    // 对应的字符串长度小于 1<<5 32字节   
          
        unsigned char flags; // 高5位表示字符串长度，低三位表示类型  
        char buf[]; 
    };
    
    struct __attribute__ ((__packed__)) sdshdr8 {     // 对应的字符串长度小于 1<<8 256 
           
        uint8_t len;   // used，目前字符串的长度 用1字节存储                        
        uint8_t alloc; //已经分配的总长度 用1字节存储    
        unsigned char flags; //flag用3bit来标明类型，其余5bit目前没有使用  
        char buf[];        //数组，以'\0'结尾 
    }; 
    struct __attribute__ ((__packed__)) sdshdr16 {    // 对应的字符串长度小于 1<<16 
           
        uint16_t len; //已使用长度，用2字节存储    
        uint16_t alloc; // 总长度，用2字节存储    
        unsigned char flags; // 3 lsb of type, 5 unused bits
        char buf[]; 
    }; 
    struct __attribute__ ((__packed__)) sdshdr32 {    // 对应的字符串长度小于 1<<32
           
        uint32_t len; /*已使用长度，用4字节存储*/   
        uint32_t alloc; /* 总长度，用4字节存储*/   
        unsigned char flags;// 3 lsb of type, 5 unused bits    
        char buf[]; 
    }; 
    struct __attribute__ ((__packed__)) sdshdr64 {   // 对应的字符串长度小于 1<<64 
           
        uint64_t len; //已使用长度，用8字节存储   
        uint64_t alloc; // 总长度，用8字节存储   
        unsigned char flags; // 3 lsb of type, 5 unused bits   
        char buf[];
    };
    ```

  - **SDS与C字符串的比较** 
    **获取字符串长度： **

    ​	SDS是O(1)，可以很快的得到字符串长度、已用长度、未用长度 ，C字符串是O(n) 。

    **缓冲区溢出：**

    ​	使用C字符串时，如果字符串长度增加，而忘记重新分配内存， 很容易造成缓冲区的溢出；而SDS由于	记录了长度，相应的API在可能造成缓冲区溢出时会自动重新分配内存，避免了缓冲区溢出问题。 

    **修改字符串时内存的重分配：**

    ​	对于C字符串，如果要修改字符串，必须要重新分配内存，先释放再申请，如果没有重新分配，字符串	长度增大时会造成内存缓冲区溢出，字符串长度减小时会造成内存泄露。而对于SDS，由于可以记录	   	len和free，因此解除了字符串长度和空间数组长度之间的关联，可以在此基础上进行优化：

    ​		空间预分配策略，即分配内存时比实际需要的多，使得字符串长度增大时重新分配内存的概率减小；

    ​		惰性空间释放策略，使得字符串长度减小时重新分配内存的概率减小。 

    **存取二进制数据：**

    ​	SDS可以，C字符串不可以。因为C字符串以空字符作为字符串结束的标识，而对于一些二进制文件如图	片等，内容可能包括空字符串，因此C字符串无法正确存取；而SDS 以字符串长度len来作为字符串结束	标识，因此没有这个问题。 

    ​	此外，由于SDS中的buf仍然使用了C字符串即以’\0’结尾，因此SDS可以使用C字符串库中的部分函数；	但是需要注意的是，只有当SDS用来存储文本数据时才可以这样使用，在存储二进制数据时则不行，因	为不一定是以’\0’结尾。

## 三、Redis对象类型与内存编码

前面提到过，Redis支持5种对象类型，每种类型都有至少两种编码。这样做的好处是可以根据不同的应用场景切换内部编码，提高效率。

Redis各种对象类型支持的内部编码如图所示：

![1599460482622](.\typora-user-images\1599460482622.png)

### 4.1、字符串

字符串是最基础的类型，因为所有的键都是字符串类型，且字符串之外的其他几种复杂类型的元素也是字符串。

**需要注意的是字符串长度不能超过512MB。**

从内部编码图可以看出，字符串类型的内部编码有3种，它们的应用场景如下：
	int：8个字节的长整型。字符串值是整型时，这个值使用long整型表示；

​	embstr：<=44字节的字符串；

​	raw：大于44个字节的字符串 

embstr与raw都使用redisObject和sds保存数据，区别在于， embstr的使用只分配一次内存空间，因此redisObject和sds是连续的；而raw需要分配两次内存空间，分别为redisObject和sds分配空间。

因此与raw相比，embstr的好处在于创建时少分配一次空间，删除时少释放一次空间，以及对象的所有数据连在一起，寻找方便。而embstr的坏处也很明显，如果字符串的长度增加需要重新分配内存时，整个redisObject和sds都需要重新分配空间，因此redis中的embstr实现为只读。

需要注意的是，由于embstr是只读的， 因此在对embstr对象进行修改时，都会先转化为raw再进行修改，所以，只要是修改embstr对象，修改后的对象一定是raw的，无论是否达到了44个字节，示例如图：

​	![1599462182363](.\typora-user-images\1599462182363.png)	

object encodin命令返回指定key对应value所使用的内部编码，可以看到同一个key针对不同的val，它使用的编码是动态变换的。



```java
小贴士：
	3.2之后 embstr和raw进行区分的长度，是44；
    	是因为redisObject的长度是16字节，sds的长度是4+字符串长度；因此当字符串长度是44时，embstr的长		   度正好是16+4+44 =64，jemalloc正好可以分配64字节的内存单元。
    
	3.2 之前embstr和raw进行区分的长度，是39；
    	因为redisObject的长度是16字节，sds的长度是9+字符串长度；因此当字符串长度是39时，embstr的长度		   正好是16+9+39 =64，jemalloc正好可 以分配64字节的内存单元
```

### 4.2、列表 

Redis3.0之前列表的内部编码可以是压缩列表（ziplist）或双端链表（linkedlist）。选择的折中方案是两种数据类型的转换，但是在3.2版本之后因为转换也是个费时且复杂的操作，所以引入了一种新的数据格式，结合了双向列表linkedlist和ziplist的特点，称之为quicklist。所有的节点都用quicklist存储，省去了到临界条件时的格式转换，支持两端插入和弹出，并可以获得指定位置（或范围）的元素，可以充当数组、队列、 栈等。

- **双向链表（了解）**

  由一个list结构和多个listNode结构组成，典型结构如下图所示：

   

  ![img](http://dbaplus.cn/uploadfile/2018/0716/20180716094216136.jpg)

  ​									

  通过图中可以看出，双向链表同时保存了头指针和尾指针，并且每个节点都有指向前和指向后的指针。链表中保存了列表的长度，dup、free和match为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。而链表中每个节点指向的是type为字符串的RedisObject。

- **压缩列表（了解）**

  压缩列表（ziplist）是Redis为了节省内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型数据结构，一个压缩列表可以包含任意多个节点，每个节点可以保存一个字节数组或者一个整数值，放到一个连续内存区。

  当一个列表只包含少量列表项时，并且每个列表项是小整数值或短字符串，那么Redis会使用压缩列表来做该列表的底层实现。

  与双端链表相比，压缩列表可以节省内存空间，但是进行修改或增删操作时，复杂度较高，因此当节点数量较少时，可以使用压缩列表。但是节点数量多时，还是使用双向链表划算。

  压缩列表不仅用于实现列表，也用于实现哈希、有序列表，使用非常广泛。

  需要注意的是只有同时满足下面两个条件时，才会使用压缩列表：
  		1）、列表中元素数量小于512个； 
  		2）、列表中所有键值对的键和值字符串长度都小于64字节。
  如果有一个条件不满足，则使用双向列表，且编码只可能由压缩列表转化为双向链表，反方向则不可能。

- **快速列表（重要）**

  简单的说，我们仍旧可以将其看作是一个双向列表，但是列表的每个节点都是一个ziplist，其实就是 linkedlist和ziplist的结合。quicklist中的每个节点ziplist都能够存储多个数据元素。 Redis3.2开始，列表采用quicklist进行编码，如图所示，

  ![1599464885102](.\typora-user-images\1599464885102.png)

  ​	![1599465660814](.\typora-user-images\1599465660814.png)

  

  源码解析如下，

  ```c
  //32byte 的空间 
  typedef struct quicklist {    
      // 指向quicklist的头部    
      quicklistNode *head;  
       // 指向quicklist的尾部    
      quicklistNode *tail;    
      // 列表中所有数据项的个数总和    
      unsigned long count;      
      // quicklist节点的个数，即ziplist的个数    
      unsigned int len;     
      // ziplist大小限定，由list-max-ziplist-size给定    
      // 表示不用整个int存储fill，而是只用了其中的16位来存储    
      int fill : 16;       
      // 节点压缩深度设置，由list-compress-depth给定    
      unsigned int compress : 16; 
  } quicklist;
  
  typedef struct quicklistNode {  
      // 指向上一个ziplist节点 
      struct quicklistNode *prev;  
      // 指向下一个ziplist节点   
      struct quicklistNode *next;  
      // 数据指针，如果没有被压缩，就指向ziplist结构， 反之指向quicklistLZF结构    
      unsigned char *zl;           
      // 表示指向ziplist结构的总长度(内存占用长度)    
      unsigned int sz;             
      // 表示ziplist中的数据项个数   
      unsigned int count : 16;     
      // 编码方式，1--ziplist，2--quicklistLZF    
      unsigned int encoding : 2;   
      // 预留字段，存放数据的方式，1--NONE，2-ziplist    
      unsigned int container : 2;  
      //解压标记，当查看一个被压缩的数据时，需要暂时解 压，标记此参数为1，之后再重新进行压缩    
      unsigned int recompress : 1; 
      // 测试相关    	
      unsigned int attempted_compress : 1;
      // 扩展字段，暂时没用
      unsigned int extra : 10; 
       
  } quicklistNode;
  ```

  

### 4.3、哈希

哈希，不仅是redis对外提供的5种对象类型的一种，也是Redis所使用的数据结构。为了便于表达，后面使用“内层哈希”代表redis5种对象类型的一种；使用“外层哈希”代表Redis作为Key-Value数据库所使用的数据结构。

内层哈希使用的内部编码可以是压缩列表（ziplist）和哈希表（hashtable）两种；  ziplist前面已介绍，用于元素个数少、元素长度小的场景；其优势在于集中存储，节省空间；虽然对元素的操作复杂度由O(1)变为了O(n)，但由于元素数量较少，因此操作的时间并没有明显劣势。

需要注意的是，只有同时满足下面两个条件时，才会使用ziplist，

- 哈希中元素数量小于512个；
- 哈希中所有键值对的键和值字符串长度都小于64字节。

如果有一个条件不满足，则使用hashtable；且编码只可能由ziplist转化为hashtable，反方向则不可能，如图所示，

​	![1599468894765](.\typora-user-images\1599468894765.png)





Redis的外层哈希则只使用了hashtable。

hashtable：一个hashtable由1个dict结构、2个dictht结构、1个dictEntry指针数组（称为bucket）和 n个dictEntry结构组成。
正常情况下即hashtable没有进行rehash时，各部分关系如下图所示：

 ![1599466669434](.\typora-user-images\1599466669434.png)

下面我们依次来介绍下各部分的功能，

- **dict**

  一般来说，通过使用dictht和dictEntry结构，便可以实现普通哈希表的功能；但是Redis的实现中，在 dictht结构的上层，还有一个dict结构。下面说明dict结构的定义及作用，源码如下，

  ```c
  typedef struct dict{ 
      // type里面主要记录了一系列的函数,可以说是规定了一系列的接口 
      dictType *type;   
      // privdata保存了需要传递给那些类型特定函数的可选参数 
      void *privdata;      
      //两张哈希表，便于渐进式rehash          
      dictht ht[2];
      //rehash 索引，并没有rehash时，值为 -1 
      int trehashidx;    
      //目前正在运行的安全迭代器的数量    
      int iterators; 
  } dict;
  ```

  其中，type属性和privdata属性是为了适应不同类型的键值对，用于创建多态字典；

  ht属性和trehashidx属性则用于rehash，即当哈希表需要扩展或收缩时使用。ht是一个包含两个项的数组，每项都指向一个dictht结构，这也是Redis的哈希会有1个dict、2个dictht结构的原因。

  通常情况下，所有的数据都是存在放dict的ht[0]中，ht[1]只在rehash的时候使用。dict进行rehash操作的时候， 将ht[0]中的所有数据rehash到ht[1]中。然后将ht[1]赋值给ht[0]，并清空ht[1]。

  因此，Redis中的哈希之所以在dictht和dictEntry结构之外还有一个dict结构，一方面是为了适应不同类型的键值对，另一方面是为了rehash。

- **dictht**

  dictht结构定义如下，

  ```c
  typedef struct dictht{      
      //哈希表数组，每个元素都是一条链表    
      dictEntry **table;        
      //哈希表大小    
      unsigned long size;     
      // 哈希表大小掩码，用于计算索引值，总是等于 size - 1    
      unsigned long sizemask;    
      // 该哈希表已有节点的数量    
      unsigned long used; 
  }dictht;
  ```

  其中，各个属性的功能说明如下：
  	table属性是一个指针，指向bucket数组； 

  ​	size属性记录了哈希表的大小，即bucket的大小；

  ​	used记录了已使用的dictEntry的数量； 

  ​	sizemask属性的值总是为size-1，这个属性和哈希值一起决定一个键在table中存储的位置。

- **bucket**

  bucket是一个数组，数组的每个元素都是指向dictEntry结构的指针。

  bucket数组的大小计算规则，大于dictEntry的数量的最小的2^n。

  如1000个dictEntry，那么bucket大小为1024；如果有1500个dictEntry，则bucket大小为2048。

- **dictEntry**

  前面提到过，每个键值对都会有一个dictEntry进行存储（这时候你的脑海里应该浮现的是那张经典的图），它的结构定义如下，

  ```c
  typedef struct dictEntry{        
      void *key; //键       
      union{ 
          //值v的类型可以是以下三种类型               
          void *val;                
          uint64_tu64;                
          int64_ts64;        
      }v;         
      // 指向下个哈希表节点，形成链表    
      struct dictEntry *next; 
  }dictEntry;
  ```

  其中，各个属性的功能说明如下：
  	key：键值对中的键 

  ​	val：键值对中的值，使用union即共用体实现，存储的内容既可能是一个指向值的指针，也可能是64位整			 型，或无符号64位整型； 

  ​	next：指向下一个dictEntry，用于解决哈希冲突问题。

  在64位系统中，一个dictEntry对象占24字节，其中key/val/next各占8字节。



### 4.4、集合

集合的内部编码可以是整数集合（intset）或哈希表（hashtable）。哈希表前面已经讲过，不过需要注意的是，集合在使用哈希表时，值全部被置为null。

集合（set）与列表类似，都是用来保存多个字符串，但集合与列表有两点不同：

- 集合中的元素是无序 的，因此不能通过索引来操作元素；
- 集合中的元素不能有重复。

intset的结构定义如下，

```c
typedef struct intset{        
    uint32_t encoding;    // 编码方式     
    uint32_t length;      // 集合包含的元素数量     
    int8_t contents[];    // 保存元素的数组 
} intset;
```

其中，encoding代表contents中存储内容的类型，虽然contents（存储集合中的元素）是int8_t 类型，但实际上其存储的值是int16_t、int32_t或int64_t，具体的类型便是由encoding决定的； length表示元素个数。

intset适用于集合所有元素都是整数且集合元素数量较小的时候，与哈希表相比，它的优势在于集中存储，节省空间；同时，虽然对于元素的操作复杂度由O(1)变为了O(n)，但由于集合数量较少，因此操作的时间并没有明显劣势。

需要注意的是，只有同时满足下面两个条件时，集合才会使用intset，

- 集合中元素数量小于512个；
- 集合中所有元素都是整数值。

如果有一个条件不满足，则使用哈希表；且编码只可能由整数集合转化为哈希表，反方向则不可能，如图所示，

​	![1599470161962](.\typora-user-images\1599470161962.png)

下图展示了集合编码转换的特点：

### 4.5、有序集合

有序集合的内部编码可以是压缩列表（ziplist）或跳跃表（skiplist）。ziplist在列表和哈希中都有分析过，这里不再重复。
跳跃表是一种有序数据结构，通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。除了跳跃表，实现有序数据结构的另一种典型实现是平衡树；大多数情况下，跳跃表的效率可以和平衡树媲美，且跳跃表实现比平衡树简单的多，因此redis中选用跳跃表代替平衡树。

跳跃表支持平均O(logN)、最坏O(N)的复杂度进行节点查找，并支持顺序操作。 Redis的跳跃表实现由zskiplist和zskiplistNode两个结构组成：前者用于保存跳跃表信息如头结 点、尾节点、长度等，后者用于表示跳跃表节点。

只有同时满足下面两个条件时，才会使用压缩列表：

- 有序集合中元素数量小于128个；
- 有序集合中所有成员长度都不足64字节。

如果有一个条件不满足，则使用跳跃表；且编码只可能由压缩列表转化为跳跃表，反方向则不可能，如图所示，

![1599471199499](.\typora-user-images\1599471199499.png)

下图展示了有序集合编码转换的特点

跳跃表结构定义如下，

```c
typedef struct zskiplistNode {     
    //层     
    struct zskiplistLevel{           
        //前进指针 后边的节点           
        struct zskiplistNode *forward;           
        //跨度           
        unsigned int span;     
    }level[];
 
     //后退指针     
    struct zskiplistNode *backward;     
    //分值     double score;     
    //成员对象     robj *obj;
 
} zskiplistNode
    
//链表 
typedef struct zskiplist{     
    //表头节点和表尾节点     
    structz skiplistNode *header, *tail;     
    //表中节点的数量     
    unsigned long length;     
    //表中层数最大的节点的层数     
    int level;
 
}zskiplist;
```

关于跳跃表的总结：

​	1）搜索：

​		从最高层的链表节点开始，如果比当前节点要大和比当前层的下一个节点要小，那么则往下找，也就是和当		前层的下一层的节点的下一个节点进行比较，以此类推，一直找到最底层的最后一个节点，如果找到则返		回，反之则返回空。
2）插入：

​		首先确定插入的层数，有一种方法是假设抛硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的		次数作为插入的层数。当确定插入的层数k后，则需要将新元素插入到从底层到k层。
3）删除：

​		在各个层中找到包含指定值的节点，然后将节点从链表中删除即可，如果删除以后只剩下头尾两个节点，则		删除这一层。

## 参考文献

- 《Redis开发与运维》
- 《Redis设计与实现》
-  http://www.redis.cn/commands.html 
-  http://zhangtielei.com/posts/blog-redis-robj.html 
-  https://dbaplus.cn/news-158-2127.html 

# 每日一皮

在野外遇到熊时，不要有过大的动作，这时要慢慢的下蹲把头低下，这样能确保在救援来的时候辨认死者的相貌。 