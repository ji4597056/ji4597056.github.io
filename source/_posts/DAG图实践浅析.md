---
title: DAG图实践浅析
categories: 技术
tags: [java,数据结构,算法]
date: 2018/5/7 20:00:00
---

## start

&emsp;&emsp;作为一名热爱算法的程序员,可惜在实际项目开发中很少遇到自己实现复杂数据结构的机会.有幸开发中遇到一次DAG图(有向无环图)的应用实践,很有必要记录一下.

&emsp;&emsp;DAG图的应用场景其实挺多的,这里简单描述下我在项目中的应用场景.需求:实现应用编排发布的功能.应用之间存在依赖关系,比如应用B依赖应用A,应用C依赖应用A,应用D依赖应用B和应用C.此时编排发布的顺序即为应用A->应用B(或C)->应用C(或B)->应用D.是不是觉得这种处理有依赖关系任务并且串行执行的场景特别适合使用DAG图呢.

- - -

<!--more-->

- - -

## content

### 概念

&emsp;&emsp;在图论中,如果一个有向图从任意顶点出发无法经过若干条边回到该点,则这个图是一个有向无环图(DAG图).

&emsp;&emsp;因为有向图中一个点经过两种路线到达另一个点未必形成环,因此有向无环图未必能转化成树,但任何有向树均为有向无环图.

