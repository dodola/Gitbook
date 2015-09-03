
#[微信ANDROID客户端-会话速度提升70%的背后][reflink]

转载自：WeMobileDev



#背景
*   打开会话速度慢

*   在同一个会话有较多的历史消息下，各种查询，更新,删除等操作，速度明显下降。

*   在会话内有较大量历史消息情况下，进入速度/刷新速度明显降低。

## 分析阶段

整个优化我们分2个阶段进行：

**第一阶段,针对历史记录较小的会话**

通过Android自带的trace工具分析，我们发现较大的耗时分布在进入会话的几个关键点：

*   在打开会话过程中涉及的磁盘读写操作

*   加载会话UI所执行的inflate操作（inflate指的是创建View对象）

*   退出会话后，列表控件的数据适配器被重置，触发清空列表控件的View（视图）缓存，再次进入需要重新创建此前已经创建过的view控件

*   系统切换 Activity（界面） 耗时

针对第一个问题，我们通过Android SDK 自带的systrace工具查找出所有写操作，把所有数据库或普通文件写操作任务提交到独立的后台线程执行，针对数据库读操作，我们通过sqlite自带的 explain query plan 指令，优化了该过程中SQL查询效率不够高的一些语句。

对于第2～4个问题，我们考虑到会话内UI控件这部分内存在实际使用过程中被重复命中的机会很大，采取合适的cache可以降低这里的每次进入时候的加载时间，故对View采取cache策略在这里看起来是必须的。

首先想到的是把整个会话界面的View static化，不同的会话对应的Activity复用同一个View进行渲染展示，但这么做会出现Activity中的Context（上下文）与View中的Context不一致的问题，View能复用的前提是必须保证View及其子View中的Context与Activity容器的Context一致，否则会出现诸如当前界面弹出的对话框关闭后返回的界面不是此前的界面，或者由于旧Context对象被当前的Activity持有导致旧Activity内存泄露等一系列的问题。

另外，由于Android系统组件ActivityManager进行Activity调度时候本身涉及较多的计算，在低端机器上这个调度时长一度超过150ms，即便在部分高端机上也有超过100ms的情况。我们发现，通过Fragment代替Activity实现界面切换，能够解决因ActivityManager调度耗时较久的问题，并且如果进一步考虑，上述View缓存的问题实际就能够换成用Fragment实现解决，关于Fragment，简单引入介绍下：
> Fragment
Android是在Android 3.0 (API level 11)开始引入Fragment的，并对2.x系列提供了support包支持。可以把Fragment想成Activity中的模块，这个模块有自己的布局，有自己的生命周期，单独处理自己的输入，在Activity运行的时候可以加载或者移除Fragment模块，同时可以把Fragment设计成可以在多个Activity中复用的模块，当开发的应用程序同时适用于平板电脑和手机时，可以利用Fragment实现灵活的布局，改善用户体验。

通过Fragment方式，我们把会话界面的实现进行了一次改造,如下图:

![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPJa1G2c5B8eo5UaLVxWuPxMclHfViaAVbqUyDUlAKsiaFedarw7N5d0qA/0?wx_fmt=png)

其中，蓝色线框内表示会话界面已从原来的Activity模式切换成Fragment，与4个子TAG设计在同一层，只要进程不销毁，会话界面就不会重建，会话进入/退出通过控制Fragment的可见/隐藏来实现。这样一来，在首次创建了会话界面后，后续再次打开，只需要把相关的变量复位，列表控件内所有子View也不需要重建（因数据适配器adapter没有更换），我们要做的是仅仅是刷新要显示的数据，及复位子View的状态。

采用上面方案，也面临一些问题：

