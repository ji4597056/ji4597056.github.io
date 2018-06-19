---
title: CompletableFuture实践
categories: 技术
tags: [java]
date: 2017/12/12 20:00:00
---

## start

&emsp;&emsp;java8已经用很久了,`CompletableFuture`使用过一段时间,但是一直理解的不够深入,最近项目中由于需要对远程调用进行异步处理,所以选择使用`CompletableFuture`,在使用过程中发现一些问题,虽然`java8 in action`翻了几遍,依然没有解决心中的疑惑,于是决定深入研究一下`CompletableFuture`的用法.

- - -

<!--more-->

- - -

## content

### 说明

&emsp;&emsp;`CompletableFuture`继承了`Future`和`CompletionStage`接口,`Future`不多介绍,不太理解`Future`设计原理的朋友可以看下[async-method-invocation](https://github.com/iluwatar/java-design-patterns/tree/master/async-method-invocation).`CompletionStage`接口下定义了我们常用的`CompletableFuture`方法,这篇文章主要针对几个容易混淆的方法以及结合实际场景介绍`CompletableFuture`的常用使用方式.

### 场景

&emsp;&emsp;场景介绍:初始化文本(`init`:sleep 1s)、获取文本的第一个单词(`getFirst`:sleep 1s)、获取文本的最后一个单词(`getLast`:sleep 2s)、拼接单词(`concat`)、拼接单词(`concatForSeconds`:sleep 1s).

```java
    private static String text;	//文本
    private static long start;	//起始时间

    @Before
    public void before() {
        text = "A B C D F";
        start = System.currentTimeMillis();
    }

    @After
    public void after() {
        System.out.println("cost time:" + (System.currentTimeMillis() - start) + "(ms)");
    }

    public String init(String text) {
        System.out.println("'init' method thread:" + Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(1L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return text;
    }

    public String getFirst(String text) {
        System.out.println("'getFirst' method thread:" + Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(1L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return text.substring(0, text.indexOf(" "));
    }

    public String getLast(String text) {
        System.out.println("'getLast' method thread:" + Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(2L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return text.substring(text.lastIndexOf(" ") + 1);
    }

    public String concat(String first, String last) {
        System.out.println("'concat' method thread:" + Thread.currentThread().getName());
        return first + "/" + last;
    }

    public String concatForSeconds(String first, String last) {
        System.out.println("'concatForSeconds' method thread:" + Thread.currentThread().getName());
        try {
            TimeUnit.SECONDS.sleep(1L);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return first + "/" + last;
    }
```

**业务一:**初始化文本(1s),获取文本的第一个单词(1s)和最后一个单词(2s)并进行拼接,打印.

- 写法一: `thenApply`->`thenApply`->`thenCombine`->`thenAccept`
```java
    @Test
    public void test() {
        CompletableFuture<String> initFuture = CompletableFuture.supplyAsync(() -> init(text));
        CompletableFuture<String> firstFuture = initFuture.thenApply(this::getFirst);
        CompletableFuture<String> lastFuture = initFuture.thenApply(this::getLast);
        firstFuture.thenCombine(lastFuture, this::concat).thenAccept(System.out::println).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-1
        'concat' method thread:ForkJoinPool.commonPool-worker-1
        A/F
        cost time:4130(ms)
     */
```

- 写法二: `thenApplyAsync`->`thenApplyAsync`->`thenCombine`->`thenAccept`
```java
    @Test
    public void test() {
        CompletableFuture<String> initFuture = CompletableFuture.supplyAsync(() -> init(text));
        CompletableFuture<String> firstFuture = initFuture.thenApplyAsync(this::getFirst);
        CompletableFuture<String> lastFuture = initFuture.thenApplyAsync(this::getLast);
        firstFuture.thenCombine(lastFuture, this::concat).thenAccept(System.out::println).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-2
        'concat' method thread:ForkJoinPool.commonPool-worker-1
        A/F
        cost time:3155(ms)
     */
```

- 写法三: `thenCompose`->`thenCompose`->`thenCombine`->`thenAccept`
```java
    @Test
    public void test() {
        CompletableFuture<String> initFuture = CompletableFuture.supplyAsync(() -> init(text));
        CompletableFuture<String> firstFuture = initFuture
            .thenCompose(s -> CompletableFuture.supplyAsync(() -> this.getFirst(s)));
        CompletableFuture<String> lastFuture = initFuture
            .thenCompose(s -> CompletableFuture.supplyAsync(() -> this.getLast(s)));
        firstFuture.thenCombine(lastFuture, this::concat).thenAccept(System.out::println).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-2
        'concat' method thread:ForkJoinPool.commonPool-worker-1
        A/F
        cost time:3238(ms)
     */
```

- 写法四: `thenCompose`(内嵌`thenCombine`)->`thenAccept`
```java
    @Test
    public void test() {
        CompletableFuture.supplyAsync(() -> init(text))
            .thenCompose(s -> CompletableFuture.supplyAsync(() -> getFirst(s))
                .thenCombine(CompletableFuture.supplyAsync(() -> getLast(s)), this::concat))
            .thenAccept(System.out::println).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-2
        'concat' method thread:ForkJoinPool.commonPool-worker-2
        A/F
        cost time:3155(ms)
     */
```

- 写法五: `thenCompose`->`thenAccept`
```java
    @Test
    public void test() {
        CompletableFuture.supplyAsync(() -> init(text))
            .thenCompose(s -> CompletableFuture.supplyAsync(() -> concat(getFirst(s), getLast(s))))
            .thenAccept(System.out::println).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-1
        'concat' method thread:ForkJoinPool.commonPool-worker-1
        A/F
        cost time:4190(ms)
     */
```

**分析:**
&emsp;&emsp;`thenApply`为同步操作.`thenApplyAsync`和`thenCompose`为异步操作.不同的是`thenApply`、`thenApplyAsync`传入的`Function()`返回值为`T`,而`thenCompose`传入的`Function()`返回值是`CompletableFuture<T>`,两者的区别可以类比于`stream`中`map`和`flatMap`.
&emsp;&emsp;`thenApply`、`thenApplyAsync`、`thenCompose`三个方法都是对前一个`CompletableFuture`的结果作为入参进行后续操作.那么该如何区别使用这三种方法呢?`stackoverflow`上有大神解答:
![thenApply和thenCompose区别](http://okzr61x6y.bkt.clouddn.com/CompletableFuture-%E4%B8%9A%E5%8A%A11.png)
&emsp;&emsp;如上所示,简而言之,同步操作使用`thenApply`,异步操作使用`thenCompose`.不过这样回答也太简单.从写法结果而言,`写法二`、`写法四`是最优的写法(`写法三`虽然时间性能上达到要求,但是写法较冗余,不推荐),似乎`thenApplyAsync`和`thenCompose`可互为替代,当对之前`CompletableFuture`的结果进行一个复杂的后序操作,推荐使用`thenCompose`.

**业务二:**初始化文本(1s),获取文本的第一个单词(1s)和最后一个单词(2s)并进行拼接(1s),获取文本的最后一个单词(1s)和第一个单词(2s)并进行拼接(1s),打印2种拼接结果.

- 写法一: `thenApplyAsync`->`thenApplyAsync`->`thenCombineAsync`->`thenCombineAsync`->`thenCombine`
```java
    @Test
    public void test() {
        CompletableFuture<String> initFuture = CompletableFuture.supplyAsync(() -> init(text));
        CompletableFuture<String> firstFuture = initFuture.thenApplyAsync(this::getFirst);
        CompletableFuture<String> lastFuture = initFuture.thenApplyAsync(this::getLast);
        CompletableFuture<String> combineFuture1 = firstFuture
            .thenCombineAsync(lastFuture, this::concatForSeconds);
        CompletableFuture<String> combineFuture2 = lastFuture
            .thenCombineAsync(firstFuture, this::concatForSeconds);
        combineFuture1.thenCombine(combineFuture2, (s1, s2) -> {
            System.out.println("s1:" + s1 + ",s2:" + s2);
            return null;
        }).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-2
        'concatForSeconds' method thread:ForkJoinPool.commonPool-worker-1
        'concatForSeconds' method thread:ForkJoinPool.commonPool-worker-2
        s1:A/F,s2:F/A
        cost time:4137(ms)
     */
```

- 写法二: `thenApplyAsync`->`thenApplyAsync`->`thenCombine`->`thenCombine`->`thenCombine`
```java
    @Test
    public void test() {
        CompletableFuture<String> initFuture = CompletableFuture.supplyAsync(() -> init(text));
        CompletableFuture<String> firstFuture = initFuture.thenApplyAsync(this::getFirst);
        CompletableFuture<String> lastFuture = initFuture.thenApplyAsync(this::getLast);
        CompletableFuture<String> combineFuture1 = firstFuture
            .thenCombine(lastFuture, this::concatForSeconds);
        CompletableFuture<String> combineFuture2 = lastFuture
            .thenCombine(firstFuture, this::concatForSeconds);
        combineFuture1.thenCombine(combineFuture2, (s1, s2) -> {
            System.out.println("s1:" + s1 + ",s2:" + s2);
            return null;
        }).join();
    }
    /*
        'init' method thread:ForkJoinPool.commonPool-worker-1
        'getLast' method thread:ForkJoinPool.commonPool-worker-1
        'getFirst' method thread:ForkJoinPool.commonPool-worker-2
        'concatForSeconds' method thread:ForkJoinPool.commonPool-worker-1
        'concatForSeconds' method thread:ForkJoinPool.commonPool-worker-1
        s1:A/F,s2:F/A
        cost time:5154(ms)
     */
```

**分析:**
&emsp;&emsp;上述两种写法的区别仅为使用`thenCombine`/`thenCombineAsync`将`firstFuture`和`lastFuture`合并.`CompletableFuture`提供的操作接口中有很多`then***`,`then***Async`,那么`Async`后缀到底应该在何时使用呢?
&emsp;&emsp;综合上述两种业务,其实当对`CompletableFuture`的结果只进行一次后续处理,用`then***`和`then***Async`在性能上是没有区别的,甚至`then***Async`由于异步的原因需要将数据在线程间复制,性能略有损失(可忽略).但是当`CompletableFuture`的结果进行多个后续处理(如在第一种业务中,对初始化的`text`需要`getLast(text)`和`getFirst(text)`两种操作),如果使用`then***`,则多个后续操作会同步执行完成(一个操作完成再执行下一个),而使用`then***Async`,则多个后序操作会异步同时执行.
&emsp;&emsp;分析到这里,应该能明白`thenApply`、`thenApplyAsync`、`thenCompose`的使用区别,可以根据对先前`CompletableFuture`的结果进行的后续处理是否应该同步(非阻塞操作)/异步(阻塞操作),从而选用合适的方法.

**业务三:**初始化10个文本,在每个文本中,获取第一个单词(1s)和获取最后一个单词(2s)并进行拼接,打印.

&emsp;&emsp;当对`CompletableFuture`执行`runAsync()`和`supplyAsync()`方法时,可以传递线程池参数,之后所有的异步操作都会从线程池中选取空闲线程执行操作,若未传递线程池参数,会使用默认线程池`ForkJoinPool.commonPool()`,该线程池的线程数为`Runtime.
getRuntime().availableProcessors()`的返回值.对于自定义线程池该如何配置线程数,可以参考`Java并发编程实战`中的公式:`Nthreads = Ncpu * Ucpu * (1 + W/C)`,其中`Ncpu`表示处理器的核数(`Runtime.getRuntime().availableProcessors()`),`Ucpu`是期望的cpu利用率(介于0和1之间),`W/C`是等待时间和计算时间的比率(内核态时间与用户态时间之比),例如:在4核的操作系统下,执行某操作99%的时间都在等待IO,所以估算出的W/C比率为100.这意味着如果期望的CPU利用率是100%,需要创建一个拥有400个线程的线程池,当然在实际操作中线程池的大小大于实际并发使用的线程数反而是一种浪费.

- 辅助方法:
```java
	// 获取文本列表(非阻塞)
	private List<String> getTexts(int size) {
        List<String> origin = Lists.newArrayList("A", "B", "C", "D", "E", "F");
        List<String> texts = Lists.newArrayList();
        for (int i = 0; i < size; i++) {
            texts.add(origin.stream().collect(Collectors.joining(" ")));
            Collections.shuffle(texts);
        }
        return texts;
    }

	// 线程池
	private static Executor threadPool = Executors.newFixedThreadPool(20);
```

- 写法一:
```java
    @Test
    public void test() {
        List<String> texts = getTexts(10);
        texts.stream().map(
            text -> CompletableFuture.supplyAsync(() -> getFirst(text), threadPool)
                .thenCombine(CompletableFuture.supplyAsync(() -> getLast(text), threadPool),
                   this::concat).thenAccept(System.out::println)).map(CompletableFuture::join)
            .collect(Collectors.toList());
    }
    /*
        cost time:20213(ms)    
     */
```

- 写法二:
```java
    @Test
    public void test() {
        List<String> texts = getTexts(10);
        CompletableFuture<Void>[] futures = texts.stream()
            .map(text -> CompletableFuture.supplyAsync(() -> getFirst(text), threadPool)
                .thenCombine(CompletableFuture.supplyAsync(() -> getLast(text), threadPool),
                    this::concat)
                .thenAccept(System.out::println)).toArray(CompletableFuture[]::new);
        CompletableFuture.allOf(futures).exceptionally(throwable -> {
            System.out.println("throwable message:" + throwable.getMessage());
            return null;
        }).join();
    }
    /*
        cost time:2206(ms)
     */
```

- 写法三:
```java
    @Test
    public void test() {
        List<String> texts = getTexts(10);
        List<CompletableFuture<String>> futures = texts.stream().map(
            text -> CompletableFuture.supplyAsync(() -> getFirst(text), threadPool)
                .thenCombine(CompletableFuture.supplyAsync(() -> getLast(text), threadPool),
                    this::concat)).collect(Collectors.toList());
        futures.stream().map(CompletableFuture::join).collect(Collectors.toList())
            .forEach(System.out::println);
    }
    /*
        cost time:2221(ms)
     */
```

**分析:**可以明显看出`写法一`同步执行,`写法二`和`写法三`为异步执行,原因在于`写法二`和`写法三`使用了两个不同的`stream`流水线,当`CompletableFuture::join`写在同一个`stream`中,则会同步地执行`CompletableFuture`操作,这是由于流操作之间的延迟特性,如果在单一流水线中处理流,发向`CompletableFuture`操作只能以同步、顺序执行的方式才会成功,即当第一个`CompletableFuture`操作执行`join`等待结果执行完后,才会开始执行第二个`CompletableFuture`操作.

### end

&emsp;&emsp;`CompletableFuture`的使用心得就暂时写到这里,虽然想深入源码进行研究,但是发现用到的很多底层方法都是使用native方法,这期间反复看了几遍`java8 in action`关于`CompletableFuture`的章节,虽然只是一本工具书,但是通俗易懂却又不失深度,值得细细多看几遍.
