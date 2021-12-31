# 《Python高性能编程》

## 理解高性能Python

### 理想计算模型

```python
import math

def is_prime(number):
    sqrt_number = math.sqrt(number)
    number_float = float(number)
    for i in range(2, int(sqrt_number) + 1):
        if (number_float / i).is_integer():
            return False
    return True


print(is_prime(10000000))
print(is_prime(10000019))
```

理解程序执行

1. `number` 的值保存于 RAM 中
2. 为了计算 `sqrt_number` 和 `number_float` 将 `number` 传给 CPU，保存于CPU的 L1/L2 缓存中
   - 减少从 RAM 读取次数，转而以更多的 L1/L2 缓存读取
   - 减少前端总线传输次数
3. CPU进行两次计算，并将结果传回 RAM 保存
4. `number_float` 和 `sqrt_number` 传给 CPU 缓存
5. 循环部分，每次 `i` 通过总线传给 CPU，CPU计算 `number_float/i` 并检查结果是否为整数，然后通过总线返回结果

优化

在代码的循环部分，与其一次次将i输入CPU，我们更希望一次就将 `number_float` 和多个 `i` 的值输入CPU 进行检查。这是可能的，因为CPU 的矢量操作不需要额外的时间代价， 意味着它可以一次进行多个独立计算。

```python
import math
def is_prime(number):
    sqrt_number = math.sqrt(number)
    number_float = float(number)
    numbers = range(int(sqrt_number) + 1)
    for i in range(0, len(numbers), 5):
        # 以下只是示例，不是合法代码
        result = (number_float / numbers[i:i+5]).is_integer()
        if any(result):
            return False
    return True
```

这里，一次让程序对5个 `i` 的值进行除法和整数检查。当被正确地矢量化时，CPU 仅需一条指令完成这行代码而不是对每个 `i` 进行独立操作。理想情况下，`any(result)` 操作将只发生于 CPU 内部而无须将数据传回RAM。

### Python虚拟机

Python 解释器为了抽离底层用到的计算元素做了很多工作。这让编程人员无须考虑如何为数组分配内存、如何组织内存以及用什么样的顺序将内存传入CPU。这是Python 的一个优势，让你能够集中在算法的实现上。然而它有一个巨大的性能代价。

一、不具备直接进行矢量化操作的能力

- 通过外部库如numpy增加矢量化数学操作解决

二、垃圾回收机制的存在导致内存布局不是最优的

- 内存自动分配与释放，导致内存碎片、影响CPU缓存传输
- 程序员不能直接改变数据结构在内存的中的布局，导致可能一次总线传输并没有包含一次计算所需的所有数据

三、Python是动态类型语言，不是编译性语言

- 编译器在编译过程会做很多优化，如编译静态代码时，可以让CPU运行某些指令优化对象内存布局
- 动态类型，意味着很多算法优化更加难以实现
- 一个解决办法：Cython可以将Python代码进行编译并允许用户“提示”编译器代码究竟有多“动态”

四、GIL（全局解释器锁）影响并行性能

- 由于GIL的存在，Python进程一次只能执行一条指令，无论当前有多少个核心
- 解决方法：使用多进程（multiprocessing模块）而不是多线程，或者使用Cython或外部函数来避免



## 列表和元组

> 写高性能程序最重要的事情是了解你的数据结构所能够提供的性能保证。
>
> 选择一个能够迅速响应这个查询的数据结构。

列表和元组之类的数据结构被称为数组。区别：列表是动态的数组，而元组则是静态的数组。

数据存储

一台计算机的系统内存可以被看作是一系列编了号的桶，每个桶可以存放一个数字。这些数字可以被用来代表任何我们关心的变量（整数、浮点数、字符串，或其他数据结构），因为它们只是引用了数据被保存在内存中的位置。

当我们想要创建一个数组（列表或元组）时，我们首先必须分配一块系统内存（其每一段都将被当成是一个整型大小的指向实际数据的指针）。这需要进入内核，操作系统的子进程，去申请使用 N 个连续的桶。

**以下例子显示了一个大小为6的列表的系统内存布局。注意在 Python 中列表还记录了它们的大小，所以在分配的6个块中，仅可使用5个——第一个元素是列表的长度。**

（反过来说，程序员想申请一个大小为5的列表，系统中将开辟一块大小为6的内存空间，第一块存储列表长度）

