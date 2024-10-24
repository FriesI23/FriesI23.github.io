---
#  friesi23.github.io (c) by FriesI23
#
#  friesi23.github.io is licensed under a
#  Creative Commons Attribution-ShareAlike 4.0 International License.
#
#  You should have received a copy of the license along with this
#  work. If not, see <http://creativecommons.org/licenses/by-sa/4.0/>.
title: xx
excerpt:
author: FriesI23
date: 1970-01-01 00:00:00 +0000
category:
tags:
  - tag1
  - tag2
---


`二叉查找树 (BST)` <sup>[[1]](#参考资料)</sup> 可能是我们能接触到最早也最简的二叉树结构.
虽然其用于理论上基于树高度的插入, 删除与查找速度, 但由于其并不 "平衡" (e.g. 顺序插入节点),
最坏情况会导致其退化为 `线性链表`, 这显然是不可接受的.

```text
// 二叉树      // 线性链
     b              a
   /   \             \
  a     c             b
                       \
                        c
```

因此人们需要探寻一种能够 "自平衡" 的二叉查找树, `AVL 树` <sup>[[2]](#参考资料)</sup> 因此被提出.
`AVL 树` 相比基础的 `BST` 最大区别是其可以根据树的**高度**进行自平衡.

`AVL 树` 是提出的第一种可以自平衡的二叉查找树, 虽然其不是严格平衡的,
但最坏 `h=O(logn)` 的高度使 `AVL 树` 在进行插入, 删除和查找操作时，时间复杂度保持在 `O(logn)`.

由于这篇文章并不是介绍 `AVL 树` 的操作和实现, 如果感兴趣推荐 ["Hello 算法 - AVL 树"][avl-impl] 这篇文章,
这里只是简单复述一下其性质 (后面会用到):

> 术语:
>
> - 高度 (Height): 指当前节点到其最远叶子节点的距离.
> - 平衡因子(Balance factor): 指当前节点左子树高度减去右子树高度, 即 `BF(x) = Height(RS(x)) - Height(LS(x))`
>
> 性质:
>
> - 一棵 `AVL 树` 任意节点的 `平衡因子` 只能是 `-1`, `0` 或 `1`.

在 `AVL 树` 中插入或删除一个节点可能会导致其性质无法保存, 此时就需要通过 `旋转 (Rotaion)` 来递归的修复问题.

## 参考资料

1. [P. F. Windley (1960). "Trees, Forests and Rearranging" The Computer Journal, Volume 3, Issue 2, Pages 84–88][paper-bst]
2. [Adelson-Velsky, Georgy; Landis, Evgenii (1962). "An algorithm for the organization of information". Proceedings of the USSR Academy of Sciences (in Russian)][paper-avl]

<!-- refs -->

[paper-bst]: https://academic.oup.com/comjnl/article/3/2/84/504799?login=false
[paper-avl]: https://zhjwpku.com/assets/pdf/AED2-10-avl-paper.pdf
[avl-impl]: https://www.hello-algo.com/chapter_tree/avl_tree/