![有向树、DAG图、有向图](http://okzr61x6y.bkt.clouddn.com/%E6%9C%89%E5%90%91%E6%A0%91%E3%80%81DAG%E5%9B%BE%E3%80%81%E6%9C%89%E5%90%91%E5%9B%BE.png)

&emsp;&emsp;在计算机科学领域,有向图的拓扑排序或拓扑排序是其顶点的线性排序,使得对于从顶点u到顶点v的每个有向边uv,u在排序中都在v之前.例如,图形的顶点可以表示要执行的任务,并且边缘可以表示一个任务必须在另一个任务之前执行的约束;在这个应用中,拓扑排序只是一个有效的任务顺序.如果且仅当图形没有定向循环,即如果它是有向无环图(DAG),则拓扑排序是可能的.任何DAG具有至少一个拓扑排序,并且已知这些算法用于在线性时间内构建任何DAG的拓扑排序.

&emsp;&emsp;在图论中,由一个有向无环图的顶点组成的序列,当且仅当满足下列条件时,称为该图的一个拓扑排序.

- 每个顶点出现且只出现一次.
- 若A在序列中排在B的前面,则在图中不存在从B到A的路径.

&emsp;&emsp;也可以定义为:拓扑排序是对有向无环图的顶点的一种排序,它使得如果存在一条从顶点A到顶点B的路径,那么在排序中B出现在A的后面.

### 代码实现

&emsp;&emsp;先定义节点对象,主要字段为每个节点的唯一id,作为拓扑排序中的顶点唯一标识,这里引入了权重字段,权重越高的顶点在拓扑排序中更靠前.

```java
public class GraphNode<T> {

	// 唯一id
    private int id;

    private T content;
	
	// 权重
    private int weight;

    public GraphNode(int id, T content, int weight) {
        this.id = id;
        this.content = content;
        this.weight = weight;
    }

    public GraphNode(int id, T content){
        this.id = id;
        this.content = content;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public T getContent() {
        return content;
    }

    public void setContent(T content) {
        this.content = content;
    }

    public int getWeight() {
        return weight;
    }

    public void setWeight(int weight) {
        this.weight = weight;
    }
}
```

&emsp;&emsp;拓扑排序的过程：

- 先将入度为0的顶点放在队列中(为了保证权重大的顶点具有较高的优先级，所以使用优先队列).
- 当队列不空时,出队列一个顶点v存入结果集合,然后更新与顶点v邻接的顶点的入度(减一).只要有一个顶点的入度降为0,则将之入队列.
- 一直执行上述步骤,直到队列为空.如果最后队列不为空,说明该图有环.

```java
public class DirectedGraph<T> {

    private Map<Integer, Vertex> graph;

    public DirectedGraph() {
        this.graph = new HashMap<>();
    }

    // 拓扑排序
    public List<Vertex> topoSort() {
        List<Vertex> result = new ArrayList<>();
        if (traverse(result::add)) {
            throw new RuntimeException("Graph has circle!");
        }
        return result;
    }

    // 判断是否有环
    public boolean hasCircle() {
        return traverse(null);
    }

    // 清空
    public void clear() {
        graph.clear();
    }

    // 遍历,返回是否有环,true=有环,false=无环
    private boolean traverse(Consumer<Vertex> consumer) {
        int count = 0;
        Queue<Vertex> queue = new PriorityQueue<>();
        // 扫描所有的顶点,将入度为0的顶点入队列
        Collection<Vertex> vertexs = getGraphValues();
        for (Vertex vertex : vertexs) {
            if (vertex.inDegree == 0) {
                queue.offer(vertex);
            }
        }
        Vertex v;
        while ((v = queue.poll()) != null) {
            if (consumer != null) {
                consumer.accept(v);
            }
            count++;
            for (Edge e : v.adjEdges) {
                if (--e.endVertex.inDegree == 0) {
                    queue.offer(e.endVertex);
                }
            }
        }
        return count != graph.size();
    }

    // 图中增加有向边
    public void add(GraphNode<T> from, GraphNode<T> to) {
        assert from != null;
        assert to != null;
        Vertex tVertex;
        Vertex fVertex;
        // update to-vertex
        if (!graph.containsKey(to.getId())) {
            tVertex = new Vertex(to.getId(), to.getWeight(), to.getContent());
            graph.put(tVertex.id, tVertex);
        }
        tVertex = graph.get(to.getId());
        tVertex.inDegree++;

        // update from-vertex
        if (!graph.containsKey(from.getId())) {
            fVertex = new Vertex(from.getId(), from.getWeight(), from.getContent());
            graph.put(fVertex.id, fVertex);
        }
        fVertex = graph.get(from.getId());
        addEdge(fVertex, tVertex);
    }

    // 图中增加顶点
    public void add(GraphNode<T> node) {
        assert node != null;
        if (!graph.containsKey(node.getId())) {
            graph.put(node.getId(), new Vertex(node.getId(), node.getWeight(), node.getContent()));
        }
    }

    // 顶点增加边
    private void addEdge(Vertex start, Vertex end) {
        Edge edge = new Edge(end);
        List<Edge> adjEdges = start.adjEdges;
        if (adjEdges == null) {
            adjEdges = new ArrayList<>();
        }
        adjEdges.add(edge);
    }

    // 获取图的浅拷贝
    private List<Vertex> getGraphValues() {
        return graph.values().stream()
            .collect(ArrayList::new, (vertices, vertex) -> {
                    try {
                        vertices.add(vertex.clone());
                    } catch (Exception e) {
                        throw new RuntimeException("Copy graph values error!");
                    }
                },
                ArrayList::addAll);
    }

    // 定点
    public class Vertex implements Cloneable, Comparable<Vertex> {

        // 顶点标识
        private int id;

        // 边
        private List<Edge> adjEdges;

        // 入度
        private int inDegree;

        // 权重
        private int weight;

        // 内容
        private T content;

        public Vertex(int id, T content) {
            this.id = id;
            this.inDegree = 0;
            this.weight = 0;
            this.adjEdges = new LinkedList<>();
            this.content = content;
        }

        public Vertex(int id, int weight, T content) {
            this.id = id;
            this.inDegree = 0;
            this.weight = weight;
            this.adjEdges = new LinkedList<>();
            this.content = content;
        }

        public int getId() {
            return id;
        }

        public List<Edge> getAdjEdges() {
            return adjEdges;
        }

        public int getInDegree() {
            return inDegree;
        }

        public int getWeight() {
            return weight;
        }

        public T getContent() {
            return content;
        }

        @Override
        public Vertex clone() throws CloneNotSupportedException {
            return (Vertex) super.clone();
        }

        @Override
        public int compareTo(Vertex o) {
            return o.weight - this.weight;
        }
    }

    // 边
    public class Edge {

        // 指向定点
        private Vertex endVertex;

        public Edge(Vertex endVertex) {
            this.endVertex = endVertex;
        }

        public Vertex getEndVertex() {
            return endVertex;
        }
    }
}
```

&emsp;&emsp;测试结果如下：
```java
public class DirectedGraphTest {

    private DirectedGraph<String> g1 = new DirectedGraph<>();
    private DirectedGraph<String> g2 = new DirectedGraph<>();
    private DirectedGraph<String> g3 = new DirectedGraph<>();

    @Before
    public void build() {
        buidG1();
        buidG2();
        buidG3();
    }

    // build g1(no weight)
    //   --> b -->
    // a           d  |  e  | f --> g
    //   --> c -->
    private void buidG1() {
        GraphNode<String> a = new GraphNode<>(1, "a");
        GraphNode<String> b = new GraphNode<>(2, "b");
        GraphNode<String> c = new GraphNode<>(3, "c");
        GraphNode<String> d = new GraphNode<>(4, "d");
        GraphNode<String> e = new GraphNode<>(5, "e");
        GraphNode<String> f = new GraphNode<>(6, "f");
        GraphNode<String> g = new GraphNode<>(7, "g");
        g1.add(a, b);
        g1.add(a, c);
        g1.add(b, d);
        g1.add(c, d);
        g1.add(e);
        g1.add(f, g);
    }

    // build g2(weight)
    //      --> b(1) -->
    // a(3)              d(1)  |  e(1)  | f(2) --> g(3)
    //      --> c(2) -->
    private void buidG2() {
        GraphNode<String> a = new GraphNode<>(1, "a", 3);
        GraphNode<String> b = new GraphNode<>(2, "b", 1);
        GraphNode<String> c = new GraphNode<>(3, "c", 2);
        GraphNode<String> d = new GraphNode<>(4, "d", 1);
        GraphNode<String> e = new GraphNode<>(5, "e", 1);
        GraphNode<String> f = new GraphNode<>(6, "f", 2);
        GraphNode<String> g = new GraphNode<>(7, "g", 3);
        g2.add(a, b);
        g2.add(a, c);
        g2.add(b, d);
        g2.add(c, d);
        g2.add(e);
        g2.add(f, g);
    }

    // build g3(has circle)
    //   --> b -->
    // a           d --> a  |  e  | f --> g
    //   --> c -->
    private void buidG3() {
        GraphNode<String> a = new GraphNode<>(1, "a");
        GraphNode<String> b = new GraphNode<>(2, "b");
        GraphNode<String> c = new GraphNode<>(3, "c");
        GraphNode<String> d = new GraphNode<>(4, "d");
        GraphNode<String> e = new GraphNode<>(5, "e");
        GraphNode<String> f = new GraphNode<>(6, "f");
        GraphNode<String> g = new GraphNode<>(7, "g");
        g3.add(a, b);
        g3.add(a, c);
        g3.add(b, d);
        g3.add(c, d);
        g3.add(d, a);
        g3.add(e);
        g3.add(f, g);
    }
	
	// 测试结果：
	//	==========print g1=========
	//	a f c g b e d 
	//	==========print g2=========
	//	a f g c b e d
    @Test
    public void topoSort() {
        System.out.println("==========print g1=========");
        g1.topoSort().forEach(vertex -> System.out.print(vertex.getContent() + " "));
        System.out.println();
        System.out.println("==========print g2=========");
        g2.topoSort().forEach(vertex -> System.out.print(vertex.getContent() + " "));
    }

    @Test
    public void hasCircle() {
        Assert.assertTrue(!g1.hasCircle());
        Assert.assertTrue(!g2.hasCircle());
        Assert.assertTrue(g3.hasCircle());
    }
}
```

### end

&emsp;&emsp;最近面试360大数据部门惜败而归,虽然没有相关工作经验,但确实是我非常感兴趣的方向.工作方向尝试转型失败,让我痛心疾首,也让我愈加重视算法和数据结构,哎,言止于此.