![image-20211230175356149](http://image.huawei.com/tiny-lts/v1/images/3050a09aa44381ac66bb7f06ab19caa5_1386x584.png)

查询任意指定元素，时间复杂度O(1)。计算机只需通过起始地址加偏移量取数据即可。

查询未知次序的元素，时间复杂度O(n)。线性搜索，遍历数组每一个元素。

优化：

了解存放数据的桶的组织方式：对数据排序，然后二分搜索。时间复杂度O(logn)。

了解数据如何存放在内存中：如散列表，一个用于实现字典和集合的基本数据结构，通过丢弃数据的原始次序并指定另外一种次序。时间复杂度O(1)。

### 更有效的搜索

高效搜索必需的两大要素是排序算法和搜索算法。**Python 列表有一个内建的排序算法使用了Tim 排序**。Tim 排序可以在**最佳情况下以O(n)（最差情况下则是O(nlog n)）**的复杂度排序。它**运用了多种排序算法**。对于给定的数据，它**使用探索法猜测哪个算法的性能最优**（更确切地说，它混用了插入排序和合并排序算法）来达到这样的性能。基于此可以使用二分搜索。

另外，使用 bisect 模块进一步简化了二分搜索的流程，它提供了一个简便的函数使可以在保持排序的同时往列表中添加元素，以及一个**高度优化过的二分搜索算法**函数来查找元素。它提供的函数可以将新元素直接插入正确的排序位。在列表始终排序的情况下，我们可以轻松找到需要的元素。另外，我们可以非常迅速地用bisect 找到跟我们的目标值最接近的元素。这个功能对于比较两个相似但不完全一样的数据集来说极其有用。

```python
import bisect
import random


def find_closest(haystack, needle):
    """
    haystack(干草垛): 待搜索的非降序列
    needle(针): 搜索目标
    这两命名常用于搜索中，在干草堆中找针，形容事情很难，如大海捞针。
    """
    # bisect.bisect_left返回haystack中第一个大于等于needle的坐标
    idx = bisect.bisect_left(haystack, needle)
    # haystack中所有元素均比needle大
    if idx == len(haystack):
        return idx - 1
    elif haystack[idx] == needle:
        return idx
    # 检查左边元素是否更近
    elif idx > 0:
        j = idx - 1
        if haystack[idx] - needle > needle - haystack[j]:
            return j
    return idx


important_numbers = []
for _ in range(10):
    bisect.insort(important_numbers, random.randint(0, 1000))

k = random.randint(0, 1000)
# bisect.insort(important_numbers, k)

print(important_numbers)

closest = find_closest(important_numbers, k)
print(f"Closest value to {k}: ", important_numbers[closest])
```

编写高效代码的基本原则：选择正确的数据结构并坚持使用它！虽然对于某个特定操作来说也许还存在更高效的数据结构，但是在这些数据结构之间进行转换的代价可能会抵消效率上的增益。

### 列表和元组

元组缓存于Python运行时环境，意味着每次使用元组时无须访问内核去分配内存。

列表的可变性的代价在于存储它们需要额外的内存以及使用它们需要额外的计算。

#### 动态数组：列表

```python
>>> numbers = [5, 8, 1, 3, 2, 6]	# 初始化数组
>>> numbers.append(42)				# 增加元素
>>> numbers
[5, 8, 1, 3, 2, 6, 42]
```

**列表append操作分配超额空间的机制：**

列表支持增加元素是因为动态数组支持resize操作，可以增加数组的容量。

**当一个大小为N的列表第一次需要添加数据时，Python会创建一个新的列表**，足够存放原来的N个元素以及额外需要添加的元素。不过，实际分配的并不是N+1个元素，而是M个，M > N，这是为了**给未来的添加预留空间**。

因为旧列表空间用尽后，旧列表的数据被复制到新列表中，旧列表则被销毁。从设计理念上来说，**第一次的添加可能会是后续多次添加的开始，通过预留空间的做法，减少这一分配或释放空间的操作的次数以及内存复制的次数**。这一点非常重要，因为内存复制可能非常昂贵，特别是当列表大小开始增长以后。

防止会忽略了前提，再总结说明一下：在内存足够的前提下，最初创建列表时不分配超额空间，第一次 append 扩充列表时，Python 会根据计算公式分配超额空间。

超额空间分配公式：

> new_allocated = (newsize >> 3) + (newsize < 9 ? 3 : 6)

`new_allocated` 是超额分配空间数量，`newsize` 是添加元素后空间大小。

注意：以上公式只在空间不够用时起作用，如列表空间大小在$0\rightarrow1$时，超额分配空间大小为4，则后续3个append操作都不需要改变空间大小。且为了防止内存泄漏，有最大申请限制。

[Python3.7源码](https://github.com/python/cpython/blob/v3.7.0/Objects/listobject.c#L19) 注释参考：

```c
    /* This over-allocates proportional to the list size, making room
     * for additional growth.  The over-allocation is mild, but is
     * enough to give linear-time amortized behavior over a long
     * sequence of appends() in the presence of a poorly-performing
     * system realloc().
     * The growth pattern is:  0, 4, 8, 16, 25, 35, 46, 58, 72, 88, ...
     * Note: new_allocated won't overflow because the largest possible value
     *       is PY_SSIZE_T_MAX * (9 / 8) + 6 which always fits in a size_t.
     */
    new_allocated = (size_t)newsize + (newsize >> 3) + (newsize < 9 ? 3 : 6);
    if (new_allocated > (size_t)PY_SSIZE_T_MAX / sizeof(PyObject *)) {
        PyErr_NoMemory();
        return -1;
    }

    if (newsize == 0)
        new_allocated = 0;
    num_allocated_bytes = new_allocated * sizeof(PyObject *);
    items = (PyObject **)PyMem_Realloc(self->ob_item, num_allocated_bytes);
    if (items == NULL) {
        PyErr_NoMemory();
        return -1;
    }
    self->ob_item = items;
    Py_SIZE(self) = newsize;
    self->allocated = new_allocated;
    return 0;
}
```

思考：pop操作空间如何变化？

额外分配的空间一般来说是非常小的，但累加起来可能会很多。在**维护很多小列表或一个非常大的列表时**，这一效果会十分显著。如果维护1 000 000个列表，每个列表10个元素，假设占用了10 000 000个元素的内存。但如果构建列表时用的append操作，实际占用内存可能是16 000 000个元素。同样，对于一个拥有100 000 000个元素的大列表，实际分配的可能是112 500 007个元素！

#### 静态数组：元组

元组不支持改变大小，但可以将两个元组合并成一个新元组。复制，分配新空间，时间复杂度O(n)。

```python
>>> t1 = (1,2,3,4)
>>> t2 = (5,6,7,8)
>>> t1 + t2
(1, 2, 3, 4, 5, 6, 7, 8)
```



元组的静态特性的另一个好处体现在一些会在Python 后台发生的事：资源缓存。Python 是一门垃圾收集语言，这意味着当一个变量不再被使用时，Python 会将该变量使用的内存释放回操作系统，以供其他程序（或变量）使用。

然而，对于长度为1～20 的元组，即使它们不再被使用，它们的空间也不会立刻被还给系统，而是留待未来使用。这意味着当未来需要一个同样大小的新元组时，我们不再需要向操作系统申请一块内存来存放数据，因为我们已经有了预留的内存。



**元组初始化比列表快5倍：**

元组一个很神奇的地方：它们可以被轻松迅速地创建，因为它们可以避免跟操作系统打交道，而后者很花时间。以下测试显示了初始化一个列表比初始化一个元组慢 5 倍——如果是在一个循环内部，这点差别会很快累加起来！

```python
%timeit l = [0,1,2,3,4,5,6,7,8,9]
58.5 ns ± 0.11 ns per loop (mean ± std. dev. of 7 runs, 10000000 loops each)
%timeit l = (0,1,2,3,4,5,6,7,8,9)
11.8 ns ± 0.0478 ns per loop (mean ± std. dev. of 7 runs, 100000000 loops each)
```



## 字典和集合

字典和集合通过散列化技术对无序数据进行组织，使得可以以O(1)时间复杂度查询/插入无序数据。

代价则是占用更多的内存。同时，虽然插入/查询的复杂度是O(1)，但实际的速度极大取决于其使用的散列函数。如果散列函数的运行速度较慢，那么在字典和集合上进行的任何操作也会相应变慢。

（字典和集合本质是一样的。集合相当于没有值的字典。集合是一堆键的组合。）

### 字典和集合如何工作

理解：

首先要知道，为什么列表对于指定元素如a[5]的查询能O(1)就完成？

因为只要知道a的起始地址，cpu计算一下第5个元素的相对偏移量，相加即可得到a[5]所在内存的位置！

但要在列表中找某个值的元素时，因为不知道相对位置，所以只能遍历，花销是O(n)。

而字典和集合，解决的就是如何快速得到相对位置的问题。通过散列函数，将元素值转化为坐标，将元素存储在相应位置。查询时，cpu只需对查询值输入散列函数计算一下，即可得到相对偏移量，即可得到查询对象所在内存位置！查询时间复杂度O(1)！



## 迭代器和生成器

### 生成器

一段代码了解生成器

（Python3中取消了xrange，使用range代替，返回的是一个可迭代对象）

```python
# 简单实现range和xrange

def mrange(start, stop, step=1):
    # 返回一个列表
    numbers = []
    while start < stop:
        numbers.append(start)
        start += step
    return numbers


def xrange(start, stop, step=1):
    # 返回一个生成器
    while start < stop:
        yield start
        start += step
        
for i in range(1,10000):
	pass

for i in xrange(1,10000):
	pass
```

- `yield` 关键字将xrange函数转变成一个生成器，一种可以被重复轮询下一个可用值的函数。

首先注意的是 mrange 需要对列表 append 10000次（时间与空间开销），然后返回整个列表。

而生成器每次只返回一个值，只有当外部代码需求另一个值时，该函数才会继续运行并返回下一个值。

空间性能上，mrange 循环比 xrange 多消耗了 10000 倍内存！

测试时间性能，1000w时，mrange 是 xrange 的1.5倍。

```python
def test_xrange():
    for i in xrange(1, 10000000):
        pass


def test_mrange():
    for i in mrange(1, 10000000):
        pass
    
%timeit test_mrange()
1.12 s ± 9.21 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
%timeit test_xrange()
736 ms ± 2.65 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

mrange 的实现方式，需要为整个数据集分配足够的空间，并为每个元素正确赋值，即使我们每次仅需要一个值。而为列表分配的空间其实并没有什么意义。另外，还有一个潜在问题，mrange 申请的空间不一定/不可能被满足，如 mrange(1, 100 000 000) 会创建一个3.1GB大小的列表！

举一个实例，

> 假设有一个数字的长列表，我们想知道其中有多少个数是3的倍数？
>
> list_of_numbers = [1, 3, 5, 7, 19, .... ]

如果这么写：

```python
divisible_by_three = len([n for n in list_of_numbers if n % 3 == 0])
```

这种写法的问题跟 range 一样。因为我们使用了列表表达式，我们预先生成了一个能被3 整除的数字的列表，仅仅只是为了对其进行计算并丢弃。如果这个列表很长，这可能会导致无意义地分配大量内存——大到根本无法被满足。

相对的，我们可以创建一个满足条件的值的生成器而不是一个列表，就能对内存进行优化。但生成器没有length属性，所以要转换一下：

```
divisible_by_three = sum((1 for n in list_of_numbers if n % 3 == 0))
```

生成器适用于创建数据，普通函数则操作生成的数据。

### 生成器的延迟估值

生成器之所以能够节约内存是因为它只处理当前感兴趣的值。在计算的任意点，都只能访问当前的值，而无法访问数列中的其他元素（这种算法通常称为“单通”或“在线”）。因此需要借助其他模块或函数来帮助解决这一问题。

标准库 itertools ：

- islice: 允许对一个无穷生成器进行切片

- chain: 将多个生成器链接到一起

- takewhile: 给生成器添加一个终止条件

- cycle: 通过不断重复将一个有穷生成器变成无穷

使用生成器来分析大数据集的例子：

假设现在有一个分析函数用于处理时间点数据，数据每秒产生一个，对于过去的 20 年，则有 631 152 000 个数据点！该数据被保存在一个文件中，每秒生成一行，我们无法将整个数据集读入内存。如果我们想要进行一些简单的异常检测，我们可以使用生成器而无须分配任何列表！

异常检测：对于一个格式为“timestamp, value”的数据文件，需要找到所有含有异常值的日期，比该日均值超出3 倍标准差之外的数字被视为异常值。

开始：

1. 伪造数据

```python
def gene
```





































































































































