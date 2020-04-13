# ConcurrentSkipListMap 中的非阻塞链表

本文翻译自 [Timothy L. Harris - A Pragmatic Implementation of Non-Blocking Linked-Lists][paper]。

作者简介

- Timothy L. Harris
- University of Cambridge Computer Laboratory, Cambridge, UK, tim.harris@cl.cam.ac.uk


Java 8 `ConcurrentSkipListMap` 提到了这个 paper，为了更好的理解这个数据结构，特来学习一下。


- [ConcurrentSkipListMap 中的非阻塞链表](#concurrentskiplistmap-%e4%b8%ad%e7%9a%84%e9%9d%9e%e9%98%bb%e5%a1%9e%e9%93%be%e8%a1%a8)
  - [摘要](#%e6%91%98%e8%a6%81)
  - [引言](#%e5%bc%95%e8%a8%80)
  - [概览](#%e6%a6%82%e8%a7%88)
  - [相关工作](#%e7%9b%b8%e5%85%b3%e5%b7%a5%e4%bd%9c)
  - [算法](#%e7%ae%97%e6%b3%95)
    - [实现集合](#%e5%ae%9e%e7%8e%b0%e9%9b%86%e5%90%88)
  - [正确性](#%e6%ad%a3%e7%a1%ae%e6%80%a7)
  - [结果](#%e7%bb%93%e6%9e%9c)
  - [DeleteGE](#deletege)
  - [结论](#%e7%bb%93%e8%ae%ba)
  - [参考](#%e5%8f%82%e8%80%83)

## 摘要

我们提出了一种支持线性的插入和删除操作，非阻塞版本的并行链表。新算法比以前的方案更快更简单。

## 引言

越来越明显的是，非阻塞算法可以向并行系统[MP91，LaM94，GC96，ABP98，Gre99]提供重要的好处。这些算法使用 CAS 这样的底层的原子操作，通过精心设计和避免使用锁，从而有可能构建可扩展到高度并行的环境并且对调度决策具有弹性的系统。

链表是程序设计中最基本的数据结构之一，因此简单有效的非阻塞链表实现可以作为许多数据结构的基础。本文提出了一种链表的新颖实现，该链表非阻塞，可线性化，基于现代处理器上的 CAS 操作。

第 5 节，概述了正确性证明，描述了使用模型检查在有限的应用程序域内执行详尽的验证，还描述了对来自实际实现的执行迹线进行的经验测试。

第 6 节，我们将新算法的性能与基于锁的实现以及现有的非阻塞算法的性能进行了比较。与其他线程安全算法相比，我们的算法在三种模拟工作负载中的每一个上以及在每个并发级别上均提供最佳性能。

## 概览

在本节中，我们概述了我们的算法以及实现非阻塞链表的难度。下面的有序列表包含整数 10 、30 和 *head*、*tail* 两个哨兵节点：

![NBLL-2-1](http://image.rainstorm.vip/blog/NBLL-2-1.png)

列表中节点的数据结构实现一般包含两个字段：一个 **key** 字段来保存列表元素，一个 **next** 字段来保存到列表下一个节点的引用。

插入很简单：创建一个新的列表节点（在左下），然后在列表中合适位置的前一个节点的 **next** 字段（在右下）上使用 CAS 操作将其引入。

![NBLL-2-2](http://image.rainstorm.vip/blog/NBLL-2-2.png)

在这种情况下，CAS 的原子性可确保插入两侧的节点保持相邻。 这种简单的保证不足以删除列表中的内容。 假设我们希望删除值10，删除该节点的一种明显方法是执行 CAS 操作将 *head* 直接指向包含30的节点

![NBLL-2-3](http://image.rainstorm.vip/blog/NBLL-2-3.png)

尽管此 CAS 操作确保节点10仍在列表的开头，但是它不能确保在10节点和30节点之间没有引入其他节点。 如果此删除与上一次插入同时发生，则该新节点将丢失：

![NBLL-2-4](http://image.rainstorm.vip/blog/NBLL-2-4.png)

一旦删除程序选择了30，则单个CAS既无法检测也无法阻止10到30之间的变化。我们提出的解决方案-这是本算法的核心{使用两个单独的CAS运算来代替单个CAS。 其中一个用于以某种方式 *mark* 已删除节点的 **next** 字段（左下方），第二个用于切除节点（右下方）：

![NBLL-2-5](http://image.rainstorm.vip/blog/NBLL-2-5.png)

第一个 CAS 执行的是逻辑删除，第二个 CAS 执行的是物理删除。标记的字段仍可以遍历，但在数值上与其先前的未标记状态不同； 在发信号通知并发插入时，保留列表的结构，以避免在逻辑删除的节点之后立即引入新节点。 在我们的示例中，并发插入20将观察到10在逻辑上被删除，并在重新尝试插入之前尝试物理删除它。

## 相关工作

Herlihy [Her91，Her93]提出了基于CAS的通用非阻塞实现。但是，基于此通用方案的链表高度集中并且性能不佳，因为它们本质上使用CAS将共享的全局指针从结构的一个版本更改为另一个版本。

Valois 是第一个提出基于CAS的链表的非阻塞实现的有效方法[Val95]。

> Although highly distributed, his implementation is very involved. The list is held with auxilliary cells between adjacent pairs of ordinary cells. Auxilliary exist to provide an extra level of indirection so that a cell may be removed by joining together the auxilliary cells adjacent to it.

- *这段没看懂，直接放出来了*

Valois 的算法比我们这里公开了更通用，更底层的接口。他提供了显式游标以标识列表中的节点，并提供了在这些点处插入或删除节点的操作。

最初发布的算法包含许多与参考计数存储的管理方式有关的错误。在本文以前已经报道了一个，在实现 Valois 的比较算法时发现了其他问题[MS95，Val01]。

为了克服使用 CAS 构建可线性化的无锁链表的复杂性，Greenwald提出了一种更强大的双重比较和交换（DCAS）原语，该原语在确认两个存储位置都包含必需的值后会自动更新两个存储位置[Gre99]。 DCAS 在当今的多处理器体系结构上不可用。但是，它确实提供了一种简单的线性化链表算法：插入按照第二节中的描述进行、通过原子更新来删除要删除的节点及前一个节点的 **next** 字段。Greenwald的工作是对早期 Massalin和Pu [MP91]基于DCAS的不可线性化链表算法的扩展。

## 算法

在本节中，我们以 C++ 为模型，以伪代码展示我们的新算法，该算法旨在支持 *read*，*write* 和 原子 *compare-and-swap* 在常规共享内存多处理器系统上的执行。 我们假定此处定义的操作是访问链表对象的唯一方法。每个处理器执行一系列这些操作，定义调用/响应的历史记录并在它们之间产生实时顺序。 我们说，如果对A的响应在调用B之前发生，则操作A在B之前，并且如果它们没有实时排序，则这些操作是并发的。

我们的基本正确性要求是可线性化/串行化，并发操作可以用某种方式排列串行执行来达到相同的效果。线性化意味着操作似乎在调用和响应之间的某个时刻原子地起作用。我们的实现是非阻塞的，这意味着即使其他操作停止，某些操作也将以有限的步骤完成。

```c++
class List<KeyType> {
    Node<KeyType> *head;
    Node<KeyType> *tail;

    List() {
        head = new Node<KeyType> ();
        tail = new Node<KeyType> ();
        head.next = tail;
    }
}

class Node<KeyType> {
    KeyType key;
    Node *next;

    Node (KeyType key) {
        this.key = key;
    }
}
```

`List` 类的实例包含两个字段，`head` 和 `tail`，`Node` 包含两个字段表示 `key` 和后继节点。

```c++
public boolean List::insert (KeyType key) {
    Node *new_node = new Node(key);
    Node *right_node, *left_node;
    do {
        right_node = search (key, &left_node);
        if ((right_node != tail) && (right_node.key == key)) /*T1*/
            return false;
        new_node.next = right_node;
        if (CAS (&(left_node.next), right_node, new_node)) /*C2*/
            return true;
    } while (true); /*B3*/
}
```

`List::insert` 方法试图为给定的 key 插入一个新节点。

我们为 CAS 操作编写 `CAS(addr, o, n)`，该操作原子性地将addr的内容与 旧值o 进行比较，如果匹配则将 n 写入该位置。CAS 返回一个布尔值，指示是否进行了此更新。我们的设计基于以下假设：CAS操作的执行速度比写入操作慢，而写入操作则比读取操作慢。

### 实现集合

我们先考虑一个支持 `Insert(k)`、`Delete(k)`，`Find(k)` 三个操作的集合对象。每个参数k 从一组完全有序的键列表中取到，`Insert`、`Delete`、`Find` 操作返回一个 `boolean` 类型的结果来表示操作成功或失败。这个集合使用单链表 `List` 的实例来表示。如第2节介绍的那样，key 在链表中以升序排列，并且有 head 和 tail 两个哨兵节点。

```c++
public boolean List::delete (KeyType search_key) {
    Node *right_node, *right_node_next, *left_node;
    do {
        right_node = search (search_key, &left_node);
        if ((right_node == tail) || (right_node.key != search_key))
            /*T1*/
            return false;
        right_node_next = right_node.next;
        if (!is_marked_reference(right_node_next))
            if (CAS (&(right_node.next), right_node_next, get_marked_reference (right_node_next)))
                /*C3*/
                break;
    } while (true); /*B4*/
    if (!CAS (&(left_node.next), right_node, right_node_next))
        /*C4*/
        right_node = search (right_node.key, &left_node);
    return true;
}
```

`List::delete` 方法试图删除包含指定 key 的节点。

```c++
public boolean List::find (KeyType search_key) {
    Node *right_node, *left_node;

    right_node = search (search_key, &left_node);
    if ((right_node == tail) || (right_node.key != search_key))
        return false;
    else
        return true;
}
```

`List::find` 方法检测指定的 key 是否在 list 中。

一个节点的 **next** 字段只有两种状态，`marked` 或者 `unmarked`。只有在 **next** 字段为 `marked` 的时候，节点才为 `marked`。被标记的引用和正常引用不同，这里有一个区分它们的可选方式：因为 C++ 对象的内存对齐，使得正常对象引用的最低位为 0，节点的 next 字段的最低位记录当前节点的逻辑删除状态。

**译者补充** : 下面是一种可行的代码实现 [ab-passos - Non Blocking Linked List][NBLL]

```c++
static const bool is_marked_reference(const uintptr_t p){
    return p & 0x1;
}

static const uintptr_t get_unmarked_reference(const uintptr_t p){
    return p & 0xFFFFFFFE;
}

static const uintptr_t get_marked_reference(const uintptr_t p){
    return p | 0x1;
}
```

一个 `marked` 节点应该被忽略，因为该节点正在被删除。`is_marked_reference(r)` 仅在 `r` 是 `marked` 引用时返回 true。`get_marked_reference(r)` 和 `get_unmarked_reference(r)` 用来将引用在两个状态直接转换。

并发实现由四个方法组成。`List::insert`、`List::delete`、 `List::find` 实现了 插入删除和查找操作， `List::search` 在上面三个操作中都用到了，它的入参是一个 key，返回该 key 的 `left node` 和 `right node`，这个方法确保这些节点满足一些条件。

1. `left node` 的 key 必须小于搜索key，`right node` 的key 必须大于等于搜索key。
2. 两个节点必须都是 `unmarked`。
3. `right node` 必须是 `left node` 的直接后继节点。

最后的这个条件需要 search 操作去删除 `marked` 节点来保证两个节点是相邻的。`List::search` 方法保证在方法调用和结束之间的一些时刻满足这些条件。

```c++
private Node *List::search (KeyType search_key, Node **left_node) {
    Node *left_node_next, *right_node;

    search_again:
    do {
        Node *t = head;
        Node *t_next = head.next;

        /* 1: Find left_node and right_node */
        do {
            if (!is_marked_reference(t_next)) {
                (*left_node) = t;
                left_node_next = t_next;
            }
            t = get_unmarked_reference(t_next);
            if (t == tail) break;
            t_next = t.next;
        } while (is_marked_reference(t_next) || (t.key<search_key)); /*B1*/
        right_node = t;

        /* 2: Check nodes are adjacent */
        if (left_node_next == right_node)
            if ((right_node != tail) && is_marked_reference(right_node.next))
                goto search_again; /*G1*/
            else
                return right_node; /*R1*/

        /* 3: Remove one or more marked nodes */
        if (CAS(&(left_node.next), left_node_next, right_node)) /*C1*/
            if ((right_node != tail) && is_marked_reference(right_node.next))
                goto search_again; /*G2*/
            else
                return right_node; /*R2*/
    } while (true); /*B2*/
}
```

`List::search` 被分为三个部分。

1. 第一个部分在 list 中迭代，来发现第一个大于等于 搜索key 的 `unmarked` 节点，这是 `right node`。`left node` 是 `right node` 前面最近的一个 `unmarked` 节点。
2. 第二个部分检测这些节点，如果 `left node` 的直接后继是 `right node` 则直接返回。
3. 第三部分，使用 CAS 操作删除 `left node` `right node` 之间的 `marked` 节点。

为了便于理解第一部分，下面是一个 demo：

![Find left_node and right_node demo](http://image.rainstorm.vip/blog/nbll-search-demo.png)

`List::insert` 使用 `List::search` 来定位到新节点插入时的 节点对。使用单个 CAS 操作将 `left_node.next` 从 `right node` 替换为新节点。

`List::delete` 使用 `List::search` 来定位要删除的节点，然后使用两阶段处理来执行操作请求。收集，节点通过将 `right_node.next` 指向的引用标记为 `marked` 来进行逻辑删除。然后节点被物理删除，这个操作可能在 `delete` 时操作或者在 `search` 方法中被触发。

`List::find` 调用 `List::search` 然后检测返回的 `right node`。

## 正确性

// 看的我一脸懵逼，不翻译了。有兴趣自己看原文。

## 结果

我们使用 C + SPARC V9 汇编语言实现了上面描述的算法，并在以下环境评估性能：

- E450 服务器
- Solaris 8
- 4 400MHz SPARC V9 core
- 4GB 物理内存

值得强调的是，上述代码仅用作伪代码，并不反映优化后的（甚至是正确的）实现。处理器可能需要另外的的内存屏障-例如在初始化新节点的字段并将其引入列表之间，或者逻辑删除的CAS与物理删除的CAS之间。

![非阻塞链表实现压测](http://image.rainstorm.vip/blog/nbll-test.png)

测试应用比较了我们的实现和 Valois 的无锁算法、简单加互斥锁之间的性能（这种比较在某种程度上是不公平的，因为加锁的实现完全可以选择一种更高效的数据结构，比如树或者跳表）。两种无锁算法评估了基于引用计数和不基于引用计数的的性能。所有的列表节点在操作之前提前分配，以保障内存分配不会影响到我们的测试结果。操纵引用计数的代码基于Michael和Scott对Valois算法的修改版本，不同之处在于，在释放节点时递归地减少引用计数。

key 从指定返回内随机选择，插入操作和删除操作的机会均等。我们每个线程都使用相同参数调用独立的线性同余随机数生成器来生成随机数，使用的函数是 Solaris 8 libc 库中的 lrand48。seed 是提前选好的，已给出不重复的随机数序列。

我们对测试工具进行了参数化，参数有使用的算法，并发线程数、key的范围。在每种情况下，每个线程执行 1000000 个操作。上图显示了不同算法在不同负载下的CPU处理时间。

显而易见的是，对于使用多个线程的实验，我们的算法的性能均显著提高。在单线程执行的情况下，在这些测试中它的性能优于Valois算法，并且其性能与基于锁的实现相同。 与Valois算法相比，相对性能不足为奇：我们避免了创建，遍历和切除辅助节点的操作。

除了这些图中显示的负载之外，我们还测试了具有较大范围的key，或者使用一长串永不删除的节点对列表进行“初始”填充的情况。这都会增加列表中节点的总数，从而增加CAS指令失败时重试操作的成本。一种担心是，由于可能会进行多次重试，因此无锁算法的性能将开始下降。我们研究了多达 65536 个元素的工作负载，并且无法找到基于互斥算法可提供更优性能的测试场景。我们怀疑尽管每次重试都变得更加昂贵，但是随着冲突更新率的下降，重试的可能性会降低。

上图（a）显示了引用计数实现的性能。在单线程情况下，Valois 算法的 CPU 需求降低了 5倍，而对于 16个线程，则提高到 11倍以上。类似地，我们算法所需的CPU时间降低了10 - 15倍。在每种情况下，这都是在遍历列表的每个阶段都需要操纵引用计数（使用CAS操作）的结果。
这说的是啥？？？能不能靠点谱。

因为 SPARC 处理器不提供原子获取和添加操作，所以引用计数操作的性能受到了限制。 但是，在双处理器Intel x86计算机（具有该设备[PPr96]）上进行的测量表明，与此处看到的总成本相比，性能提高很少。 当引用计数位于L1的单独缓存行上时，则通过CAS的更新比使用fetch-andadd进行的更新慢10％。当两个处理器尝试更新相同的地址时，这增加了2倍。

当然，我们的结果是乐观的，因为他们没有考虑执行GC的成本。但是，正如Jones和Lins所写的那样，如果活跃数据结构的大小是固定的，则可以以总堆大小为代价降低复制收集器的成本[JL96]。实际上，他们报告说，在现代实现良好的系统中，GC总成本通常约为10-20％

我们研究了基于延迟释放节点的另一种存储回收方法。在该方案中，每个节点都包含一个附加域，当将其从主列表中删除时，可以通过该域链接到要释放的列表。在开始每个操作之前，每个线程都将全局计时器的快照作为其当前时间。当条目的剔除时间小于所有线程的当前时间时，将他们从要释放的列表中删除：此时，没有线程的本地变量持有该节点的引用。

我们的实现为每个线程分配了一对要释放的列表。它们被称为 *old list* 和 *new list*，并与当前线程的计时器快照一起保存，该快照比旧列表中任何元素的执行时间都新。当所有线程的当前时间都超过快照时，旧列表的全部内容将被释放，新列表的元素将移至旧列表。

与使用垃圾回收相比，此延迟释放方案引入了两个主要开销。首先，需要进行CAS操作以将节点放置在要释放的列表上 - 在我们的实现中，与上图（a）使用16个线程运行的结果相比，这将CPU需求提高了15％。 第二个开销是从要释放的列表中删除元素并确定何时安全地这样做的开销。如果每1000次操作执行一次，则为1％；每100次操作执行一次，则为5％；如果在每次操作之后执行，则增加到52％。

下图中展示了对三种非引用计数算法的运行时性能的进一步分析，显示了四种不同操作的执行时间在 8个并发线程、 0...255 范围内执行键的插入和删除时的分布。在成功操作的情况下，基于锁的实现比任何一种无锁方案都可以实现更短的执行时间。但是，偶尔也有较长的执行时间，这解释了上图较高的平均执行时间。

对于不成功的操作，情况有所不同，因为两种无锁算法的执行时间都比基于锁的实现的执行时间短 - 请记住，不进行任何CAS操作或对数据结构进行其他更新，就可以进行不成功的操作。

![操作时间分布](http://image.rainstorm.vip/blog/nbll-test-1.png)

## DeleteGE

现在进一步考虑实现 `DeleteGE(k)`，该操作删除并返回大于或等于k的最小项。 试图通过修改 `List::delete` 来实现这一点，以便如果右节点的键大于搜索键，测试T1不会失败。

不幸的是这个实现是不可线性化的。假设三个插入操作按如下顺序执行：`insert(20)`, `insert(15)`, `insert(20)`。前两个成功，第三个必须是失败的，因为 20 已经在集合里了。在这个过程中加入并发的 `DeleteGE(10)` 操作，尝试删除一个大于等于10的元素。并发执行可能以下面的顺序执行：

- `List::deleteGE` 在第一个20插入后立即调用 `List::search`。拿到的 left node 是 head，right node 是20
- 15 成功插入
- 第二个 20 插入失败，它的 search 返回的 left node 是 15，right node 是 20，
- `List::deleteGE(10)` 完成了 20 的逻辑删除

我们必须对 `DeleteGE(10)` 操作进行排序，以便通过顺序执行获得其结果 20。这要求将其放置在插入15之前，不然应该优先将键15返回而不是20。但是，在插入20失败之后，我们还必须线性化删除操作，因为否则插入会成功。 这些约束是不可调和的。

直观地讲，问题是在执行C3时，右节点不必是左节点的直接后继。这在 `Delete(k)` 操作中是可以接受的，因为我们只关心并发更新具有相同key的节点。更新必须标记了这个右节点，因此C3将会失败。相反，在执行 `List::deleteGE` 的过程中，我们必须关注对键大于或等于搜索键的任何节点的更新。

我们可以通过保留 `List::deleteGE` 的实现，但是更改 `List::insert`，以免在左右节点之间插入新节点时C3必须失败。这意味着，每当C3成功时，右节点的键仍必须是大于或等于搜索键的最小键。

这是通过使用一次CAS操作来实现的：（a）引入一对新节点，一个新节点包含要插入的值，另一个新节点复制右节点，并且（b）标记原始的右节点：

![NBLL-7-1](http://image.rainstorm.vip/blog/nbll-7-1.png)

这个CAS从概念上讲有两个作用。首先，它将新节点引入列表：事先未标记后继节点的下一个字段，因此右节点仍必须是左节点的后继节点。 其次，通过标记 next 字段，CAS将导致任何具有相同右节点的并发 `List::deleteGE` 失败。请注意，现在标记的右节点的 key 顺序不正确。 但是，已编写`List::search`，`List::delete`，`List::find` 和 `List::deleteGE` 的现有实现，它们不依赖于标记节点的正确顺序。

## 结论

本文提出了一种新的非阻塞实现的链表。我们认为这里介绍的算法是可线性化的。它们也已实现，并且我们已经表明，它们的测量性能比以前发布的非阻塞数据结构和基于锁的实现性能更优。

## 参考

- [ABP98] Nimar S. Arora, Robert D. Blumofe, and C. Greg Plaxton. Thread scheduling for multiprogrammed multiprocessors. In Proceedings of the 10th Annual ACM Symposium on Paral lel Algorithms and Architectures, pages 119{129, Puerto Vallarta, Mexico, June 28{July 2, 1998. SIGACT/SIGARCH.
- [GC96] Michael Greenwald and David Cheriton. The synergy between non-blocking synchronization and operating system structure. In USENIX, editor, 2nd Symposium on Operating Systems Design and Implementation (OSDI '96), October 28{31, 1996. Seattle, WA, pages 123{136, Berkeley, CA, USA, October 1996. USENIX.
- [Gre99] M Greenwald. Non-blocking synchronization and system design. PhD thesis, Stanford University, August 1999. Technical report STAN-CS-TR-99-1624.
- [Her91] Maurice Herlihy. Wait-free synchronization. ACM Transactions on Program- ming Languages and Systems, 13(1):124{149, January 1991.
- [Her93] Maurice Herlihy. A methodology for implementing highly concurrent data objects. ACM Transactions on Programming Languages and Systems, 15(5):745{770, November 1993.
- [Hol97] Gerard J Holzmann. The moel checker SPIN. IEEE Transactions on Software Engineering, 23(5):279{295, May 1997.
- [HW90] Maurice P. Herlihy and Jeannette M. Wing. Linearizability: A correctness condition for concurrent ob jects. ACM Transactions on Programming Languages and Systems, 12(3):463{492, July 1990.
- [IS99] Radu Iosif and Riccardo Sisto. dSPIN: A dynamic extension of SPIN. In Proc. of the 6th International SPIN Workshop, volume 1680 of LNCS, pages 261{276. Springer-Verlag, September 1999.
- [JL96] Richard Jones and Rafael Lins. Garbage Col lection: Algorithms for Automatic Dynamic Memory Management. John Wiley and Sons, July 1996.
- [LaM94] Anthony LaMarca. A performance evaluation of lock-free synchronization protocols. In Proceedings of the Thirteenth Symposium on Principles of Distributed Computing, pages 130{140, 1994.
- [MP91] Henry Massalin and Calton Pu. A lock-free multiprocessor OS kernel. Technical Report CUCS-005-91, Columbia University, 1991.
- [MS95] Maged M. Michael and Michael L. Scott. Correction of a memory management method for lock-free data structures. Technical Report TR599, University of Rochester, Computer Science Department, December 1995.
- [PPr96] Pentium Pro Family Developer's Manual, volume 2, programmer's reference manual. Intel Corporation, 1996. Reference number 242691-001.
- [Val95] John D. Valois. Lock-free linked lists using compare-and-swap. In Proceedings of the Fourteenth Annual ACM Symposium on Principles of Distributed
- Computing, pages 214{222, Ottawa, Ontario, Canada, 2{23 August 1995.
- [Val01] John D Valois. Personal communication. March 2001.

[paper]: https://www.cl.cam.ac.uk/research/srg/netos/papers/2001-caslists.pdf
[NBLL]: https://github.com/ab-passos/Non-Blocking-Linked-List/blob/master/main.c