*   动画播放由原来Activity级别降成View级别，Activity的动画是Window（窗口）级别的，系统对Activity动画播放原理上是通过矩阵变换并且是通过WindowManager所在进程执行，而改成View后切换动画则是通过本进程不断的重绘View自身来实现，效率上会降低。
*   在播放动画过程中，如果主线程刚好执行到此前通过定时器分发过来的一些较为耗时的任务，会导致动画丢帧，针对该问题，我们有自己的线程池及Handler消息队列管理，在播放过程中暂停Handler的消息派发及降低线程池内其他线程的优先级来解决。
*   会有部分低端机器因GPU性能过差导致播放动画卡顿。

    实际上，我们经过对的对国外优秀app一些研究成果注意到，国外的一些较高大上的公司的产品如google的环聊，facebook的messenger，均采用类似的方案，权衡利弊后，最后采用的是该方案。

    ### 数据佐证

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygP0bdSFafpSOKiciaSk2OfstAxa7n7GExxXgE8LuIeYAfyOKKZquLt8Dww/0?wx_fmt=jpeg)

    从测试同学反馈的测试数据来看，提升幅度是较为明显的，首次打开会话提升约10%-15%，非首次打开提升约50%-70%。虽然弊端是仍然存在在某些场合及机型下进入会话动画不够流畅问题，但对大多数机型上的体验来说，带来的提升是较大的。

    至此我们第一阶段的优化到此为止。

    * * *

    **第二阶段，针对会话内历史记录数量较大的情况**

    我们有自己的SQL性能数据上报系统,通过该系统，可以查看到外头用户SQL的执行耗时情况从上报的SQL性能统计系统来看，在会话内记录数较大情况下某些场景（进入会话，向上翻页，查看大图，删除历史消息等等）存在一定程度性能问题,见下图:

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPaScTFoGibO7DKwHic4N20pzh5gthedo4GNHPekusa23xc3QjagGKPZuQ/0?wx_fmt=png)

    上面“message&quot;为我们微信用于存储消息的表名，“lasttime”则是对应SQL的平均执行耗时（单位毫秒），以&quot;talker=?&quot;打头作为where过滤条件的SQL是消息模块涉及的查询语句，从平均的执行耗时来看这些SQL应该存在一定的优化空间。

    首先我们挑2条直接影响进入会话/会话内数据刷新速度的2条SQL语句进行explain query plan分析：

    1.计算会话内消息条数

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPDNQdbWr5h9jVNvUOSkicyLbz4yPJZESaLD4mRCupc1FVfmYS3XytD9w/0?wx_fmt=jpeg)

    2.查找会话内最近的18条消息并以时间升序方式排序

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPRRNPia0q2oEYN7XuZz1cBbzCkOFWOJowBHMxuiaZ1IqSc8jUKstR4npQ/0?wx_fmt=jpeg)

    先简要介绍一下explain query plan :没用过的同学可以直接看（http://www.sqlite.org/eqp.html）

    引用官方的一段话：
