# Bigtable摘要

Bigtable是设计用来管理那些可能达到很大大小(比如可能是存储在数千台服务器上的数PB的数据)的结构化数据的分布式存储系统。Google的很多项目都将数据存储在Bigtable中，比如网页索引，google 地球，google金融。这些应用对Bigtable提出了很多不同的要求，无论是数据大小(从单纯的URL到包含图片附件的网页)还是延时需求。尽管存在这些各种不同的需求，Bigtable成功地为google的所有这些产品提供了一个灵活的，高性能的解决方案。在这篇论文中，我们将描述Bigtable所提供的允许客户端动态控制数据分布和格式的简单数据模型，此外还会描述Bigtable的设计和实现。 

1.导引

在过去的2年半时间里，我们设计,实现,部署了一个称为Bigtable的用来管理google的数据的分布式存储系统。Bigtable的设计使它可以可靠地扩展到成PB的数据以及数千台机器上。Bigtable成功的实现了这几个目标：广泛的适用性，可扩展性，高性能以及高可用性。目前，Bigtable已经被包括Google分析，google金融，Orkut，个性化搜索，Writely和google地球在内的60多个google产品和项目所使用。这些产品使用Bigtable用于处理各种不同的工作负载类型，从面向吞吐率的批处理任务到时延敏感的面向终端用户的数据服务。这些产品所使用的Bigtable集群也跨越了广泛的配置规模，从几台机器到存储了几百TB数据的上千台服务器。 

在很多方面，Bigtable都类似于数据库：它与数据库采用了很多相同的实现策略。目前的并行数据库和主存数据库已经成功实现了可扩展性和高性能，但是Bigtable提供了与这些系统不同的接口。Bigtable并不支持一个完整的关系数据模型，而是给用户提供了一个可以动态控制数据分布和格式的简单数据模型，允许用户将数据的局部性属性体现在底层的数据存储上。数据使用可以是任意字符串的行列名称进行索引。Bigtable将数据看做是未经解释的字符串，尽管用户经常将各种形式的结构化或半结构化的数据存储到这些字符串里。用户可以通过在schema中的细心选择来控制数据的locality。最后，Bigtable的schema参数还允许用户选择从磁盘还是内存获取数据。 

第2节更加详细的描述了该数据模型。第3节提供了关于用户API的概览。第4节简要描述了Bigtable所依赖的底层软件。第5节描述了Bigtable的基本实现。第6节描述了我们为提高Bigtable的性能使用的一些技巧。第7节提供了一些对于Bigtable的性能测量数据。第8节展示了几个Google内部的Bigtable的使用实例。第9节讨论了我们在设计支持Bigtable所学到的一些经验教训。最后第10节描述了相关工作，第11节进行了总结。

 2．数据模型

Bigtable是一个稀疏的，分布式的一致性多维有序map。这个map是通过行关键字，列关键字以及时间戳进行索引的；map中的每个值都是一个未经解释的字节数组。

(row:string,column:string,time:int64)  -> string

我们在对于这种类Bigtable系统的潜在使用场景进行了大量考察后，最终确定了这个数据模型。举一个影响到我们的某些设计决策的具体例子，比如我们想保存一份可以被很多工程使用的一大集网页及其相关信息的拷贝。我们把这个表称为webtable，在这个表中，我们可以使用URL作为行关键字，网页的各种信息作为列名称，将网页的内容作为表的内容存储：获取的时候还需要在列上加上时间戳，如图1所示。

