---
title: 一致性hash算法探究
categories: 技术
tags: [算法,分布式]
date: 2018/3/5 20:00:00
---

## start

&emsp;&emsp;前年写过网关模块设计,涉及到**针对有状态服务**请求转发负载均衡方案时,提供了一种`hash`负载均衡方案:`每个uri进行hash计算得到一个数值，这个数值除以整个节点数量取余数得到该uri对应节点在节点集合(list)的index.(取模)`这个方法很简单,但是缺点也很明显:`如果对节点集合进行增加/删除操作,节点数量就会随之改变,先前uri和节点之间的对应关系便会打乱`.最近了解到`一致性hash算法`就是专门解决这种问题,让我们来深入探究一下吧!

- - -

<!--more-->

- - -

## content

### 实际场景

&emsp;&emsp;一切问题必须以实际应用场景作为出发点,首先这里提出两个在日常开发中可能会遇到的问题场景.

- 假设有一种静态资源,例如视频、网页、大型log文件等,只有用户第一次请求时才加载到系统内存中.我们希望以后每次对相关资源的请求都送达到同一个节点.这样可以最大限度的利用整个集群的能力,这也是CDN的基本思路.
- 一个有状态服务,持有大量有状态session,且session不断的被更新和查询.一般情况,分布式环境下这种session会存储在redis中,但是由于session被更新和查询的频率非常频繁,为了提高系统性能和网络开销,可以将系统内存作为一级缓存,redis作为二级缓存.当内存缓存命中失败时,才会去redis缓存中读取session并写入内存缓存.当对内存缓存更新session时,异步备份数据到redis缓存.为了提高内存缓存的命中率,则需要用户每次对session进行读取和更新操作请求到服务的同一节点.

&emsp;&emsp;基于上述场景,网关发出的请求会负载到该服务的各个节点,随之而来的问题就是针对该用户的同一份数据可能被存在不同服务节点上造成数据冗余.要解决这个问题只需要保证针对相同`key`(例如用户请求ip,用户登录后token)的请求都发送到该服务相同的节点上.如果使用简单的hash算法,服务地址列表变更(添加或移除)时,`key`的哈希结果的变化会随之变化,所有的映射关系都可能发生变化.即不满足`单调性`.

&emsp;&emsp;`单调性(Monotonicity)`是指如果已经有一些内容通过哈希分派到了相应的缓冲区中,又有新的缓冲区加入到系统中,那么哈希的结果应能够保证原有已分配的内容可以被映射到新的缓冲区中去,而不会被映射到旧的缓冲集合中的其他缓冲区.如果有缓存区从系统中移除,那么该缓冲区已分配的内容会被映射到缓冲区集合中的其他缓冲区,不会影响其他缓冲区下分配的内容.

&emsp;&emsp;在分布式环境下,每个服务节点都有可能失效,并且新的节点很可能动态的增加进来,如何保证当系统的节点数目发生变化时仍然能够对外提供良好的服务,这是值得考虑的,尤其是在设计分布式缓存系统时,如果某台服务器失效,对于整个系统来说如果不采用合适的算法来保证一致性,那么缓存于系统中的所有数据都可能会失效.

### 原理分析

&emsp;&emsp;为了解决上述场景,让我们一起来探究`一致性hash算法`吧!

&emsp;&emsp;具体原理:`先构造一个长度为2^32的整数环(这个环被称为一致性Hash环),根据节点名称的Hash值(其分布为[0, 2^32-1])将服务器节点放置在这个Hash环上,然后根据数据的Key值计算得到其Hash值(其分布也为[0, 2^32-1]),接着在Hash环上顺时针查找距离这个Key值的Hash值最近的服务器节点,完成Key到服务器的映射查找.`

