---
layout: wiki
title: C++STL容器类简介
cate1: C++
cate2: 
description: C++STL容器类简介
keywords: C++ STL
---


本文旨在对 C++ 标准模板库的 `array`, `vector`, `deque`, `list`, `forward_list`, `queue`, `priority_queue`, `stack`, `map`, `multimap`, `set`, `multi_set`, `unordered_map`, `unordered_multimap`, `unordered_set`, `unordered_multiset` 共十六类容器进行系统的对比分析，参考资料为[C++官方文档](https://cplusplus.com/reference/stl/)。

## 容器列表
### Sequence containers:
* array	Array class (class template)
* vector	Vector (class template)
* deque	Double ended queue (class template)
* forward_list	Forward list (class template)
* list	List (class template)

### Container adaptors:
* stack	LIFO stack (class template)
* queue	FIFO queue (class template)
* priority_queue	Priority queue (class template)

### Associative containers:
* set	Set (class template)
* multiset	Multiple-key set (class template)
* map	Map (class template)
* multimap	Multiple-key map (class template)

### Unordered associative containers:
* unordered_set	Unordered Set (class template)
* unordered_multiset	Unordered Multiset (class template)
* unordered_map	Unordered Map (class template)
* unordered_multimap	Unordered Multimap (class template)

## array
```
Container properties: Sequence | Contiguous storage | Fixed-size aggregate
容器属性：顺序容器（支持随机访问），连续内存空间，固定大小；//连续内存
类模板头：template < class T, size_t N > class array;

```

array 即数组，其大小固定，所有的元素严格按照内存地址线性排列，array 并不维护元素之外的任何多余数据，甚至也不会维护一个size这样的变量，这保证了它在存储性能上和C++语法中的数组符号[]无异。尽管其它大部分标准容器都可以通过 std::allocator 来动态的分配和回收内存空间，但 Array 并不支持这样做。

Array 和其它标准容器一个很重要的不同是：对两个 array 执行 swap 操作意味着真的会对相应 range 内的元素一一置换，因此其时间花销正比于置换规模；但同时，对两个 array 执行 swap 操作不会改变两个容器各自的迭代器的依附属性，这是由 array 的 swap 操作不交换内存地址决定的。

Array 的另一个特性是：不同于其它容器，array 可以被当作 std::tuple 使用，因为 array 的头文件重载了get()以及tuple_size()和tuple_element()函数（注意这些函数非 array 的成员函数，而是外部函数）。

最后需要注意，虽然 array 和 C++语法中的[]符号无限接近，但两者是两个存在，array 毕竟是标准模板库的一员，是一个class，因此支持begin(), end(), front(), back(), at(), empty(), data(), fill(), swap(), ... 等等标准接口，而[]是真正的最朴素的数组。

## vector
```
Container properties: Sequence | Dynamic array | Allocator-aware
容器属性：顺序容器（支持随机访问），动态调整大小，使用内存分配器动态管理内存；//连续内存
类模板头：template < class T, class Alloc = allocator<T> > class vector;
```
一句话来说，vector 就是能够动态调整大小的 array。和 array 一样，vector 使用连续内存空间来保存元素，这意味着其元素可以用普通指针的++和--操作来访问；不同于 array 的是，其存储空间可以自动调整。

在底层上，vector 使用动态分配的 array，当现有空间无法满足增长需求时，会重新分配（reallocate）一个更大的 array 并把所有元素移动过去，因此，vector 的 reallocate 是一个很耗时的处理。所以，每次 reallocate 时都会预留多余的空间，以满足潜在的增长需求，也就是说，vector的capacity()通常会大于size()。vector 什么时候做 reallocate，reallocate 多少多余空间，是有具体策略的，按下不表。总体来说，vector 比 array 多了一些内存消耗，以换取更灵活的内存管理。

和其它的动态顺序容器（deque, list, forward_list）相比，vector 在元素访问上效率最高，在尾部增删元素的效率也相对最高。如果调用者有在尾部以外的地方增删元素的需求，vector 则不如其它容器，并且迭代器的一致性也较差（have less consistent iterators and references than lists and forward_lists）。

## deque
```
Container properties: Sequence | Dynamic array | Allocator-aware
容器属性：顺序容器（支持随机访问），动态调整大小，使用内存分配器动态管理内存；//分段连续内存
类模板头：template < class T, class Alloc = allocator<T> > class deque;
```
deque（读作"deck"）是 double-ended queue 的缩写，是一个可以在首尾两端进行动态增删的顺序容器。

不同的库对 deque 的实现可能不同，但大体上都是某种形式的动态 array，且都支持随机访问。deque 的功能和 vector 比较接近，但 deque 额外支持在头部动态增删元素。和 vector 不一样的是，deque 不保证存储区域一定是连续的！因此用指向元素的普通指针做++和--操作是非常危险的行为。

从底层机理上能更透彻地理解 deque 的特点：vector 使用的是单一的 array，deque 则会使用很多个离散的 array 来组织数据「the elements of a deque can be scattered in different chunks of storage」！如果说 vector 是连续的，deque 则是分段连续。deque 会维护不同 array 之间的关联信息，使用户无需关心分段这个事实。这样做的好处是很明显的：deque 在 reallocate 时，只需新增/释放两端的 storage chunk 即可，无需移动已有数据（vector 的弊端），极大提升了效率，尤其在数据规模很大时，优势明显。

相比于 vector 和 list，deque 并不适合遍历！因为每次访问元素时，deque 底层都要检查是否触达了内存片段的边界，造成了额外的开销！deque 的核心优势是在双端都支持高效的增删操作，程序员选择使用 deque 时需要有双端操作的明确理由。


## list
```
Container properties: Sequence | Doubly-linked list | Allocator-aware
容器属性：顺序容器（可顺序访问，但不支持随机访问），双链表，使用内存分配器动态管理内存；//离散内存
类模板头：template < class T, class Alloc = allocator<T> > class list;
```

list 是一种支持在任意位置都可以快速地插入和删除元素的容器，且支持双向遍历。list 容器能够做到这些的原因在于其底层结构是双链表，双链表允许把各个元素都保存在彼此不相干的内存地址上，但每个元素都会与前后相邻元素关联。

和其它的顺序容器（array, vector, deque）相比，list 的最大优势在于支持在任意位置插入、删除和移动元素，对 list 来说，在哪个位置进行操作并没有区别。list 在部分算法（如 sorting）中的效率可能优于其它顺序容器。

list 的主要缺点是不支持元素的随机访问！如果我们想要访问某个元素，则必须从一个已知元素（如 begin 或 end）开始朝一个方向遍历，直至到达要访问的元素。此外，list 还要消耗更多的内存空间，用于保存各个元素的关联信息。

list 对内存空间的使用效率并不高，一方面元素内存地址是离散的而非连续，另一方面，list 需要保存额外的关联信息。

## forward_list
```
Container properties: Sequence | Linked list | Allocator-aware
容器属性：顺序容器（可顺序访问，但不支持随机访问），单链表，使用内存分配器动态管理内存 ；
类模板头：template < class T, class Alloc = allocator<T> > class list;
```

forward_list 也是一种支持在任意位置快速插入和删除元素的容器，forward_list 相比于 list 的核心区别是它是一个单链表，因此每个元素只会与相邻的下一个元素关联！由于关联信息少了一半，因此 forward_list 占用的内存空间更小，且插入和删除的效率稍稍高于 list。作为代价，forward_list 只能单向遍历。

相比于其它顺序容器（array, vector, deque），forward_list 的优缺点和 list 基本相同。

既然已经有了 list，为什么 C++ STL 又设计了 forward_list 这一容器呢？设计 forward_list 的目的是为了达到不输于任何一个C风格手写链表的极值效率！为此，forward_list 是一个最小链表设计，它甚至没有size()接口，因为内部维护一个size变量会降低增删元素的效率。如果想要获取 forward_list 的 size，一个通常的做法是，用 std::distance 计算 begin 到 end 的距离得出 size。一句话总结：list 兼顾了接口丰富性牺牲了效率，而 forward_list 舍弃了不必要的接口只为追求极致效率。

## queue
```
容器属性：容器适配器，先进先出型容器（FIFO）；//C++设计模式之适配器模式
template <class T, class Container = deque<T> > class queue;
```
queue（普通队列）是一个专为 FIFO 设计的容器适配器，也即只能从一端插入、从另一端删除；所谓容器适配器，是指它本身只是一个封装层，必须依赖指定的底层容器（通过模板参数中的class Container指定）才能实现具体功能。

容器适配器实际上是C++设计模式的一种 —— 称为 Adapter 模式（适配器模式），Adapter 模式的目的是将第三方库提供的接口做一个封装和转化，使其适配自己工程中预留的接口，或者适应自己工程的调用风格。换句话说，Adapter 模式的目的是将被调用类（如第三方库）的接口转化为希望的接口。

回到正题，queue 可以接纳任何一个至少支持下列接口的容器作为底层容器：
```
empty(); size(); front(); back(); push_back(); pop_front().
```
在标准模板库容器中，deque 和 list 满足上述要求，当然用户也可以自定义一个满足上述要求的容器。通过模板参数可以看出，默认情况下，queue 使用 deque 作为底层容器。

## priority_queue
```
容器属性：容器适配器，严格弱序（Strict Weak Ordering），优先级队列；
template <class T, class Container = vector<T>,
class Compare = less<typename Container::value_type> > class priority_queue;
```
和 queue 类似，priority_queue（术语叫作优先级队列）也只是一个容器适配器，需要指定底层容器才能实例化，参见模板参数中的class Container形参。priority_queue 的核心特点在于其严格弱序特性（strict weak ordering）：也即 priority_queue 保证容器中的第一个元素始终是所有元素中最大的！为此，用户在实例化一个 priority_queue 时，必须为元素类型（class T）重载<运算符，以用于元素排序！

priority_queue 的原理可以用一个大顶堆来解释：priority_queue 在内部维护一个基于二叉树的大顶堆数据结构，在这个数据结构中，最大的元素始终位于堆顶部，且只有堆顶部的元素（max heap element）才能被访问和获取，大顶堆的具体原理可参见任何一本数据结构书籍。

为了支持这种工作原理，priority_queue 对底层容器也是有要求的，priority_queue 的底层容器必须支持随机访问和至少以下接口：
```
empty(); size(); front(); push_back(); pop_back().
```
标准模板库中的 vector 和 deque 能够满足上述需求，默认情况下，priority_queue 使用 vector 作为底层容器。

某种程度上来说，priority_queue 默认在 vector 上使用堆算法将 vector 中元素构造成大顶堆的结构，因此 priority_queue 就是堆 ，所有需要用到堆的位置，都可以考虑使用 priority_queue。priority_queue 默认是大顶堆，用户也可以通过自定义模板参数中的 class Compare 来实现一个小顶堆。

相比于 queue（普通队列）的先进先出/FIFO，priority_queue 实现了最高优先级先出。

## stack
```
容器属性：容器适配器，后进先出型容器（LIFO）；
template <class T, class Container = deque<T> > class stack;
```

stack（栈）是一个专为 LIFO 设计的容器适配器，也即只能从一端插入和删除；作为适配器，需要指定底层容器才能实例化，参见模板参数中的class Container形参。

stack 的特点是后进先出（一端进出），不允许遍历；任何时候外界只能访问 stack 顶部的元素；只有在移除 stack 顶部的元素后，才能访问下方的元素。stack 需要底层容器能够在一端增删元素，这一端也即 stack 的“栈顶”；stack 可以接纳任何一个至少支持下列接口的容器作为底层容器：
```
empty(); size(); back(); push_back(); pop_back().
```
在标准模板库容器中，vector、deque 和 list 满足上述要求，当然用户也可以自定义一个满足上述要求的容器。通过模板参数可以看出，默认情况下，stack 使用 deque 作为底层容器。

stack 容器应用广泛，例如，编辑器中的 undo （撤销操作）机制就是用栈来记录连续的操作。stack 的设计场景和自助餐馆中堆叠的盘子、摞起来的一堆书类似。

## map

```
Container properties: Associative | Ordered | Map | Unique keys | Allocator-aware
容器属性：关联容器，有序，元素类型<key, value>，key是唯一的，使用内存分配器动态管理内存 ；
template < class Key, // map::key_type
class T, // map::mapped_type
class Compare = less<Key>, // map::key_compare
class Alloc = allocator<pair<const Key,T> > // map::allocator_type
> class map;
```

map 是一个关联型容器，其元素类型是由 key 和 value 组成的 std::pair，实际上 map 中元素的数据类型正是 typedef pair<const Key, T> value_type;，这就看的很清楚了。

所谓关联容器，是指对所有元素的检索都是通过元素的 key 进行的（而非元素的内存地址），map 通过底层的「红黑树」数据结构来将所有的元素按照 key 的相对大小进行排序，所实现的排序效果也是严格弱序特性（strict weak ordering），为此，开发者需要重载 key 的<运算符或者模板参数中的 class Compare。所提到的红黑树是一种自平衡二叉搜索树，它衍生自B树，这里推荐两篇文章（文章1，文章2）作为更深入的参考。

大体来说，map 访问元素的速度要稍慢于下文的 unordered_map，这是因为虽然都叫“map”，但两者的底层机制完全不一样。但是，相比于后者，map 支持在一个子集合上进行直接迭代器访问，原因在于 map 中的元素是被有序组织的。

最后，map 也支持通过operator[]的方式来直接访问 value。

## multimap
```
Container properties: Associative | Ordered | Map | Multiple equivalent keys | Allocator-aware
容器属性：关联容器，有序，元素类型<key, value>，允许不同元素key相同，使用内存分配器管理内存 ；
template < class Key, // map::key_type
class T, // map::mapped_type
class Compare = less<Key>, // map::key_compare
class Alloc = allocator<pair<const Key,T> > // map::allocator_type
> class map;
```
map 中不允许出现 key 相同的两个元素，但 multimap 则可以这样做！

multimap 与 map 底层原理完全一样，都是使用「红黑树」对元素数据按 key 的比较关系，进行快速的插入、删除和检索操作；所不同的是 multimap 允许将具有相同 key 的不同元素插入容器（这个不同体现了 multimap 对红黑树的使用方式的差异）。在 multimap 容器中，元素的 key 与元素 value 的映射关系，是一对多的，因此，multimap 是多重映射容器。

注意，在向 multimap 中新增元素时，multimap 只会判断 key 是否相同，而完全不会判断 value 是否相同！这意味着如果相同的 <key, value> 插入了多次，multimap 会对它们悉数保存！

在使用中，我们可以通过迭代器配合 lower_bound() 和 upper_bound() 来访问一个 key 对应的所有 value，也可以使用equal_range()来访问一个 key 对应的所有 value，也可以通过find()配合count()来访问一个 key 对应的所有 value，个人认为前两种方法使用起来更方便一点。

下文中将要提到的 multiset 之于 set 类似于这里的 multimap 之于 map。

## set
```
Container properties: Associative | Ordered | Set | Unique keys | Allocator-aware
容器属性：关联容器，有序，元素自身即key，元素有唯一性，使用内存分配器动态管理内存 ；
template < class T, // set::key_type/value_type
class Compare = less<T>, // set::key_compare/value_compare
class Alloc = allocator<T> // set::allocator_type
> class set;
```

set 是一个关联型容器，和 map 一样，它的底层结构是「红黑树」，但和 map 不一样的是，set 是直接保存 value 的，或者说，set 中的 value 就是 key。

set 中的元素必须是唯一的，不允许出现重复的元素，且元素不可更改，但可以自由插入或者删除。

由于底层是红黑树，所以 set 中的元素也是严格弱序（strict weak ordering）排序的，因此支持用迭代器做范围访问（迭代器自加自减）。

实际使用中，set 和 map 是近亲，性能相似，他们的差别是元素的 value 本身是否也作为 key 来标识自己。

## multiset
```
Container properties: Associative | Ordered | Set | Multiple equivalent keys | Allocator-aware
容器属性：关联容器，有序，元素自身即key，允许不同元素值相同，使用内存分配器动态管理内存 ；
template < class T, // multiset::key_type/value_type
class Compare = less<T>, // multiset::key_compare/value_compare
class Alloc = allocator<T> > // multiset::allocator_type
> class multiset;
```
multiset 之于 set 就如同 multimap 之于 map：

multiset 和 set 底层都是红黑树，multiset 相比于 set 支持保存多个相同的元素；

multimap 和 map 底层都是红黑树，multimap 相比于 map 支持保存多个key相同的元素。

鉴于以上近亲关系，multiset 的性能特点与其它三者相似，不再赘述。

## unordered_map
```
Container properties: Associative | Unordered | Map | Unique keys | Allocator-aware
容器属性：关联容器，无序，元素类型<key, value>，key是唯一的，使用内存分配器动态管理内存 ；
template < class Key, // unordered_map::key_type
class T, // unordered_map::mapped_type
class Hash = hash<Key>, // unordered_map::hasher
class Pred = equal_to<Key>, // unordered_map::key_equal
class Alloc = allocator< pair<const Key,T> > // unordered_map::allocator_type
> class unordered_map;
```
unordered_map 和 map 一样，都是关联容器，以键值对儿 <key, value> 作为元素进行存储；但是，除此之外，两者可以说是完全不一样！

这是由底层的数据结构决定的，map 以红黑树作为底层结构组织数据，而 unordered_map 以哈希表（hash table）作为底层数据结构来组织数据，这造成了两点重要影响：1. unordered_map 不支持排序，在使用迭代器做范围访问时（迭代器自加自减）效率更低；2. 但 unordered_map 直接访问元素的速度更快（尤其在规模很大时），因为它通过直接计算 key 的哈希值来访问元素，是O(1)复杂度！

网络上有对 map VS unordered_map 效率对比的测试，通常 map 增删元素的效率更高，unordered_map 访问元素的效率更高，可以参见这篇文章。另外，unordered_map 内存占用更高，因为底层的哈希表需要预分配足量的空间。

综上，unordered_map 更适用于增删操作不多，但需要频繁访问，且内存资源充足的场合。

## unordered_multimap
```
Container properties: Associative | Unordered | Map | Multiple equivalent keys | Allocator-aware
容器属性：关联容器，无序，元素类型<key, value>，允许不同元素key相同，使用内存分配器管理内存 ；
template < class Key, // unordered_multimap::key_type
class T, // unordered_multimap::mapped_type
class Hash = hash<Key>, // unordered_multimap::hasher
class Pred = equal_to<Key>, // unordered_multimap::key_equal
class Alloc = allocator< pair<const Key,T> > // unordered_multimap::allocator_type
> class unordered_multimap;
```
unordered_multimap 是对 unordered_map 的拓展，唯一区别在于 unordered_multimap 允许不同元素的 key 相同，但两者无论是在底层结构还是在容器特性上都是相通的，仅仅是对底层哈希表的使用方式稍有不同。

在 unordered_multimap 中想要访问同一个 key 下对应的所有元素的话，可以使用equal_range()轻松做到；当然，也可以使用find()和count()配合的方式来访问。

unordered_multimap 的容器特性参见 unordered_map，不再赘述。

## unordered_set

```
Container properties: Associative | Unordered | Set | Unique keys | Allocator-aware
容器属性：关联容器，无序，元素自身即key，元素有唯一性，使用内存分配器动态管理内存 ；
template < class Key, // unordered_set::key_type/value_type
class Hash = hash<Key>, // unordered_set::hasher
class Pred = equal_to<Key>, // unordered_set::key_equal
class Alloc = allocator<Key> // unordered_set::allocator_type
> class unordered_set;
```
所有unordered_XXX类容器的特点都是以哈希表作为底层结构；所有 XXX_set 类容器的特点都是「元素自身也作为key」来标识自己。我们在把两类特性叠加到一起，就得到了 unordered_set。

在 unordered_set 中，元素自身同时也作为 key 使用；既然是作为 key 使用，那么元素就不能被更改，也即 unordered_set 中的元素都是 constant 的，但我们可以自由的插入和删除元素，这也是所有XXX_set类容器的性质。既然底层结构是哈希表，意味着 unordered_set 中的元素是无序的，不能按照大小排序，这也是所有unordered_XXX类容器的性质。

和所有的unordered_XXX类容器一样：1. unordered_set 直接用迭代器做范围访问时（迭代器自加自减）效率更低，低于 set；2. 但 unordered_set 直接访问元素的速度更快（尤其在规模很大时），因为它通过直接计算 key 的哈希值来访问元素，是O(1)复杂度！

## unordered_multiset
```
Container properties: Associative | Unordered | Set | Multiple equivalent keys | Allocator-aware
容器属性：关联容器，无序，元素自身即key，允许不同元素值相同，使用内存分配器动态管理内存 ；
template < class Key, // unordered_multiset::key_type/value_type
class Hash = hash<Key>, // unordered_multiset::hasher
class Pred = equal_to<Key>, // unordered_multiset::key_equal
class Alloc = allocator<Key> // unordered_multiset::allocator_type
> class unordered_multiset;

```

## pair & tuple
```
template <class... Types> class tuple;
template <class T1, class T2> struct pair;
```
std::pair 和 std::tuple 并不是stl容器库中的容器，不过鉴于经常用到，就顺便整理一下。先从 tuple 说起，pair 相当于 tuple 的特例。

tuple 叫作元组，它可以把一组类型相同或不同的元素组合到一起，且元素的数量不限。tuple 的底层原理与 stl 中的容器完全不同，但在功能上，tuple 是对容器的有效补充，因为所有的容器都只能组合相同类型的元素，但tuple 可以组合任意不同类型的元素。在使用上，可以用std::make_tuple()来构造 tuple 对象，可以用std::get<index>()来获取 tuple 对象的某个元素，注意std::get<index>()返回的是 tuple 对象中某个元素的索引，因此是可以用作左值的！此外，也可以用std::tie()打包一组变量来作为左值接受 tuple 对象的赋值。

tuple 的底层原理大概是一个层层继承的类，详情可以参考这篇文章，写的非常透彻。

pair 可以看作是把 tuple 的 size 限制为 2 的一个特例，pair 只能把一对儿元素组合到一起。在使用上，可以用std::make_pair()来直接构建 pair 对象，可以用std::get<0>()和std::get<1>()来分别获取 pair 对象的两个元素，但更方便的做法是直接访问 pair 类型的两个数据成员pair对象.first和pair对象.second来访问元素。