【google论文四】Bigtable:结构化数据的分布式存储系统(上) - 星星 - 银河里的星星

 表中的行关键字是大字符串(目前最大可以到64KB，尽管对于大多数用户来说最常用的是10-100字节)。在一个行关键字下的数据读写是原子性的(无论这一行有多少个不同的列被读写)，这个设计使得用户在对相同行的并发更新出现时，更容易理解系统的行为。

 Bigtable按照行关键字的字典序来维护数据。行组{row range，将它翻译为行组，一个row range可能由多个行组成}是可以动态划分的。每个行组叫做一个tablet，是数据存放以及负载平衡的单位。这样，对于一个短的行组的读就会很有效，而且只需要与少数的机器进行通信。客户端可以通过选择行关键字来利用这个属性，这样它们可以为数据访问得到好的局部性。比如，在webtable里，相同域名的网页可以通过将URL中的域名反转而使他们放在连续的行里来组织到一块。比如我们将网页maps.google.com/index.html的数据存放在关键字com.google.maps/index.html下。将相同域名的网页存储在邻近位置可以使对主机或域名的分析更加有效。

 列族

不同的列关键字可以被分组到一个集合，我们把这样的一个集合称为一个列族，它是基本的访问控制单元。存储在同一个列族的数据通常是相同类型的(我们将同一列族的数据压缩在一块)。在数据能够存储到某个列族的列关键字下之前，必须首先要创建该列族。我们假设在一个表中不同列族的数目应该比较小(最多数百个)，而且在操作过程中这些列族应该很少变化。与之相比，一个表的列数目可以没有限制。

 一个列关键字是使用如下的字符来命名的：family:qualifier。列族名称必须是可打印的，但是qualifier可能是任意字符串。比如webtable有一个列族是language，它存储了网页所使用的语言。在language列族里，我们只使用了一个列关键字，里面存储了每个网页的language id。该表的另一个列族是anchor，在该列族的每个列关键字代表一个单独的anchor，如图1所示。Qualifier是站点的名称，里面的内容是链接文本。

 访问控制以及磁盘的内存分配都是在列族级别进行的。在webtable这个例子中，这些控制允许我们管理不同类型的应用：一些可能会添加新的基础数据，一些可能读取这些基础数据来创建新的列族，一些可能只需要查看现有数据(甚至可能因为隐私策略不需要查看所有现有数据)。

 时间戳

Bigtable里的每个cell可以包含相同数据的多个版本；这些不同的版本是通过时间戳索引的。Bigtable的时间戳是一个64位的整数。它们可以由Bigtable来赋值，在这种情况下它们以毫秒来代表时间。也可以由客户端应用程序显式分配。应用程序为了避免冲突必须能够自己生成唯一的时间戳。一个cell的不同版本是按照时间戳降序排列，这样最近的版本可以被首先读到。

为了使不同版本的数据管理更简单，我们支持2个针对每个列族的设定来告诉Bigtable可以对cell中的数据版本进行自动的垃圾回收。用户可以指定最近的哪几个版本需要保存，或者保存那些足够新的版本(比如只保存那些最近7天写的数据)。

 在我们的webtable中，我们将爬取的网页的时间戳存储在内容里：这些时间说明了这些网页的不同版本分别是在何时被抓取的。前面描述的垃圾回收机制，使我们只保存每个网页最新的三个版本。

 3. API

Bigtable API提供了一些函数用于表及列族的创建和删除。它也提供了一些用于改变集群，表格及列族元数据的函数，比如访问控制权限。

 客户端应用程序可以写或者删除Bigtable里的值，从行里查找值或者在表中的一个数据子集中进行迭代。图2展示了使用RowMutation执行一系列更新的C++代码(为了保持简单省略了不相关的细节)。Apply调用对webtable执行了一个原子性的变更操作：给www.cnn.com增加一个anchor，然后删除另一个anchor。

【google论文四】Bigtable:结构化数据的分布式存储系统(上) - 星星 - 银河里的星星

图3展示的c++代码使用Scanner来在一个特殊行上的所有anchor进行迭代，用户可以在多个列族上进行迭代，存在几种机制来对扫描到的行，列，时间戳进行过滤。比如我们可以限制只扫描那些与正则表达式”anchor:*.cnn.com”匹配的列，或者那些时间戳距离当前时间10天以内的anchor。