> The EXPLAIN QUERY PLAN SQL command is used to obtain a high-level description of the strategy or plan that SQLite uses to implement a specific SQL query. Most significantly, EXPLAIN QUERY PLAN reports on the way in which the query uses database indices

    简而言之，该指令是查看sqlite在执行SQL时候所采用的计划，例如，可以看到执行该SQL时候所采用的index（索引），并且可以看到执行该SQL过程前sqlite对整个查询所涉及的元数据条数的预估。
    
    此前，通过该指令，我们很轻松解决了很多明显的SQL设计上的问题，但这次貌似该指令**也无法让我们清晰定位到性能瓶颈**， 从explain query plan 的结果来看，在进行上述2个查询时候，sqlite 已经采用了我们预期指定的索引，并且预估值约是10条左右。

 OK，一切看起来很正常。那么，问题又出在哪里?

 针对该问题，在与ios相关同事交流过后，我们首先想到的是：拆表！

 **当时能想到的拆表之后的一些优势如下：**

 *   数据内聚，减少I/O> sqlite所有的表是通过B+树进行存储，当整个message表数据量较大的时候，因该表所在的B+树的深度较大，所有的查询或更新操作都会因此而多走很多的磁盘I/O流程。 而把message表按照talker（联系人）为单位分表，一个联系人一个表。则整个消息的存储就在物理空间上被分成了多个区间，同一个联系人的消息，在空间上被内聚到临近的磁盘块，这样的话，整个消息模块所在的B+树的深度就降低了，读取时候也会因磁盘的临近性（连续4k，磁盘一次读取最小的单位，大小根据不同磁盘的实际设定而定）而减少不少的磁盘I/O，上面的查询慢的问题也就解决了。（背景:关于B+树介绍,可见http://www.semaphorecorp.com/btp/algo.html）

 *   增加损坏后恢复数据成功机率> 用过sqlite的同学应该清楚，其存在不可避免的损坏机率，（关于损坏的介绍，建议直接看官方介绍 http://sqlite.org/howtocorrupt.html），我们此前对这种损坏的情况做了一套DB损坏后尝试恢复数据的方案，该方案从统计数据看恢复成功率在80%左右，而把消息分散到各个talker表，即便db损坏了，进行数据恢复的时候，恢复数据的成功率就会相应的比此前更高，因为损坏的范围缩小到以当前的talker为单位，与其他联系人的会话数据不会丢失。

 ##没那么简单

    从上面2个分析的点来看，听上去很有道理，而且实际带来的优势也的确如此，但我们只看到了好的一面，还没有看到负面的影响,在经过一段时间的拆表改造之后，陆陆续续发现问题来了，列举如下：

 *   第一点：开发周期长，牵扯范围大
 > message是整个微信的主模块，各个子模块都或多或少与其有干系，不少模块直接把message的id当作自己模块主表的外键，还有直接以message的id值作为文件路径的，此外按照talker分表后，原来以非talker开头的多列索引全部被废掉，涉及到这些索引的一系列功能需要重新实现等等。。。简而言之，牵扯的范围非常广，且往后的数据迁移几乎成了不可能。

 *   第二点：启动速度被拖垮，内存暴涨
 > 这个点，也是我们真正放弃拆表的最主要的原因：在创建了一定数量的联系人会话，我们发现，启动速度越来越慢了，经过分析之后发现，在创建了2000个消息会话（也就是2000张表）之后，进程重启后首次调用sqlite db模块进行prepare SQL（_sqlite在执行每条SQL前需要先将该SQL编译成用于查询引擎执行的字节码，该过程为prepare_）耗时将近2s ! 通过Android系统自带的traceview跟踪如图：

    2000个联系人会话：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPwgeSSP8PtXp5Dsq9UdNf6Db27aFSFlTq8dZzq3YPHENHHibu1G2liclA/0?wx_fmt=jpeg)

    **拆表后启动时首次prepare SQL 占整个启动过程cpu开销的40%以上！这还仅仅是2000个联系人会话，随着会话数的增多，该值线性增大。**

 这个数据与ios同学的此前对ios版本db-init 耗时的统计一致，这里引用一下ios组提供的一组数据

 ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPJD1aicdDOVtxh5ZG16n6ibSX4hV1j8lZOBzvVKh6Nh5zEbxmQMyn6WWA/0?wx_fmt=jpeg)
