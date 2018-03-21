---
title: java集合启示录(三)之TreeMap
categories: 技术
tags: [java,集合,数据结构]
---

## start

&emsp;&emsp;在启示录(二)中对`HashMap`进行了细致的解读,但对于其中涉及到的一种数据结构---**红黑树**并没有特别说明.因为红黑树概念上比较好理解,但是针对红黑树的插入、删除等操作,由于各种情况分别处理而导致的复杂性,可能单独一篇文章详细说明比较好.对于许多算法思想偏弱的程序员(包括我)可能会对于这种复杂的数据结构很容易产生畏惧心理,但是算法就好比恶龙,而我们就是披荆斩棘的勇者,勇者岂能畏惧恶龙,让我们勇往直前,将那一条条恶龙踩在脚下吧.

- - -

<!--more-->

- - -

## content

### 概要

&emsp;&emsp;本文主要解读`TreeMap`,由于`TreeMap`的核心思想就是通过红黑树将key进行排序,为增、删、改、查这些操作提供了`log(N)`的时间开销.其重点自然是**红黑树**,所以本文会以大篇幅去详细介绍红黑树.

### 红黑树二叉树

&emsp;&emsp;对比普通的二叉查找树,在构造树的过程中可能会出现失衡的情况,极端情况下会只有左子树或只有右子树,这时候其实就像遍历链表一样需要遍历所有节点,导致树的索引效率大大降低.

&emsp;&emsp;为了避免出现上述情况,更大程度发挥二叉树的性能,此时我们需要这样一种树结构,对树进行增、删节点时,依然可以将树维持在一种**相对平衡**的状态.此处说的是相对平衡,而并没有说绝对平衡(**绝对平衡指每个结点的左右子树的高度之差的绝对值（平衡因子）最多为1.**,这便是`AVL树`的特点).虽然我们的愿景是绝对平衡,显然绝对平衡带来的优势是由于树的高度相对最低,索引效率也相对最高(`O(logN)`).与此同时,增、删节点时需要花费更多的操作去修复树(`O(logN)`).

&emsp;&emsp;`红黑树`相对于`AVL树`而言,通过满足部分平衡性使得增、删节点操作时需要相对较少的旋转操作,使整个树处于一种相对平衡的状态.根据维基百科的定义,红黑树是每个节点都带有颜色属性的二叉查找树,颜色为红色或黑色.在二叉查找树强制一般要求以外,对于任何有效的红黑树增加了如下的额外要求：

- 节点是红色或黑色
- 根是黑色
- 所有叶子都是黑色(子是NIL节点)
- 每个红色节点必须有两个黑色的子节点(个叶子到根的所有路径上不能有两个连续的红色节点.)
- 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点

![红黑树示例图](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E7%A4%BA%E4%BE%8B%E5%9B%BE.png)

&emsp;&emsp;根据红黑树的限制条件,我们可以很直观地分析出:红黑树中最短的可能路径是全部为黑色节点的路径,最长的可能路径是红黑相间的路径,由于每条路径都包含相同数目的黑色节点,所以**从根到叶子的最长的可能路径不多于最短的可能路径的两倍长**.正是因为这个性质,保证了红黑树的相对稳定性,在理论上最坏情况下检索效率依然高于普通二叉树.

&emsp;&emsp;了解了红黑树的特性后,会发现即便当前的树满足红黑树特性,但是对其进行增、删节点操作后,极有可能破坏了这种特性,此时便需要进行后续的修复操作来恢复红黑树.针对各种情况下的修复操作也正是红黑树代码实现的复杂点.

#### 结构