【google论文四】Bigtable:结构化数据的分布式存储系统(上) - 星星 - 银河里的星星

 Bigtable提供了几种其他的feature允许用户使用更复杂的方式熟练控制数据。首先，Bigtable支持单行事务，能够支持对存储在一个行关键字上的执行原子性的读写修改序列。Bigtable当前并不支持跨行的事务，尽管它提供了一个多个用户的跨行写的接口。其次，Bigtable允许用户将cell作为一个整数计数器来使用。最后，Bigtable支持在服务器地址空间内执行一个客户端脚本。这些脚本是使用google内部开发的数据处理语言sawzall编写的。当前，我们基于Sawzall的API不允许客户端脚本向Bigtable中写回，但是它允许进行各种形式的数据转换，基于各种表达式的过滤以及大量的统计操作符。

 Bigtable可以与MapReduce(google内部开发的一个运行大规模并行程序的框架)一起使用。我们写了很多wrapper它允许将Bigtable作为输入源或者输出目标。

4. 基础构件

Bigtable是建立在google的其他几个设施之上。Bigtable使用GFS来存储日志和数据文件。Bigtable集群通常运行在一个运行着大量其他分布式应用的共享机器池上。Bigtable依赖于一个集群管理系统进行job调度，共享机器上的资源管理，处理机器失败以及监控机器状态。

 Bigtable内部采用Google SSTable文件格式来存储数据。一个SSTable提供了一个一致性的，有序的从key到value的不可变map，key和value都是任意的字节串。操作通常是通过一个给定的key来查找相应的value，或者在一个给定的key range上迭代所有的key/value对。每个SSTable内部包含一系列的块(通常每个块是64KB大小，但是该大小是可配置的)。一个块索引(保存在SSTable的尾部)是用来定位block的，当SSTable打开时该索引会被加载到内存。一次查找可以通过一次磁盘访问完成：首先通过在内存中的索引进行一次二分查找找到相应的块，然后从磁盘中读取该块。另外，一个SSTable可以被完全映射到内存，这样就不需要我们接触磁盘就可以执行所有的查找和扫描。{关于SSTable(StaticSearchTable)的具体格式可以参考YunTable开发日记（4）-BigTable的存储模型，中对HBASE的HFile的介绍}

 Bigtable依赖于一个高可用的一致性分布式锁服务Chubby。Chubby由5个活动副本组成，其中的一个选为master处理请求。当大部分的副本运行并且可以相互通信时，该服务就是活的。Chubby使用Paxos算法来在出现失败时，保持副本的一致性。Chubby提供了一个由目录和小文件组成的名字空间。每个文件或者目录可以当作一个锁来使用，对于一个文件的读写是原子性的。Chubby的客户端库为Chubby文件提供一致性缓存。每个Chubby客户端维护这一个与Chubby服务的会话。如果在租约有效时间内无法更新会话的租约，客户端的会话就会过期。当一个客户端会话过期后，它会丢失所有的锁和打开的文件句柄。Chubby客户端也会在Chubby文件和目录上注册回调函数来处理这些变更或者会话的过期。

 Bigtable使用Chubby来完成各种任务：保证任意时刻最多只有一个活动的master；存储Bigtable数据的bootstrap location(参见5.2节)；发现tablet服务器以及finalize tablet服务器的死亡(参见5.2节)；保存Bigtable schema信息(每个表的列族信息)；存储访问控制列表。如果chubby在一段时间内不可用，Bigtable也会不可用。我们最近在使用了11个chubby实例的14个Bigtable集群进行了测量。由于Chubby不可用而造成的存储在Bigtable上的数据不可用的平均概率是0.0047%。对于单个集群来说，由于Chubby不可用造成的这个概率是0.0326%。

 5. 实现

Bigtable实现有3个主要的组件：每个客户端需要链接的库，一个master服务器，很多tablet服务器。tablet服务器可以从一个集群中动态添加(或者删除)来适应工作负载的动态变化。

 Master负责将tablet分配到tablet服务器，检测tablet服务器的添加和过期，平衡tablet服务器负载，GFS文件的垃圾回收。另外，它还会处理schema的变化，比如表和列族的创建。

