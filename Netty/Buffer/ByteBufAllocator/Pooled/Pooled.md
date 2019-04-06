# Buffer 之 ByteBufAllocator（三）PooledByteBufAllocator

## 1. 概述

本文，我们来分享 PooledByteBufAllocator ，基于**内存池**的 ByteBuf 的分配器。而 PooledByteBufAllocator 的内存池，是基于 **Jemalloc** 算法进行分配管理，所以在看本文之前，先跳到 <<Buffer 之 Jemalloc（一）简介>> ，将 Jemalloc 相关的**几篇**文章看完，在回到此处。

