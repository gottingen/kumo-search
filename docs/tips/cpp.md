cpp tips
==========================

# TIPS: 高性能容器选择

通常情况下，`turbo::flat_hash_*` 在不进行频繁创建和销毁的情况下，`turbo::flat_hash_*` 系列容器是最快的。
当数据集比较小，作为临时容器使用时，`std::unordered_*` 也是一个不错的选择。这点在开发 `mizar` 时有所体会。


# TIPS: stack vector vs std::vector(heap vector)

`std::vector` 是一个动态数组，它的内存是在堆上分配的，而 `stack_vector` 是一个栈上的数组，它的内存是在栈上分配的。
在`turbo`中，我们提供了`SmallVector`，`SmallVector`是一个栈上的数组，它的内存是在栈上分配的，当数据量比较小的时候，`SmallVector`是一个不错的选择。
`SmallVector`的内存分配策略是：当数据量小于`N`时，数据是在栈上分配的，当数据量大于`N`时，数据是在堆上分配的。`SmallVector`有更好的内存局部性，更好的性能。
同时，在服务中，也能减少内存碎片，提高内存利用率。