每个tablet服务器管理一个tablets集合(通常每个tablet服务器有10到1000个tablet)。Tablet服务器负责它已经加载的那些tablet的读写请求，也会将那些过于大的tablet进行分割。{tablet服务器本身实际上也是GFS的用户，它们只是负载它加载的那些tablet的管理，这些tablet的物理存储并不一定存放在管理它的服务器上，底层的存储是由GFS完成的，tablet服务器可以只调用它的接口来完成相应任务。而METADATA表中的位置信息应该是指某个tablet由哪个tablet服务器管理，而不是物理上存储在哪个机器上。}

 正如很多单master的分布式存储系统，客户端数据的移动并不会经过master：客户端直接与tablet服务器进行通信来进行读写。因为Bigtable客户端并不依赖于master得到tablet的位置信息，大部分的客户端从来不会于master通信。所以，master实际中通常都是负载很轻的。

 Bigtable集群存储了大量的表。每个表由一系列的tablet组成，每个tablet包含一个行组的所有相关数据。一开始，每个表由一个tablet组成。随着表格的增长，它会自动分割成多个tablet，它们大小默认是100-200MB。

 5.1 Tablet存放位置

我们使用一个类似于B+树的三级结构来存储tablet的放置信息如图4。

【google论文四】Bigtable:结构化数据的分布式存储系统(上) - 星星 - 银河里的星星

第一级是一个存储在Chubby的包含了root tablet位置信息的文件。root tablet包含了在一个特殊的METADATA表里的所有tablet的位置信息{root tablet实际上是METADATA表的第一个tablet，它存储了该表其他的tablet的位置信息}。每个METADATA tablet包含了一集用户tablet的位置。Root tablet仅仅是METADATA表的第一个tablet，但是是特殊对待的-它永远不会被分割-为了保证tablet位置信息的层次结构不会超过3级。

 METADATA表的每一个行关键字(由tablet所属的表标识符和它的结束行组成)下存储了一个tablet的位置。每个METADATA行在内存中大概存储了1KB数据。我们限制METADATA的tablet的大小为128MB，我们的三级层次结构足以用来寻址2^34个tablet(如果tablet按照128MB算，就是2^61字节){root tablet大小为128M，每个行1KB，那么它就可以存储128*2^20/2^10=128*2^10个METADATA tablet，同样的，每个METADATA tablet可以存储128*2^10个普通tablet，这样总共可以存储128*2^10*128*2^10即2^34个普通tablet，每个tablet又将近1KB数据，这样算起来存储这些元信息就需要4TB的数据，所以该METADATA表也不可能全部放入内存，而是采用与普通的表一样的存储方式，放在GFS上。但是会把某些特殊信息放在内存中，比如第6节提到的：METADATA中的location列族会被放入内存 }。

 客户端库会缓存tablet的位置信息。如果客户端不知道某个tablet的信息，或者发现缓存的位置信息是错误的，那么它就会递归地在tablet位置存储结构中查找。如果客户端缓存是空的，定位算法需要三次网络往返，包括从Chubby的一次读操作。如果客户端缓存是陈旧的，定位算法将需要多达6次的往返，因为陈旧的缓存值只有在不命中时才会被发现(假设METADATA tablet并不会经常移动)。尽管Tablet位置信息是存储在内存中(如上所述)，不需要访问GFS，但是我们还是通过在客户端库里进行预取来降低花费：当读取METADATA表时，它会读取不止一个tablet的信息。

 我们也会将一些额外信息存放在METADATA表里，包括对于每个tablet有关的事件日志(比如一个服务器何时开始提供服务)。这些信息对于调试和性能分析很有帮助。

 5.2 tablet分配

