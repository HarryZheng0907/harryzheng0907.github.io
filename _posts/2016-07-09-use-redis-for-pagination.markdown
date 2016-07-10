---
layout:     post
title:      "使用Redis缓存分页数据"
date:       2016-02-29 12:00:00
author:     "Harry"
header-img: "img/post-bg-2015.jpg"
tags:
    - Redis
---

# 使用Redis缓存分页

项目中经常会遇到需要对某些访问量较高的分页数据做缓存的情况，下面讲讲这几天考虑到的几种分页方案。

### 通过List缓存数据

当有新数据时，将新数据推入List中，并且使用ltrim删除旧的数据，达到缓存前N条数据的效果。

```
$redis->lpush("data",json_encode($object));
//缓存50条数据
$redis->ltrim("data",0,50);
```

获取缓存数据

```
$redis->lrange("data",($page-1)*pageSize,$page*$pageSize)
```

这种方法无法修改已经存入缓存中的数据，也就是说，如果分页中的数据有修改，只能删除所有的分页数据，下面我们继续改进这个方法。

### 结合String跟List缓存数据
为了克服这个问题，我们增加String类型来帮助我们缓存数据，List只保存分页数据的ID，由String保存具体的值，这样的话，当缓存中有数据发生改变时，只需要修改对应的缓存数据，而不影响其他数据。

```
//list中只存储id
$redis->lpush("data",$object->id);
//以id为key保存整个对象的数据
$redis->set("object:{$object->id}",json_encode($object));
//缓存50条数据
$redis->ltrim("data",0,50);
```

获取缓存

```
$ids = $redis->lrange("data",($page-1)*$pageSize,$page*$pageSize)
$redis->mget($ids);
```

通过这种方式，我们成功的解决了缓存数据不可修改的问题，但是仔细观察这个方案，仍然存在两个缺陷。
1. 由于List没有去重的功能，所以高并发下，容易产生重复的数据。
2. 由于影响List中元素的位置只有进入List的顺序，所以一般只适用于按新增时间排序的列表。

针对这两个问题，我们继续进行改进
### 结合String跟SortedSet缓存数据
通过观察上面的方案我们发现存在的缺陷都是由于List的特性引起的，所以我们需要使用另外一种数据结构来存储分页数据的ID，下面让我们用SortedSet来尝试解决这个问题。

```
//以新增时间作为score，已id作为member
$redis->zadd("dataSortByCreated",$object->created,$object->id); 
//以新增时间作为score, 已id作为member
$redis->zadd("dataSortByUpdated",$object->updated,$object->id); 
//以id为key保存整个对象的数据
$redis->set("object:{$object->id}",json_encode($object))
```

获取缓存

```
//以新增时间排序获取分页id
$idSortByCreated = $redis->zrevrange("dataSortByCreated",($page-1)*pageSize,$page*$pageSize);
//以修改时间排序获取分页id
$idSortByUpdated = $redis->zrevrange("dataSortByUpdated",($page-1)*pageSize,$page*$pageSize);
$redis->mget($idSortByCreated);
$redis->mget($idSortByUpdated);
```

利用SortedSet的特性，我们可以将缓存数据的ID以不同的维度保存，从而实现按不同维度排序的分页

通过方案的持续优化，我们终于得到了一种相对完善的分页缓存方案，但不意味着一开始的两种方案就不能用，还是需要根据不同的场景灵活运用

>**注意** 为了突出思路，这里的方案隐藏了很多细节，只能当初一种思路，不能直接复制代码