- 如图示,计算`node1`、`node2`、`node3`、`node4`四个节点的hash值(0~2^32-1)映射到环中(一般情况下对机器的hash计算是采用机器的IP或者机器唯一的别名作为输入值).将若干对象计算hash值(0~2^32-1)映射到同一个环中,每个对象对应的节点为其顺时针最近的一个hash值对应的节点.
![一致性hash算法图示一](http://okzr61x6y.bkt.clouddn.com/%E4%B8%80%E8%87%B4%E6%80%A7hash%E7%AE%97%E6%B3%95%E5%9B%BE%E7%A4%BA%E4%B8%80.png)

- 如图示,当添加`node5`,计算其hash值映射到环中,其中三个对象会受到影响,对应节点由`node4`变为`node5`,其余对象不受影响.
![一致性hash算法图示二](http://okzr61x6y.bkt.clouddn.com/%E4%B8%80%E8%87%B4%E6%80%A7hash%E7%AE%97%E6%B3%95%E5%9B%BE%E7%A4%BA%E4%BA%8C.png)

&emsp;&emsp;上述设计即可保证hash算法的`单调性`.但是依然存在问题,假设遇到极端情况:
&emsp;&emsp;`hash(node1)=1`,`hash(node2)=100`,`hash(node3)=1000`,`hash(node4)=10000`.
&emsp;&emsp;此时,对象对应的节点极有可能为`node1`,因为`node4`与`node1`hash值之间的区间范围明显大于其他节点之间的hash值范围.这个例子比较极端,旨在说明由于节点数量不多,会出现某几个两两节点之间的区间明显大于其他两两节点之间的区间,从而导致对象分布不均匀,即不满足`平衡性`.

&emsp;&emsp;`平衡性(Balance)`是指哈希的结果能够尽可能均匀分布到所有的缓冲中去,这样可以使得所有的缓冲空间都得到利用.

&emsp;&emsp;通过分析,发生上述不满足`平衡性`的主要原因为节点数量太少,而hash区间太大(2^32).为了满足`平衡性`,`一致性哈希算`法引入了虚拟节点机制,即对每一个服务节点计算多个哈希,每个计算结果位置都放置一个此服务节点,称为虚拟节点.具体做法可以在节点key(服务器ip或主机名)的后面增加编号来实现.例如图示,有2个节点(`nodeA`、`nodeB`),可以为每台服务器计算三个虚拟节点,于是可以分别计算`Node A#1`、`Node A#2`、`Node A#3`、`Node B#1`、`Node B#2`、`Node B#3`的hash值.
![一致性hash算法图示三](http://okzr61x6y.bkt.clouddn.com/%E4%B8%80%E8%87%B4%E6%80%A7hash%E7%AE%97%E6%B3%95%E5%9B%BE%E7%A4%BA%E4%B8%89.png)

&emsp;&emsp;不难分析出,当实际节点对应的虚拟节点数越多,`一致性hash算法`将会愈加满足`平衡性`.

### 算法实现

- **Hash算法的选择:**
计算hash值的算法有很多,比如`CRC32_HASH`、`FNV1_32_HASH`、`KETAMA_HASH`等,不管选择何种hash算法,需要保证对应同一个输入的值,要有相同的输出,并且尽可能保证hash算法速度较快.看过网上很多算法的选择,推荐`FNV1_32_HASH`,对于这些算法的性能比较我没做过调研,这里仅贴出`FNV1_32_HASH`的代码实现.

```java
public interface Hash {

    int getHash(String key);
}

public class FnvHash implements Hash {

    @Override
    public int getHash(String key) {
        final int p = 16777619;
        int hash = (int) 2166136261L;
        for (int i = 0; i < key.length(); i++) {
            hash = (hash ^ key.charAt(i)) * p;
        }
        hash += hash << 13;
        hash ^= hash >> 7;
        hash += hash << 3;
        hash ^= hash >> 17;
        hash += hash << 5;
        // hash >= 0
        if (hash < 0) {
            hash = Math.abs(hash);
        }
        return hash;
    }
}
```

- **数据结构选型:**
可能我们首先想到用`List`数据结构去存储虚拟服务节点,这里需要将节点集合排序,选择性能较好的快速排序,时间复杂度为`O(n*lgn)`,然后遍历该集合找出hash值大于路由结点的hash值的第一个虚拟服务节点,时间复杂度为`O(lgn)`,最后根据映射规则得到该虚拟服务节点对应的真实服务节点.
比起上述方案,使用二分查找法能够更加快速地查询到虚拟服务节点,时间复杂度为`O(lgn)`,所以我们可以选择`TreeMap`来存储虚拟服务节点.`TreeMap`基于二叉平衡树原理(红黑树)存储有序的数据,因此查询性能非常好,但与此同时,插入性能相对较差.但在这种场景下,查询的频率是远远高于插入频率的.因此,这里选择`TreeMap`作为数据存储结构比较合理.

- **代码实现:**

&emsp;&emsp;源码见[算法应用之一致性hash算法](https://github.com/ji4597056/algorithm-application)

```java
public class ConsistentHashingWithVirtualNode {

    /**
     * virtual server node suffix
     */
    private static final String VIRTUAL_SERVER_NODE_SUFFIX = "##";

    /**
     * virtual server nodes,key:node hash/value:node name
     */
    private static final SortedMap<Integer, String> VIRTUAL_SERVER_NODES = new TreeMap<>();

    /**
     * real server nodes
     */
    private static final List<String> REAL_SERVER_NODES = new LinkedList<>();

    /**
     * hash function
     */
    private final Hash HASH;

    /**
     * virtual nodes numbers
     */
    private final int virtualNodesNum;

    public ConsistentHashingWithVirtualNode(List<String> realNodes, int virtualNodeNum, Hash hash) {
        REAL_SERVER_NODES.addAll(realNodes);
        this.virtualNodesNum = virtualNodeNum;
        HASH = hash;
        init();
    }

    public ConsistentHashingWithVirtualNode(List<String> realNodes, int virtualNodeNum) {
        this(realNodes, virtualNodeNum, new FnvHash());
    }

    /**
     * put virtual server nodes
     */
    private void init() {
        REAL_SERVER_NODES.forEach(node -> IntStream.range(0, virtualNodesNum)
            .forEach(index -> VIRTUAL_SERVER_NODES
                .put(HASH.getHash(getVirtualNodeKey(node, index)),
                    getVirtualNodeKey(node, index))));
    }

    /**
     * get server from node
     *
     * @param node node
     * @return real node
     */
    public String getServer(String node) {
        int hash = HASH.getHash(node);
        SortedMap<Integer, String> subMap = VIRTUAL_SERVER_NODES.tailMap(hash);
        String virtualNode;
        if (subMap.size() == 0) {
            virtualNode = VIRTUAL_SERVER_NODES.get(VIRTUAL_SERVER_NODES.firstKey());
        } else {
            virtualNode = subMap.get(subMap.firstKey());
        }
        return virtualNode.substring(0, virtualNode.indexOf(VIRTUAL_SERVER_NODE_SUFFIX));
    }

    /**
     * add server node
     *
     * @param serverNode server node
     */
    public void addServerNode(String serverNode) {
        REAL_SERVER_NODES.add(serverNode);
        IntStream.range(0, virtualNodesNum).forEach(index -> VIRTUAL_SERVER_NODES
            .put(HASH.getHash(getVirtualNodeKey(serverNode, index)),
                getVirtualNodeKey(serverNode, index)));
    }

    /**
     * remove server node
     *
     * @param serverNode server node
     */
    public void removeServerNode(String serverNode) {
        REAL_SERVER_NODES.remove(serverNode);
        IntStream.range(0, virtualNodesNum).forEach(index -> VIRTUAL_SERVER_NODES
            .remove(HASH.getHash(getVirtualNodeKey(serverNode, index))));
    }

    /**
     * get real server nodes
     *
     * @return real server nodes
     */
    public List<String> getRealServerNodes() {
        return Collections.unmodifiableList(REAL_SERVER_NODES);
    }

    /**
     * get virtual server nodes
     *
     * @return virtual server nodes
     */
    public Map<Integer, String> getVirtualServerNodes() {
        return Collections.unmodifiableMap(VIRTUAL_SERVER_NODES);
    }

    /**
     * get virtual node key
     *
     * @param realNodeKey real node
     * @param index index
     * @return virtual node
     */
    private String getVirtualNodeKey(String realNodeKey, int index) {
        return realNodeKey + VIRTUAL_SERVER_NODE_SUFFIX + index;
    }
}
```

- **代码测试:**
这里主要测试设置不同虚拟节点数对于真实服务节点选择的影响,可以看出和我们预期一样,随着虚拟节点数设置越高,真实服务节点的选择也越趋于平均,即满足`平衡性`.

```java
public class ConsistentHashingWithVirtualNodeTest {

    private List<String> realServerNodes = Lists
        .newArrayList("172.0.0.1", "172.0.0.2", "172.0.0.3", "172.0.0.4", "172.0.0.5");

    private Random random = new Random();

    @Test
    public void testGetServer() {
        printGetServer(1, 500000);
        printGetServer(10, 500000);
        printGetServer(50, 500000);
        printGetServer(100, 500000);
    }

    private void printGetServer(int virtualNodeNum, int nodeNum) {
        System.out.println("==================================");
        System.out.println("virtual node num:" + virtualNodeNum + ",node num:" + nodeNum);
        ConsistentHashingWithVirtualNode consistentHashing = new ConsistentHashingWithVirtualNode(
            realServerNodes, virtualNodeNum);
        Map<String, Integer> result = new HashMap<>();
        IntStream.range(0, nodeNum).forEach(i -> {
            String serverNode = consistentHashing.getServer(getRandomNode());
            if (result.get(serverNode) == null) {
                result.put(serverNode, 1);
            } else {
                result.put(serverNode, result.get(serverNode) + 1);
            }
        });
        result.forEach((key, count) -> System.out.println(key + ":" + count));
    }

    private String getRandomNode() {
        int max = 256;
        return random.nextInt(max) + "." + random.nextInt(max) + "." + random.nextInt(max) + random
            .nextInt(max);
    }
}
```
```
==================================
virtual node num:1,node num:500000
172.0.0.1:47336
172.0.0.4:94302
172.0.0.5:53151
172.0.0.2:91794
172.0.0.3:213417
==================================
virtual node num:10,node num:500000
172.0.0.1:84303
172.0.0.4:106161
172.0.0.5:142935
172.0.0.2:81781
172.0.0.3:84820
==================================
virtual node num:50,node num:500000
172.0.0.1:102522
172.0.0.4:101109
172.0.0.5:87228
172.0.0.2:121043
172.0.0.3:88098
==================================
virtual node num:100,node num:500000
172.0.0.1:96348
172.0.0.4:96249
172.0.0.5:93649
172.0.0.2:114121
172.0.0.3:99633
```

### end

&emsp;&emsp;`一致性hash算法`主要应用在大规模高可用性的分布式存储上,尤其是KV存储上面,比如memcaced,redis集群,一些负载均衡中间件如nginx也提供该功能模块的支持.最后需要强调的是,`一致性hash算法`主要是针对有状态的服务,若是无状态的服务,选择该算法没有太大意义,反而相对其他负载均衡算法性能可能更差.