每个Tablet一次只会分配给一个tablet服务器。Master保存了现有的活着的tablet服务器集合的所有行踪，tablet服务器当前分配的tablet，哪些tablet未被分配。当一个tablet没有被分配并且有一个可以存储该tablet的tablet服务器存在时，master通过给那个tablet服务器发送一个tablet负载请求来分配该tablet。

 Bigtable使用Chubby来追踪tablet服务器的状态。当一个tablet服务器启动时，它创建并获取一个在给定的Chubby目录上的唯一命名的文件的独占锁。Master通过监控这个目录(服务器目录)来发现tablet服务器。Tablet服务器如果丢失了它的独占锁(比如由于网络问题)就停止它上面的tablet服务。一个tablet服务器会尝试重新获取在该文件上的独占锁，只要该文件还存在。如果该文件也不存在了，那么tablet服务器就永远无法提供该服务了。当一个tablet服务器停止(比如集群管理系统从集群中删除了该tablet 服务器机器)，它就会尝试释放这个锁这样master就可以更快地重新分配它上面的tablets了。

 Master负责检测一个tablet服务器何时停止提供服务，以尽快重新安排它的tablets。为了进行检测，master周期性的向每个tablet服务器询问它的锁状态。如果一个tablet服务器报告它丢失了它的锁，或者master在它的几次尝试中不能到达一个服务器，master会尝试获取该服务器的锁。如果master可以获取该锁，那么Chubby就是活的，而tablet服务器要么是死的要么因为某些问题而无法到达Chubby，那么master就可以通过删除它的server文件来使得该tablet服务器永远都不能提供服务。一旦一个服务器的文件被删除了，master就可以将之前分配给该服务器的所有tablet移到那些为分配的tablet集合中。为了保证一个Bigtable集群不会因为与master和Chubby间的网络问题而变得脆弱，如果master的Chubby会话过期了，master会自杀。然而，如前面所述，master的失败并不会改变tablet服务器的tablet分配。

当master被集群管理系统启动后，在它可以改变tablets之前需要知道它们当前的分配状态。Master在启动时执行如下步骤：1.master在Chubby获得一个唯一的master锁，该锁可以防止出现同时生成多个master实例。2.master扫描Chubby的服务器目录来找到所有活着的服务器。3.master与活着的tablet服务器通信来发现每个服务器安排了哪些tablet。4.master扫描METADATA表来找到tablet集合。当扫描中碰到一个未被分配的tablet，master会将它添加到未分配的tablet集合，并对这个tablet进行分配。

在METADATA 的tablets未被分配之前，对于METADATA的扫描不能进行。因此在开始扫描之前(步骤4)，如果在步骤3没有发现对于root tablet的分配，master会将root tablet添加到未分配tablets集合中。这个添加将会使root tablet变得可以被分配。因为root tablet包含所有METADATA tablets的名称，master当扫描完root tablet后就能知道METADATA的所有的tablets。

 只有当一个表被创建，现有的两个tablets合并为一个，或者一个tablet被分割为一个时，现有的tablet集合才会发生变化。Master能够追踪所有的这些变化，因为它负责维护它们。Tablet分割需要特殊对待因为它们是由一个tablet服务器启动的。Tablet服务器通过将新的tablet的信息记录到METADATA表中来提交这个分割。当分割提交后，它会通知master。为了防止分割通知丢失(因为tablet服务器或者master死了)，当master向tablet服务器请求加载刚刚发生分割的那个tablet时，它会检测到这个新的tablet。Tablet服务器会将这个分割通知master，因为它在METADATA表中的tablet键值仅包含了master让它加载的那个tablet的一部分。{假设master没有收到这个分割通知，那么它所记录的tablet与METADATA表中的就是不一致的，这样在它让tablet服务器加载该tablet时就会发现该不一致}。

 5.3 tablet服务

Tablet的持久性状态是通过GFS进行存储的，如图5所示。

【google论文四】Bigtable:结构化数据的分布式存储系统(上) - 星星 - 银河里的星星