&emsp;&emsp;为了单纯地说明红黑树,这里不使用`HashMap`和`TreeMap`的代码实现,而是参考其代码自己实现了一遍红黑树.(这样才能加深印象嘛~)完整源码见[红黑树代码实现](https://github.com/ji4597056/algorithm-application).

```java
 	// red node
    private static final boolean RED = false;
    // black node
    private static final boolean BLACK = true;

    static final class Node<T> {
        T value;
        Node<T> left;
        Node<T> right;
        Node<T> parent;
        boolean color = BLACK;

        Node(T value, Node<T> parent) {
            this.value = value;
            this.parent = parent;
        }
	}
```

#### 旋转操作

&emsp;&emsp;在分析红黑树相关操作之前,有必要先解释旋转操作(`Rotate`),目的是使节点颜色符合定义,让红黑树的高度达到平衡.旋转操作分为两种:左旋和右旋.代码不详细注解,基本就是看图写代码,心中有图,自然有码.

##### 左旋

&emsp;&emsp;如图对x进行左旋操作,x将从头节点变成一个左节点.

![左旋](http://okzr61x6y.bkt.clouddn.com/%E6%A0%91%E5%B7%A6%E6%97%8B.jpg)

```java
 	private void rotateLeft(Node<T> p) {
        if (p != null) {
            Node<T> right = p.right;
            p.right = right.left;
            if (right.left != null) {
                right.left.parent = p;
            }
            right.parent = p.parent;
            if (p.parent == null) {
                root = right;
            } else if (p.parent.left == p) {
                p.parent.left = right;
            } else {
                p.parent.right = right;
            }
            // P变为左节点
            right.left = p;
            p.parent = right;
        }
    }
```

##### 右旋

&emsp;&emsp;如图对y进行右旋操作,x将从头节点变成一个右节点.

![右旋](http://okzr61x6y.bkt.clouddn.com/%E6%A0%91%E5%8F%B3%E6%97%8B.jpg)

```java
	 private void rotateRight(Node<T> p) {
        if (p != null) {
            Node<T> left = p.left;
            p.left = left.right;
            if (left.right != null) {
                left.right.parent = p;
            }
            left.parent = p.parent;
            if (p.parent == null) {
                root = left;
            } else if (p.parent.right == p) {
                p.parent.right = left;
            } else {
                p.parent.left = left;
            }
            // P变为右节点
            left.right = p;
            p.parent = left;
        }
    }
```

#### 插入节点

1. 新插入节点为红色叶子节点(这样做是为了维持红黑树的性质:从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点)
2. 如果父节点为红色节点,需要进行修复操作(若父节点为黑色,添加了红色叶子节点不会破坏红黑树的性质,因此无需进行修复操作)

&emsp;&emsp;插入修复操作分为以下的三种情况(父节点为左节点与父节点为右节点操作基本类似,这里以父节点为左节点为例):

- **1.叔叔节点为红**
此时将父节点和叔叔节点置黑色,爷爷节点置红色,修复后若曾祖父节点为红色,则需要继续从爷爷节点开始回溯修复(继续按三种情况处理,依次类推直到根节点).

![红黑树插入情况一](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E6%8F%92%E5%85%A5%E6%83%85%E5%BD%A2%E4%B8%80.png)

- **2.叔叔节点为null/黑,且爷爷节点、父节点和新节点不处于一条斜线上**
这种情况为中间状态,需要左旋父节点转化为第三种情况再处理.

![红黑树插入情况二](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E6%8F%92%E5%85%A5%E6%83%85%E5%BD%A2%E4%BA%8C.png)

- **3.叔叔节点为null/黑,且爷爷节点、父节点和新节点处于一条斜线上**
将父节点置黑色,爷爷节点置红色,右旋爷爷节点.(当叔叔节点为null时,我感觉其实也可以按情况一处理,这里需要特别说明当第一次进行修复操作时,叔叔节点不可能为黑,只有在触发情况一后的后续回溯修复过程中才可能出现叔叔节点为黑的情形)

![红黑树插入情况三](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E6%8F%92%E5%85%A5%E6%83%85%E5%BD%A2%E4%B8%89.png)

```java
	public void add(T value) {
        Node<T> t = root;
        // 添加root node
        if (t == null) {
            root = new Node<>(value, null);
            return;
        }
        // 遍历添加node为叶子节点
        int cmp;
        Node<T> parent;
        do {
            parent = t;
            cmp = comparator.compare(value, t.value);
            if (cmp < 0) {
                t = t.left;
            } else if (cmp > 0) {
                t = t.right;
            } else {
                // 如果存在该node直接返回
                return;
            }
        } while (t != null);
        Node<T> p = new Node<>(value, parent);
        if (cmp < 0) {
            parent.left = p;
        } else {
            parent.right = p;
        }
        // 添加后修复树
        fixAfterAdd(p);
        size++;
    }
	
    private void fixAfterAdd(Node<T> p) {
        // 新插入的节点为红色的
        p.color = RED;
        // 如果遇到父节点的颜色为黑则进行修复操作结束,因为此时添加红节点并不会破坏树的平衡性
        // 也就是说,只有在父节点为红色节点的时候是需要插入修复操作的
        while (p != null && p != root && p.parent.color == RED) {
            // 1.父节点(红色)属于左节点
            if (parentOf(p) == leftOf(parentOf(parentOf(p)))) {
                // q为叔叔节点(父节点的右兄弟节点)
                Node<T> q = rightOf(parentOf(parentOf(p)));
                // 1.1 叔叔节点为红
                if (colorEqual(q, RED)) {
                    setColor(parentOf(p), BLACK);
                    setColor(q, BLACK);
                    setColor(parentOf(parentOf(p)), RED);
                    // 爷爷节点由黑变红,可能需要从爷爷节点开始回溯修复
                    // 因为父节点红,爷爷节点必黑,曾祖父节点可红可黑,当爷爷节点变红,此时与曾祖父节点为红时冲突,需要修复
                    p = parentOf(parentOf(p));
                } else {
                    // 1.2 叔叔节点为null/黑
                    // 1.2.1 当前节点属于右节点(中间状态),需要左旋父节点,使得当前节点变为左节点
                    if (p == rightOf(parentOf(p))) {
                        p = parentOf(p);
                        rotateLeft(p);
                    }
                    // 1.2.2 当前节点属于左节点,右旋爷爷节点,使之平衡
                    setColor(parentOf(p), BLACK);
                    setColor(parentOf(parentOf(p)), RED);
                    rotateRight(parentOf(parentOf(p)));
                }
                // 2.父节点(红色)属于右节点
            } else {
                // q为叔叔节点(父节点的左兄弟节点)
                Node<T> q = leftOf(parentOf(parentOf(p)));
                // 2.1 叔叔节点为红
                if (colorEqual(q, RED)) {
                    setColor(parentOf(p), BLACK);
                    setColor(q, BLACK);
                    setColor(parentOf(parentOf(p)), RED);
                    // 爷爷节点由黑变红,可能需要从爷爷节点开始回溯修复
                    p = parentOf(parentOf(p));
                } else {
                    // 2.2 叔叔节点为null/黑
                    // 2.2.1 当前节点属于左节点(中间状态),需要右旋父节点,使得当前节点变为右节点
                    if (p == leftOf(parentOf(p))) {
                        p = parentOf(p);
                        rotateRight(p);
                    }
                    // 2.2.2 当前节点属于右节点,左旋爷爷节点,使之平衡
                    setColor(parentOf(p), BLACK);
                    setColor(parentOf(parentOf(p)), RED);
                    rotateLeft(parentOf(parentOf(p)));
                }
            }
        }
        // 根节点永远为黑
        root.color = BLACK;
    }
	
	private static <T> Node<T> parentOf(Node<T> p) {
        return (p == null ? null : p.parent);
    }

    private static <T> Node<T> leftOf(Node<T> p) {
        return (p == null) ? null : p.left;
    }

    private static <T> Node<T> rightOf(Node<T> p) {
        return (p == null) ? null : p.right;
    }

    private static <T> boolean colorEqual(Node<T> p, boolean c) {
        return p != null && p.color == c;
    }

    private static <T> void setColor(Node<T> p, boolean c) {
        if (p != null) {
            p.color = c;
        }
    }
```

#### 删除节点

&emsp;&emsp;插入节点操作已经很复杂了,然而删除节点操作更加复杂!首先从二叉树的角度来看,插入的节点必然是叶子节点,删除的节点就未必是叶子节点了,此时可能将原来的树分裂成三部分.其次从红黑树的角度来看,插入的节点总是先置红色,而删除的节点可能为红色或黑色,情况相对于插入节点肯定复杂了.下文中代码主要参考`TreeMap`中源码的实现.

&emsp;&emsp;删除节点时,分为三种情况:

- **1.待删除节点无子节点**:直接删除.
- **2.待删除节点仅有左子节点或仅有右子节点**:待删除节点父节点链接相应子节点.
- **3.待删除节点既有左子节点又有右子节点**:这种情况最为复杂,需要找到待删除节点的中序遍历的后继节点,交换两个节点值,转换为前两种情况再处理.

```java
	private void deleteNode(Node<T> p) {
        size--;
        // 若待当前节点(p)左孩子和右孩子都存在,则找到该节点(p)的中序遍历的后继节点,交换值,并将当前节点(p)指向后继节点
        // 后继节点为该节点右子树的最小左节点
        // 当当前节点(p)指向后继节点时,只会出现两种情况:1.仅有右子结点 2.无子节点
        if (p.left != null && p.right != null) {
            Node<T> s = successor(p);
            p.value = s.value;
            p = s;
        }
        // 设置替换节点,此时只会出现三种情况: 1.仅有右子节点 2.仅有左子节点 3.无子节点
        Node<T> replacement = (p.left != null ? p.left : p.right);
        // 1.当有子节点时(仅有右子节点或仅有左子节点)进行修复操作
        if (replacement != null) {
            // 将替换节点与当前节点(p)的父节点链接
            replacement.parent = p.parent;
            // 若当前节点(p)为root,则将root指向修复节点
            if (p.parent == null) {
                root = replacement;
            } else if (p == p.parent.left) {
                p.parent.left = replacement;
            } else {
                p.parent.right = replacement;
            }
            // 将当前节点(p)与树解除链接(会被gc释放内存)
            p.left = p.right = p.parent = null;
            // 若当前节点(p)为黑色,删除后会破坏红黑树的性质,因此需要进行修复操作
            // 修复操作这里传的的是替换节点,后文中传的是当前节点(p),因为:
            // (1)为了获取父节点以及兄弟节点引用
            // (2)替换节点为红色则直接置黑色,即恢复红黑树性质,替换节点为null(对应后文中传p)或黑色,则从兄弟节点去借调节点进行修复
            if (p.color == BLACK) {
                fixAfterDeletion(replacement);
            }
            // 2.当无子节点时进行修复操作
        } else if (p.parent == null) {
            root = null;
        } else {
            // 若当前节点(p)为黑色,删除后会破坏红黑树的性质,因此需要进行修复操作
            if (p.color == BLACK) {
                fixAfterDeletion(p);
            }
            // 将当前节点(p)与树解除链接(会被gc释放内存)
            if (p.parent != null) {
                if (p == p.parent.left) {
                    p.parent.left = null;
                } else if (p == p.parent.right) {
                    p.parent.right = null;
                }
                p.parent = null;
            }
        }
    }
	
	// 获取中序遍历的后继节点
	private static <T> Node<T> successor(Node<T> p) {
        if (p == null) {
            return null;
        } else if (p.right != null) {
            Node<T> n = p.right;
            while (n.left != null) {
                n = n.left;
            }
            return n;
        } else {
            Node<T> n = p.parent;
            Node<T> ch = p;
            while (n != null && ch == n.right) {
                ch = n;
                n = n.parent;
            }
            return n;
        }
    }
```

&emsp;&emsp;若待删除节点为黑色,删除后破坏了红黑树的性质(**从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点**),此时需要进行修复操作.若替换节点为红色,则直接置黑色即可恢复.若替换节点为黑色(或不存在),则删除修复操作分为以下四种情况(待删除节点为左节点与待删除节点为右节点操作基本类似,这里以待删除节点为左节点为例):

- **1.待删除的节点的兄弟节点是红色的节点**
兄弟节点为红时,无法通过左旋父节点直接借调黑节点,所以此时左旋父节点的目的是为了让兄弟节点变成黑节点.然后按照其他三种兄弟节点为黑节点的情况处理.(此时可能兄弟节点变为null,这时依然按情况二处理)

![红黑树删除情况一](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E5%88%A0%E9%99%A4%E6%83%85%E5%BD%A2%E4%B8%80.png)

- **2.待删除的节点的兄弟节点是黑色的节点,且兄弟节点的子节点都是null/黑**
兄弟节点为黑时,如果通过左旋父节点来借调黑节点,由于兄弟节点的子节点不存在红节点,因此会导致兄弟节点的路径中黑节点个数少一,因此不能这么处理.所以选择将兄弟节点置红色,此时父节点的左右子树黑节点个数相同,但是经过父节点的路径的黑节点个数少一.所以父节点变为待调整节点,根据具体情况进行修复操作.

![红黑树删除情况二](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E5%88%A0%E9%99%A4%E6%83%85%E5%BD%A2%E4%BA%8C.png)

- **3.待调整的节点的兄弟节点是黑色的节点,且兄弟节点的左子节点是红色的,兄弟节点的右子节点是null/黑**
跟第二种情况一样,如果左旋父节点会导致兄弟节点的路径中黑节点个数少一.所以右旋兄弟节点转换成第四种情况处理.

![红黑树删除情况三](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E5%88%A0%E9%99%A4%E6%83%85%E5%BD%A2%E4%B8%89.png)

- **4.待调整的节点的兄弟节点是黑色的节点,且兄弟节点的右子节点是红色的,兄弟节点的左子节点随意(null/红/黑)**
该情况可进行真正的节点借调操作.兄弟节点置父节点颜色,父节点置黑色,兄弟节点的右子节点置黑色,此时左旋父节点即可完成修复操作.

![红黑树删除情况四](http://okzr61x6y.bkt.clouddn.com/%E7%BA%A2%E9%BB%91%E6%A0%91%E5%88%A0%E9%99%A4%E6%83%85%E5%BD%A2%E5%9B%9B.png)

```java
    private void fixAfterDeletion(Node<T> p) {
        // 替换节点为红色,则直接置黑色即修复完成
        // 替换节点为null/黑,则视情况讨论,总体思路为如何从兄弟节点去借调节点
        while (p != root && !colorEqual(p, RED)) {
            // 1.当前节点(p)为左节点
            if (p == leftOf(parentOf(p))) {
                // 获取兄弟节点(sib)
                Node<T> sib = rightOf(parentOf(p));
                // 1.1 兄弟节点(sib)为红色(中间状态)
                if (colorEqual(sib, RED)) {
                    setColor(sib, BLACK);
                    setColor(parentOf(p), RED);
                    rotateLeft(parentOf(p));
                    // 获取新的兄弟节点(sib),此时兄弟节点(sib)为黑色
                    sib = rightOf(parentOf(p));
                }
                // 1.2 兄弟节点(sib)为黑时,左右侄子节点都不为红色(中间状态)
                if (!colorEqual(leftOf(sib), RED) && !colorEqual(rightOf(sib), RED)) {
                    setColor(sib, RED);
                    // 父节点(可红可黑)的左右子节点的黑节点数量均少1,此时需要从父节点开始继续调整
                    p = parentOf(p);
                } else {
                    // 1.3 兄弟节点(sib)为黑时,左侄子为红色,右侄子为null/黑(中间状态)
                    if (!colorEqual(rightOf(sib), RED)) {
                        setColor(leftOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateRight(sib);
                        sib = rightOf(parentOf(p));
                    }
                    // 1.4 兄弟节点(sib)为黑时,右侄子为红色,左侄子为null/黑/红
                    setColor(sib, colorOf(parentOf(p)));
                    setColor(parentOf(p), BLACK);
                    setColor(rightOf(sib), BLACK);
                    rotateLeft(parentOf(p));
                    // 无需继续修复,跳出循环
                    p = root;
                }
                // 2.当前节点(p)为右节点
            } else {
                // 获取兄弟节点(sib)
                Node<T> sib = leftOf(parentOf(p));
                // 2.1 兄弟节点(sib)为红色
                if (colorOf(sib) == RED) {
                    setColor(sib, BLACK);
                    setColor(parentOf(p), RED);
                    rotateRight(parentOf(p));
                    // 获取新的兄弟节点(sib),此时兄弟节点(sib)为黑色
                    sib = leftOf(parentOf(p));
                }
                // 2.2 兄弟节点(sib)为黑时,左右侄子节点都不为红色
                if (!colorEqual(rightOf(sib), RED) && !colorEqual(leftOf(sib), RED)) {
                    setColor(sib, RED);
                    // 父节点(可红可黑)的左右子节点的黑节点数量均少1,此时需要从父节点开始继续调整
                    p = parentOf(p);
                } else {
                    // 2.3 兄弟节点(sib)为黑时,左侄子为红色,右侄子为null/黑
                    if (!colorEqual(leftOf(sib), RED)) {
                        setColor(rightOf(sib), BLACK);
                        setColor(sib, RED);
                        rotateLeft(sib);
                        sib = leftOf(parentOf(p));
                    }
                    // 2.4 兄弟节点(sib)为黑时,右侄子为红色,左侄子为null/黑/红
                    setColor(sib, colorOf(parentOf(p)));
                    setColor(parentOf(p), BLACK);
                    setColor(leftOf(sib), BLACK);
                    rotateRight(parentOf(p));
                    // 无需继续修复,跳出循环
                    p = root;
                }
            }
        }
        setColor(p, BLACK);
    }
```

### TreeMap

&emsp;&emsp;前面已经很大篇幅介绍了红黑树,`TreeMap`使用红黑树的性质根据有序key存储节点.可以看到`TreeMap`比`HashMap`多继承了`NavigableMap`接口(该接口继承了`SortedMap`),因此提供了`HashMap`没有的一些实用性api方法,这里不做过多介绍.在很多场景下非常有用,比如之前`一致性hash算法探究`一文中就使用`TreeMap`存储虚拟节点.这里就不详细说明了.

&emsp;&emsp;要注意的一点是初始化`TreeMap`时可以传`Comparator<? super K> comparator`比较器,当然不传也没关系,此时需要key实现`Comparable<T>`接口(比如我们常用作key的`String`类).如果key没有实现该接口,则在存取元素时就会抛异常了.

### end

&emsp;&emsp;写这篇文章真的感触挺深,因为本打算最多两天写完,但是研究红黑树过程中参阅了很多网上别人的博客文章,发现很多文章要么图画错了,要么对于红黑树的增、删节点操作解释的有问题.导致我的思维也很混乱,产生各种各样的疑问.后来结合代码一步步分析,才渐渐解决了心中的疑虑.得出的结论时:算法和数据结构相关的学习资料只能相信专业书籍(如`算法导论`等),其他博客类文章都必须带着辩证的思维去阅读.

&emsp;&emsp;红黑树要想通俗地解释明白各种操作确实挺难,我清楚这篇文章写得不算很好.唯一让我欣慰的是,我已经不像以前一样是为了写博客而写.我不会再写说服不了自己的文章!