（iphone 4）

    在iphone4 上面，在联系人会话数2k以内，启动时间达到2-5s。

    **另外，对微信进程通过dumpsys meminfo 查看内存占用情况：拆表版本pss进程比单表版本高10mb！**

    拆表：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygP4u5XYpPTicfoiabfyYibj8vwgibyIvIO42O4ibnVjtnsWrBKRQxRKwR7xibg/0?wx_fmt=png)
    
    单表：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygP81WOzTKZa5NicibUAHofUaFD3e6LYnVqoBicVDgnUnqlfaPqMPIiawIT1Q/0?wx_fmt=png)

    可以清晰的看到，**拆表后Native Heap 比原来单表情况飙升10mb**。

    那么问题来了：

    **这个sqlite 首次prepare SQL耗时如此之久，且暴涨的10mb内存，源自何处？能否进行优化？**

    首先我们尝试google，去查询这块资料，遗憾的是，我们并没有找到比较详细的这块的资料，带着问题，我们来到sqlite底层进行profile及debug调试分析： 通过Counters分析， sqlite db首次prepare SQL：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPuKXnIDpjib0uZG6oSLPhCpyibic6YTmG6Tzf6o8NmpAMvZUWqVYKaNynQ/0?wx_fmt=jpeg)

    可见，实际耗时较大位置的在sqlite3Parser里面，分别为yy__reduce 及 sqlite3Malloc，其中yy__reduce为由文法自动生成解析器代码，_sqlite3Malloc_为对malloc的包装，用于底层分配内存。 通过调试源码发现，上面两步实际为对sqlite系统表&quot;sqlite_master&quot;内所有存储的&quot;create&quot;语句（包括create table，index 等）进行分词，解析等一系列操作，生成一个常驻的内存结构，见sqliteInt.h。

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPybibhiaiaFB3BXpNOC3shadxKM1N8XzUkpEGHoQ5HesdGwKwj7cXLpqLg/0?wx_fmt=jpeg)

    顾名思义，该结构体用于存放该db schema相关的一些信息，包括该db所有的表名，索引名，触发器名，正是有了这个结构体，sqlite prepare SQL时候才知道该怎么解释Tokenizer（分词器）传进来的一个个词，才有后面代码生成器对语法树进行字节码的生成。

    也就说，这部分内存，对于后续所有SQL的编译都是必不可少的，这块想去掉除非不用VDBE引擎，否则只能按照它的规则。而加快其解析过程，我们目前也正在研究，尝试把schema对应的内存序列化到磁盘，在init时候直接从磁盘反序列化回来，倒也是种思路，但像sqlite 里面的struct，稍有研究的同学应该都知道，其每个struct内部，都包含了多个其他struct，并且不少通过链表，hash 表等的形式组织，故单纯一个schema，实际上里面包含的struct信息都是相当多的，并且要想完全把其序列化到磁盘，必须对其内部每个结构都相当了解才能做到，最少以我们目前对其的研究程度，还做不到这个事情。

    所以，这里的耗时及内存占用，以我们目前的研究程度，还无法优化的，得到这个结论之后，我们放弃了拆表这个方案，并开始另觅性能可以达到或者接近拆表后的方案。

    ## **另觅方案**

    好了，饶了一圈，回到问题的起点，那就是为何上面的SQL从explain query plan 检测看到了实际已经采用了索引，看上去是没什么问题的，但最后在外面的用户上报的统计来看，会有不少超过1s乃至更高的耗时？

    带着这个问题，继续挖深挖sqlite 整个查询过程到底都干了什么? 在对同一个会话制造了一定量的数据之后，使用counters分析其执行过程如下：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPaqX84IV6Zsia3ghuVPJFyKux0gyO3vBChukcy02qAKmN5olA1XVsz6Q/0?wx_fmt=jpeg)

    从图上可见，整个查询耗时最长的部分为sqliteVdbeExec 及 seekAndRead

    sqliteVdbeExec为Vdbe引擎计算查询结果的执行函数，中间涉及较大量的计算，包括一系列的查找策略及对每条记录的解析，字节码执行等等 ，seekAndRead 则为查询过程所调用的I/O函数，性能直接取决于操作系统对应的磁盘速度： ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPkdrSEag3DicZicibr5LD313J68BIfYmxW40ROM5ytr2qoRjtbglqJmR3w/0?wx_fmt=jpeg)

    从上面的trace分析中，可见要降低整个查询的耗时，有2个较大的瓶颈需要解决，一个是磁盘I/O的数量，另外一个引擎的计算量，而引擎的计算量经过实际测试其与查询过程所需的用到Page的数量是成线性正比关系的，也就是说，要降低整个查询时长，必须先想办法降低整个查询过程中需要用到的Page数量。

    **PAGE 数量降低分析**

    首先在了解清楚sqlite 查询前需要先了解清楚数据在sqlite 每个Page内部的存放情况，详细的可以到官方主页上看 （ https://www.sqlite.org/fileformat2.html），这里不细说,引入一个图：

    每个page头部的格式（8-12字节）

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPslNABYAdNSIBqh4EUyicxBEHNmzqnksYgDAVtfPMbzQBom4kdtZbsTg/0?wx_fmt=png)

    这里只需要知道怎么区分每个Page类型就足够了，从上面的格式上看，我们可以看到，只需要通过页头首字节就能够区分出其page类型。 sqlite的Page通过页头**首字节**划分，有如下几种类型：对于索引页，内部页为 0X02，叶子页为0X0a ，对于表页，内部页为0X05 ，叶子页为0X0d。 在弄清楚每个page实际类型怎么区分之后，我们就可以在数据加载的关键地方加入信息跟踪整个查询涉及的page页面及其流向。

    ### PAGE加载跟踪小工具-PageTracer

    为了方便调试，我们单独写了一个小工具用于跟踪统计SQLite执行过程中Page的加载类型，累积个数及跳转流程等，名字叫&quot;PageTracer&quot;，其原理是在对每个page加载的必经函数前后插入回调函数进行统计，在整个SQL执行完毕后输出该执行过程统计结果。

    **PageTracer用法**

    PageTracer工具入参为具体SQL，结果为对应page统计数量

    **PageTracer 日志输出涵义：**
    > PageCount ：总Page数量

    表页相关
    > Table embedded : 表内部页数量 Table leaf:表叶子页数量

    索引页相关
    > Index embedded :索引内部页数量 Index leaf :索引叶子页数量

    对拆表与不拆表同一个talker 相同数据量情况下Page加载的类型及个数进行统计：

    拆表前SQL：
    <pre style="margin: 14px 10px; padding: 0px; display: block; white-space: pre-wrap; unicode-bidi: embed;">`&gt;PageTrace &quot;SELECT count(*) FROM message where talker = &#39;3494847533@chatroom&#39;&quot;

    result：all PageCount:1118 ,Table embedded :1,Table leaf :6,Index embedded:45 ,Index leaf :1066`</pre>

    拆表后SQL：
    <pre style="margin: 14px 10px; padding: 0px; display: block; white-space: pre-wrap; unicode-bidi: embed;">`&gt;PageTrace &quot;SELECT count(*) FROM talker_3494847533@chatroom&quot;

    result：all PageCount:414 ,Table embedded :1,Table leaf :6,Index embedded:8 ,Index leaf :329`</pre>

    从上述2组log一对比，我们可以很清晰的看到,真正差距就在索引页上，可见拆表前后上述2条SQL, 相差70%左右的索引页的加载。而经过时间打点看到，上述2组SQL查询时间差距也在70%左右，从这一角度来看，拆表的优势很明显。现在的问题就是为何2种实现sqlite对索引页加载的Page数量差这么大。

    ### 单条索引的构成

    在经过对官网对索引格式介绍的了解及单条索引的debug跟踪后，总结出不拆表前索引条目内部元数据（不包含头部格式）构成如下图：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPznrDnWD2MkCKTEWQPJxG3ibPFWSE4Ak0SaqD8NqG5PxMzE9gWB45iccQ/0?wx_fmt=jpeg)

    可见，在整条索引数据项里面，talker字段的长度占整条索引内部空间超过70%

    注：到这里，先引入一下SQLite可变长整数的介绍：
    >可变长整数是SQLite的特色之一，使用它既可以处理大整数，又可以节省存储空间。由于单元中大量使用可变长整数。可变长整数由1~9个字节组成，每个字节的低7位有效，第8位是标志位。在组成可变长整数的各字节中，前面字节(整数的高位字节)的第8位置1，只有最低一个字节的第8位置0，表示整数结束。可变长整数可用于存储rowid、字段的字节数或Btree单元中的数据。

    故实际**每个byte能够表示的整数个数为128**（因只有低7位可用）。

    上图之所以描述rowid 占用长度为1-3byte， 实际原因为3个byte可以表示的整数个数为 128 * 128 * 128 ~= 209w 。 假设不拆表，则按照微信正常的使用情况，用户的聊天记录数在 200w 以内，则对rowid的存储，3个字节完全足够了，若聊天记录在 1.6w 以内，则需2个字节则可存储。

    在拆表后，单条索引构成如下：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygP8nODhQibpb7KApWnLiamq87bKfF07pcAHH6d6DfdSiaibRicRIADjn3zHRw/0?wx_fmt=jpeg)

    可见，拆表后，真正产生优化的原因为头部talker字段的占用被去除，另外，因为message被拆分成多个talker表，故对于部分talker表，由于聊天记录总数变小，该talker表内条数只要小于1.6w，rowid就只占用2个字节以内， 这种情况下rowid会节省1个字节，但不是主要的优化因素，**关键还是头部大小的节省**。

    至此，整个拆表带来的性能优势从存储的角度就已经很清晰的分析出来，整个优化效应链见下：

    **单条索引记录占用降低 —&gt; 用于存储索引的Page数量降低 —&gt; 用于查询加载的Page量降低 —&gt; 整个查询时间降低**

    从上面对其优势分析清楚之后，我们考虑到，既然这里talker字段是大头，而sqlite 对整数的是可变长整数，也就说，我们通过以talker作为索引第一个字段，占据了整个索引条目空间的60-70%，而我们的talker在数据库是以用户username（字符串）来存储，对于群聊及大部分用户的username，这个字符个数都将近20-24个字符，而我们的索引组的后面几列字段都是整型存储，说也就是大部分情况我们的索引条目除去talker字段外的几个字段，均是1-3个字节的占用，也就说以一个page大小1024个字节算，光存储talker字段就占了将近600-700个字节。这样子的话，对索引空间的利用率是极低的。

    实际情况中，对同一个用户，联系人会话实际情况基本不会超过1w个，也就是这1w个不同的联系人，我们如果用整型作为id存储的话，整数范围只是1-10000，按照前面的说法，在大多数情况下，2个字节已经完全足够了，原因前面可变长整数那里已经介绍了，2个字节实际能表示的整数个数为128 * 128 = 16384个数据，也就说对于1w个联系人的情况，索引头也仅仅需要2个字节而已。相对原来的20个字节，降低了90%的占用。

    针对该情况，我们对原来的talker字段进行了一级映射，把原来的字符串形式映射成整型字段（1～10000内），并对该字段建立相应的索引，代替掉旧索引。在进行这一级的优化后，所有会话内对talker字段的查询，均在底层进行了一次转换，以新的整型id代替原来的字符串，单条索引的空间占用降低为原来的30%，优化后索引条目构成如下图：

    ![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPC210ukudcklSPNa8h30Z24ucPFgm7Qqt5VhlcbuPwHVTeicrWM1OHoQ/0?wx_fmt=jpeg)

    这样的话，对索引进行查找的过程，就只需要原来的30%的page加载就可以完成。

    PageTrace一下看看结果：
    > &gt;PageTrace &quot;SELECT COUNT(*) FROM message where talkerid = 202&quot;
    
    > result：all PageCount:437 ,Table embedded :1, Table leaf :6,Index embedded:10 ,Index leaf :349

    可见,虽然还没完全达到拆表后的性能，但整个查询过程中索引Page数量在总量上已经接近了，与拆表比，索引叶子Page多加载20个，内部Page多加载2个，综合内存及启动速度考虑，明显这个方案更优。

    同理，通过PageTracer分析进入会话涉及的另外一条SQL我们也轻易发现问题：

    查找会话最近18条消息：

    此前SQL：
    >&gt;PageTrace &quot;SELECT * FROM message where talker = &#39;3494847533@chatroom&#39; order by createTime ASC limit -1 offset 30000&quot;

    >result：all PageCount:1382 ,Table embedded :11,Table leaf :213,Index embedded:10 ,Index leaf :349`</pre>

    优化后SQL：
    > &gt;PageTrace &quot;SELECT * FROM (SELECT * FROM message where 
    talker=&#39;3494847533@chatroom&#39;order by createTime desc limit 18)order by createTime ASC&quot;

    > result：all PageCount:22 ,Table embedded :4,Table leaf :13,Index embedded:4 ,Index leaf :1   

**数据佐证：**

见来自测试同事的反馈的测试数据：

**写操作**

![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygPIxvXYGLRYAadpRyz08QrRNme6RqwicvEoOwicEeKoIz5EO2xxbQFXYQQ/0?wx_fmt=jpeg)
       （会话内30w条记录）

**读操作**

在会话内条数达到10w以上之后，进入会话及刷速度提升幅度超过70%

![](http://mmbiz.qpic.cn/mmbiz/csvJ6rH9MctB8VGcUoMg8FYKLd5fuygP5ATabXqyVibticWpljsesVCSpONjib4MZ0mfvgkKPrZKjdhSyWlcajH9A/0?wx_fmt=jpeg)

至此，整个优化流程就完毕了，整个优化到最后看起来结论很简单，但每一步的验证，背后都是大量的研究及论证的过程，分享此文出来，希望能减少大家在此走的弯路。

## 下一步

*   我们会通过对每条SQL 涉及的Page数据及相应类型进行统计，以区分查询语句设计的好坏，解决用explain query plan无法检测出的SQL设计问题。

*   对于类似字符串等占用较长空间做索引字段的，未来会通过代码扫描直接提示warning，加强各个团队成员在这方面的意识。


[reflink]:https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207548094&idx=1&sn=1a277620bc28349368b68ed98fbefebe&scene=1&srcid=lbTtbXQRHO0soAYOzzD2&key=dffc561732c22651cf13a2ee4f45620137344a21f83167444e033088f26e812fa6307ca51e115edcf9bacf54184fd6b1&ascene=0&uin=Mjc4MjU3MDQw&devicetype=iMac+MacBookPro11%2C2+OSX+OSX+10.10.5+build(14F27)&version=11020201&pass_ticket=kNXrPWwF37WmkcrIR%2FjG2Gj%2BPznLc1gxbd9eWs3zqQgSXUbTHFWZvA7pwPeW36Sp