更新是提交到一个保存了redo记录的提交日志里。在这些更新里，最近提交的那些被保存到内存中一个叫做memtable的有序缓存里，老的更新则被保存在一系列的SSTable中。为了恢复一个tablet，tablet服务器从METADATA表中读取该tablet的元数据。元数据中包含组成该tablet的SSTable列表，以及一系列的redo点(指向那些包含该tablet数据的commit日志条目)的集合。Tablet服务器将所有SSTable的索引读入内存，然后通过应用那些从redo点开始以及提交的更新操作来重新构建memtable。

{更新操作肯定会被保存到commit log里，但是当某个服务器挂掉时，它那些保存在memtable的最新的更新就不存在了，而redo点应该就是记录已保存到SSTable的与还在memtable中的操作的分界点，这样通过重新执行它之后的那些操作就可以将memtable重建}{redo点何时被更新？有多少个commit log？参见第6节}

当一个写操作到达一个tablet服务器时，服务器首先检查它的格式是否合法，发送者是否有权限进行该操作。权限检查是通过从Chubby读取一个允许的写操作者列表(通常它直接存在于Chubby的客户端缓存中)完成的。一个合法的变更会被写入到已提交日志里。操作的提交可以通过分组执行来提高大量小变更操作出现时的吞吐率，当该写操作提交后，它的内容会被插入到memtable里。

当一个读操作到达一个tablet服务器时，类似的首先要进行格式和权限检查。一个合法的读操作将会在一系列SSTable和memtable的一个合并视图上执行{因为SSTable一旦写入就不可变，这样就使得更新操作必须写到新的SSTable中，这样就导致同一个key值可能在多个SSTable中出现，这样读取时就必须读取多个SSTable才能得到它真实的最终状态}。因为SSTable和memtable都是字典有序的数据结构，因此可以很快生成这个视图。{为了读取一个key时，要读入所有的SSTable，所以第6节有一个针对该问题的优化Bloom Filter。此外伴随着SSTable的增多，这种视图合并也会变得低效，所以也引出了下面的Compation}

当tablet发生分割或者合并时，也可以继续接受读写操作。

 5.4  compaction

伴随着写操作的执行，memtable的大小会逐渐变大。当memtable大小增长到一个阈值，这个memtable就会被冻结，一个新的memtable被创建，被冻结的旧的memtable会被转化为一个SSTable写入GFS。这个minor compaction过程有两个目的：降低tablet server的内存使用，降低该tablet服务器挂掉时需要从已提交日志中读取的数据大小。当compaction发生时，也可以继续接受读写操作。

 每次minor Compation会创建一个新的SSTable。如果这个行为持续的进行而不检查，那么读操作就可能会需要从大量的SSTable中合并它们的更新。我们通过周期性的执行一个merging compaction来将这样的文件数目限制在一定范围内。一个merging compaction读取多个SStable和memtable，然后写入到一个新的SSTable(形成一个最终的归并视图)。一旦这个compaction结束，这些SSTable和memtable就可以丢弃了。

将多个SSTable重新写入到一个SSTable的merging compaction称为主compaction。由非主compaction产生的SSTable里可以包含一些在旧的SSTable中仍然存活但是目前已经被删除的数据。另一方面，一个主compaction产生的SSTable不会包含删除操作信息或者已删除数据。Bigtable循环扫描它所有的tablet，周期的对它们执行主compaction。这些主compaction使得Bigtable可以回收那些被已删除的数据使用的资源。这也保证那些已经删除的数据在一定时间内会从系统中消失，这对于那些存储敏感数据的服务来说很重要。

 6 技巧

前面一节描述的实现需要大量的技巧来到达用户所需要的高性能，可用性，可靠性。这一节更细节地描述下实现的各个部分，着重讲述下使用的这些技巧。

 局部性群组(对应一个SSTable)

