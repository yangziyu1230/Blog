---
title: 'STL中std::sort的实现'
author: 'Dagest'
tags:
  - c++
  - stl
categories:
  - c++
date: 2022-10-18 09:40:16
---

## 背景

在刷洛谷水题的过程中，有一道练习写[快速排序](https://www.luogu.com.cn/problem/P1177)的题；这种小题哪能难倒我！很快啊，我就写出来了。

```cpp
#include <iostream>
#include <vector>
#include <cstdlib>

using namespace std;

void swap(int &a, int &b)
{
    int tmp = a;
    a = b;
    b = tmp;
}

void sort(int *nums, int start, int end)
{
    if (start >= end)
    {
        return;
    }
    //pivot选择为中间位置数字
    swap(nums[start], nums[rand() % (end - start + 1) + start]);
    int i = start;
    int j = end;
    int base = nums[start];
    while (i < j)
        {
            while (nums[j] >= base && i != j)
                {
                    j--;
                }

            while (nums[i] <= base && i != j)
                {
                    i++;
                }
            swap(nums[i], nums[j]);
        }
    swap(nums[start], nums[i]);
    sort(nums, start, i - 1);
    sort(nums, i + 1, end);
}

int nums[100001];
int main(int argc, char const *argv[])
{
    int n;
    cin >> n;
    for (int i = 0; i < n; i++)
    {
        int tmp;
        cin >> tmp;
        nums[i] = tmp;
    }
    sort(nums, 0, n - 1);

    for (int i = 0; i < n; i++)
    {
        cout << nums[i] << ' ';
    }
    cout << endl;

    return 0;
}
```

但是一提交，好家伙，最后一个点竟然超时了，开o2优化勉强能过。这我肯定忍不了啊，下载下来测试点输入看了一下，发现都是相同的数；众所都周知如果都是相同的数，快速排序会触发最坏的情况，时间复杂度来到$O(n^2)$。看题目说STL的sort可以直接过，试了一下果然快了亿点点，于是抱着学习的心态研究下STL中的sort是如何实现的。

## 实现

STL的sort函数由stl_algo.h头文件所提供，其具体实现为__sort函数，__sort函数实现如下所示：

```cpp
/**
* __first 所需排序空间的起始迭代器
* __end 所需排序空间的终止迭代器（不包括）
* __comp 比较器
*/
template<typename _RandomAccessIterator, typename _Compare>
inline void
__sort(_RandomAccessIterator __first, _RandomAccessIterator __last,
_Compare __comp)
{
    if (__first != __last)
    {
        std::__introsort_loop(__first, __last,
            std::__lg(__last - __first) * 2,
            __comp);
        std::__final_insertion_sort(__first, __last, __comp);
    }
}
```

在开始，__sort函数会先判断区间有效性，接着调用__introsort_loop函数。在__introsort_loop函数结束之后调用__final_insertion_sort函数。我们接下来紧接着看__introsort_loop函数：

```cpp
/**
* __first 所需排序空间的起始迭代器
* __end 所需排序空间的终止迭代器（不包括）
* __depth_limit 递归深度
* __comp 比较器
*/
enum { _S_threshold = 16 };
template<typename _RandomAccessIterator, typename _Size, typename _Compare>
void
__introsort_loop(_RandomAccessIterator __first,
_RandomAccessIterator __last,
_Size __depth_limit, _Compare __comp)
{
    while (__last - __first > int(_S_threshold))
        {
            if (__depth_limit == 0)
            {
                std::__partial_sort(__first, __last, __last, __comp);
                return;
            }
            --__depth_limit;
            _RandomAccessIterator __cut =
            std::__unguarded_partition_pivot(__first, __last, __comp);
            std::__introsort_loop(__cut, __last, __depth_limit, __comp);
            __last = __cut;
        }
}
```

我们可以看到这是一个递归函数。我们会通过某些操作将序列以__cut为界限分割为两部分，对其中[__cut, __last)进行递归调用，同时将__last左移（第25行__last = __cut），使用while循环继续对左半部分进行处理。
此处在处理[__first, __cut)区间时并没有像[__cut, __last)一样进行递归调用，原因在于使用循环处理可以有效减少函数调用次数，减少开销。如下图所是都通过递归调用和__introsort_loop实现的具体过程：

![](https://blog-1255608703.cos.ap-hongkong.myqcloud.com/stl-sort/__introsort_loop.png)

我们可以看到使用右边的方式可以减少一半的递归调用。若数据量非常庞大，节省的函数调用开销会非常可观。
现在先忽略while和if的条件，先看看__unguarded_partition_pivot是怎么实现的：

```cpp
/**
* __first 所需排序空间的起始迭代器
* __end 所需排序空间的终止迭代器（不包括）
* __comp 比较器
*/
template<typename _RandomAccessIterator, typename _Compare>
inline _RandomAccessIterator
__unguarded_partition_pivot(_RandomAccessIterator __first,
_RandomAccessIterator __last, _Compare __comp)
{
    _RandomAccessIterator __mid = __first + (__last - __first) / 2;
    std::__move_median_to_first(__first, __first + 1, __mid, __last - 1,
        __comp);
    return std::__unguarded_partition(__first + 1, __last, __first, __comp);
}
```

这个方法就已经很清晰明了，首先是选取[__first, __last)区间中间位置的数，将其移动到__first的位置，紧接着调用__unguarded_partition函数，我们再来看下__unguarded_partition函数：

```cpp
/**
* __first 所需排序空间的起始迭代器
* __end 所需排序空间的终止迭代器（不包括）
* __pivot 快排分割点
* __comp 比较器
*/
template<typename _RandomAccessIterator, typename _Compare>
_RandomAccessIterator
__unguarded_partition(_RandomAccessIterator __first,
_RandomAccessIterator __last,
_RandomAccessIterator __pivot, _Compare __comp)
{
    while (true)
    {
        while (__comp(__first, __pivot))
            ++__first;
        --__last;
        while (__comp(__pivot, __last))
            --__last;
        if (!(__first < __last))
            return __first;
        std::iter_swap(__first, __last);
        ++__first;
    }
}
```

很明显这就是对[__first, __last)区间使用__comp进行快排，所以我们可以得出结论：在STL中仍以快速排序作为实现sort的主体。

我们继续回到__introsort_loop函数，关注while和if条件。我们注意到__depth_limit会在每回调用函数时候进行--操作，那不难猜出此参数就是用来表达递归深度的；当递归深度过深，也就是__depth_limit变为0的情况下，为防止分割行为恶化，我们会对剩余部分执行__partial_sort：

```cpp
template<typename _RandomAccessIterator, typename _Compare>
inline void
__partial_sort(_RandomAccessIterator __first,
_RandomAccessIterator __middle,
_RandomAccessIterator __last,
_Compare __comp)
{
    std::__heap_select(__first, __middle, __last, __comp);
    std::__sort_heap(__first, __middle, __comp);
}

template<typename _RandomAccessIterator, typename _Compare>
void
__heap_select(_RandomAccessIterator __first,
_RandomAccessIterator __middle,
_RandomAccessIterator __last, _Compare __comp)
{
    std::__make_heap(__first, __middle, __comp);
    for (_RandomAccessIterator __i = __middle; __i < __last; ++__i)
        if (__comp(__i, __first))
            std::__pop_heap(__first, __middle, __i, __comp);
}

template<typename _RandomAccessIterator, typename _Compare>
void
__sort_heap(_RandomAccessIterator __first, _RandomAccessIterator __last,
_Compare& __comp)
{
    while (__last - __first > 1)
    {
        --__last;
        std::__pop_heap(__first, __last, __last, __comp);
    }
}
```

很明显，这是堆排序算法。当递归深度过深时，使用heap sort将可能出现的最坏情况$O(n^2)$优化为$O(nlogn)$，排序完成后退出当前循环。有意思的是关于__depth_limit初始值的确定，在我这边使用如下代码进行计算，至于为什么这样计算我就懒得研究了。

```cpp
/**
* __CHAR_BIT__： 当前计算机char类型所产位数
* __builtin_clzll(unsigned long long)： 计算参数二进制表示末尾连续0的个数
*/
inline _GLIBCXX_CONSTEXPR unsigned long long
__lg(unsigned long long __n)
{ return sizeof(long long) * __CHAR_BIT__ - 1 - __builtin_clzll(__n); }
__depth_limit = std::__lg(__last - __first) * 2；
```

在while条件中会判断待排序区间的长度，若长度小于16则跳出当前循环，继续进行__final_insertion_sort进行最后的处理。当进行到这一步时整个区间已经被分为了一个个长度不超过16的相对有序的子序列，众所都周知对相当有序的子序列进行排序若使用插入排序时间复杂度最优能达到$O(n)$；STL也是这样实现的。

我们来看__final_insertion_sort的实现：

```cpp
template<typename _RandomAccessIterator, typename _Compare>
void
__final_insertion_sort(_RandomAccessIterator __first,
_RandomAccessIterator __last, _Compare __comp)
{
    if (__last - __first > int(_S_threshold))
    {
        std::__insertion_sort(__first, __first + int(_S_threshold), __comp);
        std::__unguarded_insertion_sort(__first + int(_S_threshold), __last,
            __comp);
    }
    else
        std::__insertion_sort(__first, __last, __comp);
}
```

可以发现，STL将不同的情况走了不同的函数调用，但是本质还是插入排序。其实到这里我们就能够解释所遇到的问题了。

## 结语

在我所写的快排算法中若出现全部相同元素，算法的时间复杂度会退化为$O(n^2)$，并且由于其递归深度过深，函数切换存在大量的开销。在std::sort中，若出现这种情况则会由于递归深度过深，先进行堆排序。在堆排序过程中因为其叶子结点都不大于其父节点，所以可以省去大量的上浮操作，其时间复杂度为$O(nlogn)$。在最后进行插入排序时因为其所有节点都相等，所以其时间复杂度为$O(n)$，并且由于从快排退化成了堆排序，节省了大量的函数调用开销；消耗时间上有大量减少也不足为奇。

在进行搜索过程中发现STL所用算法为内省排序，是由Musser在1996年发布，有兴趣的可进一步浏览[此页面](http://www.cs.rpi.edu/~musser/gp/index_1.html)。

## 待解决的问题

- __depth_limit确定依据
- __final_insertion_sort中为何要分情况讨论，__unguarded_insertion_sort和__insertion_sort有何区别