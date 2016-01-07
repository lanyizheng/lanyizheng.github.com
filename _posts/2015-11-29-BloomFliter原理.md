##一.介绍
BloomFilter是由Bloom在1970年提出的一种多哈希函数映射的快速查找算法，通常应用在一些需要快速判断某个元素是否属于集合，但是并不严格要求100%正确的场合。

##二.解释说明
我们举个例子，假设我们需要写一个(web crawler),由于网络链接错综复杂，在爬行很可能形成''环''，为了避免这种情况的出现，我们需要知道蜘蛛已经访问过的那些URL,给一个URL，怎样知道是否已经访问过呢，有以下几种方案：

- 1. 将访问过的URL保存在数据库中
- 2. HashSet将访问过的URL保存起来，只需要O(1)的代价
- 3. URL经过MD5或SHA-1等单向哈希后再保存到HashSet或者数据库中
- 4. Bit-Map方法，建立一个BitSet，将每一个URL经过一个Hash函数映射到某一位。

其中方法1－3都是将访问过的URL完整保存，方法4只标URL的一个映射位。以上方法在数据量较小的时候都能完美解决问题，当数据量变得很大时，就会出现问题：

- 方法1：数据量变得很大后，关系型数据库查询的效率会变得很低，每一个URL都需要进行一次数据库查询。
- 方法2：太消耗内存，随着URL的增多，内存开销会越来越大。
- 方法3：由于字符串经过MD5处理后的信息摘要长度只有128Bit，SHA-1处理后也只有160Bit，方法3比方法2较少了好几倍内存。
- 方法4：内存消耗较小，但是缺点是单一哈希函数发生冲突的概率太高。

##三.Bloom Filter的算法
BloomFilter算法接近方法4，只是哈希函数采用了多个。

具体过程如下：

创建一个m位的BitSet,先讲所有的初始化为0，然后选择k个不同的哈希函数。第i个哈希函数对字符串str哈希的结果记做h(i,str),其中h(i,str)的范围是0-m-1.
###(1)将字符串映射到BitSet中

对于字符串 str,分别计算h(1,str),h(2,str),...,h(k,str),然后将h(1,str),h(2,str),...,h(k,str)那些位设置为1,如下图所示：

![字符串映射到BitMap](http://img.blog.csdn.net/20151111112100853)

注意：如果一个位置被多次置为1，那么只有第一次会起作用，后面几次没有什么效果。
###(2)判断字符串是否存在
对于字符串str,分别计算h(1,str),h(2,str),...,h(k,str)，然后检查这些对应位置处是否为1，那么有下面几种情况：

- 1）如果所有位都不为1，这str一定没有被记录过。
- 2）如果所有位不全位1，这个str也一定是没有被记录过
- 3）如果所有位全部为1，不能100％肯定字符串被记录过（有可能刚好这个字符串所有位是被其他字符串所对应的），这种错误划分的情况，记作 false positive。
###(3)删除字符串过程
字符串加入了就不能删除了，因为删除会影响到其他字符串。如果一定要删除字符串，可以采用Counting BloomFilter(CBF),这是一个变体，它将每一个Bit位改为一个计数器，这样就可以实现删除字符串的功能。

##四.BloomFilter 的参数选择
**（1）哈希函数的选择**

一个好的哈希函数应该是近似等概率将字符串映射到各个bit上去，选择k个不同的哈希函数比较麻烦，一种简单的实现方式就是选择一个哈希函数，然后送入k个不同的参数。

**（2）Bit数组大小的选择**

哈希函数的个数k选择,位数组大小m,加入字符串数量n的关系可以参考
[](http://pages.cs.wisc.edu/~cao/papers/summary-cache/node8.html)。

##五.HBase中的应用
**目的**：Hbase利用BloomFilter来提高随机读的性能，对于顺序读而言，BloomFilter没有任何作用。

**特点**：BloomFilter是列族级别的配置属性，如果在表中设置了BloomFilter的属性，那么HBase在生成StoreFile的时候会包含一份BloomFilter 结构的数据，称其为MetaBlock,MetaBlock和DataBlock一起由LRUBlockCache维护，所以开启bloomFilter会有一定的存储和内存开销

**原理**：对于某个region 的随机读，HBase会遍历读memstore和storeFile,如果设置了bloomFilter,那么在遍历读storefile的时候，就可以利用bloomfilter忽略掉某些 storefile.

**使用**： 根据不同级别过滤storefile,

- 1)ROW,根据KeyValue中的ROW过滤storefile
*example1*: 假设有两个storefile文件sf1和sf2,
sf1包含kv1(r1 cf:q1 v)、kv2(r2 cf:q1 v）
sf2包含kv3(r3 cf:q1 v)、kv4(r4 cf:q1 v)
如果设置CF属性中的bloomFilter为ROW，那么get(r1)会过滤sf2.

- 2)ROWCOL,根据KeyValue中的Row＋qualifier来过滤StoreFile
*example2* :假设有两个storefile文件sf1和sf2,
sf1包含kv1(r1 cf:q1 v)、kv2(r2 cf:q1 v）
sf2包含kv3(r1 cf:q1 v)、kv4(r2 cf:q1 v)
 如果设置为Row，无论get(r1,q1)还是get(r1,q2),都会读sf1+sf2
 如果设置为RowCol,那么get(r1,q1) 就会过滤sf2
 