用户可以将多个列族组织为一个局部性群组。对于每个tablet里的每个局部性群组都会生成一个单独的SSTable。将那些通常不会被一起访问的列族分离到独立的局部性群组可以增加访问的效率。比如，webtable的关于网页的元数据(比如语言，校验和)可放到一个局部性群组，网页内容可以放到另一个群组里：这样一个访问元数据的应用程序就不需要读取所有网页的内容。

 另外，一些有用的tuning参数也可以以局部性群组为单位进行设置。比如一个局部性群组可以声明为放入内存的。对于声明为放入内存的局部性群组的SSTable在需要时才会加载到tablet服务器的内存中。一旦加载之后，对于该局部性群组的访问就不需要访问磁盘。这个特点对于那些需要经常访问的小片数据很有用：比如在Bigtable内部我们将它应用在METADATA表的location列族上。

 压缩

用户可以控制对于一个局部性群组的SSTables是否进行压缩，以及使用哪种压缩格式。用户指定的压缩格式会应用在SSTable的每个块上(块大小可以通过一个局部性群组的参数进行控制)。对于每个块单独进行压缩，尽管这使我们丢失了一些空间，但是这使得我们不需要解压整个文件就可以读取SSTable的部分内容。很多用户使用一个两遍压缩模式，第一遍压缩使用Bentley and McIlroy模式，该模式在一个很大的窗口大小里压缩普通的长字符串。第二遍压缩了一个快速压缩算法，该算法在一个小的16KB窗口大小内查找重复。这两遍压缩都很快速，压缩速率在100-200MB/s，解压速率在400-1000MB/s。

 尽管在选择压缩算法时，我们更重视速率而不是空间的减少，但是这个两遍压缩模式或做的出奇地好。比如，在webtable里我们使用这种压缩模式存储网页内容。实验中，我们在一个压缩的局部性群组里存储了大量文档。为了实验目的，我们将文档的版本数限制为1。该压缩模式达到了10 :1的压缩率。这比通常的Gzip 对于HTML网页的3:1或4:1的压缩率要好多了。这是由我们的webtable的行分布方式造成的：来自相同主机的网页被存储在相邻的位置。这使Bentley and McIlroy算法可以识别出来自相同站点的大量固有模式。很多应用程序，不仅仅是webtable，选择的行名称使得类似数据会聚集在一起，因此达到了很好的压缩率。当我们在Bigtable中存储相同值的多个版本时压缩率会更好。

 为了读性能进行缓存

为了提高读性能，tablet服务器使用一个二级缓存。扫描缓存是一个用来缓存由SSTable返回给tablet服务器的key-value对的高级缓存。块缓存是用来缓存从GFS读取的SSTable块的低级缓存。扫描缓存主要用于那些倾向于重复读取相同数据的应用，块缓存则用于那些倾向于从最近读取的数据的邻近位置读取数据的应用(比如顺序读，或者读取一个热点行内的相同局部性群组里的不同列值)。

 Bloom Filters

正如5.3节所描述的，一个读操作需要从组成该tablet的所有SSTable里读取。如果这些SSTable不在内存，就需要很多磁盘操作。通过让用户为某个局部性群组的SSTables指定对应的Bloom filters，可以降低磁盘访问次数。一个Bloom Filter允许我们查询对应的SSTable是否包含某个给定的row/column对的数据。对于特定的应用程序来说，只需要很少的tablet服务器内存来保存Bloom Filter，但可以大大减少读操作所需要的磁盘操作。同时Bloom Filter可以避免那些对于不存在的行列的查找访问磁盘。

 提交日志实现

如果我们为每个tablet的提交日志建立一个独立的日志文件，就会使得大量的文件需要并发写入GFS。由于每个GFS服务器的底层文件系统实现，这些写操作会引起在不同物理日志文件上的大量的磁盘寻道。另外，每个tablet一个日志文件会降低分组提交优化的效率。为了解决这些问题，每个tablet服务器将更新操作append一个日志文件里，将对于不同的tablet的变更放到同一个物理日志文件里。

 使用一个日志为正常操作提供了很明显的性能好处，但是使恢复变复杂了。当一个tablet服务器挂掉后，它负责的那些tablet需要移动到大量其他的tablet服务器上：每个服务器通常都会加载一些该服务器的tablet。为了恢复一个tablet的状态，新的tablet服务器需要通过原来那个tablet服务器的提交日志重新应用这个tablet的变更操作。然而对于这些tablet的变更是混在同一个日志文件里的。一种方法是，每个新的tablet服务器全部读取这个日志文件，然后仅应用那些它需要恢复的tablet的变更操作。然而，在这种模式下，如果失败的那台tablet服务器的tablet被分配到了100个机器，那么这个日志文件就需要读取100次。

 我们通过对提交日志里的entry根据<table,row name,log sequence number>进行排序避免了重复的日志读取。在已排序的输出中，对于一个tablet的所有变更都是连续的，因此可以通过一次的磁盘寻道和顺序读操作就可以完成读取。为了并行化排序，我们将该日志文件划分为64MB大小的段，在不同的tablet服务器上对它们进行排序。排序过程是由master协调进行的，当一个tablet服务器指出它需要从一个日志文件中恢复变更时开始启动。

 将提交日志写入GFS有时候可能因为各种原因导致性能抖动(比如GFS服务器处在繁重的写操作中，或者网络处于拥塞或者重载)。为了避免变更操作受到GFS延时的影响，每个tablet服务器实际上有2个写日志线程，每个写它们各自的日志文件，在同一时间只有一个处在活跃期。如果对于活动日志文件的写性能急剧下降，它就会切换到另一个线程，在提交日志队列中的那些变更操作将由这个新的活动线程负责写入。日志条目里包含序列号，这就使得恢复过程中可以删除那些由于日志切换过程造成的重复条目。

 加速tablet恢复

如果master将tablet从一个tablet服务器移动到另一个，源tablet服务器会首先在该tablet上进行一个minor compaction。这个compaction将会减少在tablet服务器的日志里的uncompacted状态数。当compaction结束后，tablet服务器停止针对该tablet的服务。在彻底卸载该tablet之前，tablet服务器再进行一次minor compaction(通常是很快速的)来消除那些上次minor compaction之后该tablet服务上剩余的uncompacted状态。当第二次的minor compaction结束后，该tablet就可以直接由另一个tablet服务器加载而不需要从日志条目中进行恢复。｛通过这个过程也可以看出，tablet服务器只负责管理memtable和SSTable，对于底层的存储它并不负责，当tablet迁移到另一个服务器时，它在GFS的存储并没有变，变的只是管理它的tablet服务器，而新的tablet服务器也不需要进行数据移动之类的操作，因为它同样可以看到原来的GFS文件。｝

 利用不可变性

除了SSTable缓存，Bigtable系统的各部分通过利用”SSTable生成之后就是不可变的”这个事实也得到了大大的简化。比如我们从SSTable中读取时，不需要对文件系统的访问进行任何同步。这样，在行上的并发控制就可以有效的实现。唯一的可以读写的可变数据结构就是memtable。为了减少在memtable读取时的竞争，我们对每个memtable行进行写时复制，这就允许读写并行处理。

 因为SSTable是不可变的，已删除数据的清除就转换成了对于过时的SSTable的垃圾回收。每个tablet的SSTables会注册在METADATA表中。Master服务器采用“标记-删除”的垃圾回收方式删除SSTable集合中废弃的SSTable。

 最后，SSTable的不可变性使得我们可以快速分割tablet。我们让子tablets共享父tablet的SSTables，而不是为每个子tablet生成新的SSTables。{如果是这样的话，如前所述，一开始只有一个tablet，这样会不会导致SSTable的数目一直未变，只是它的大小一直在上升，但这样会导致它很难一次加载入内存，那么SSTable的分割又是何时发生的呢？}