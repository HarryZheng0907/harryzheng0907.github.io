# 在Yii中使用Memcache做缓存遇到的坑

近日接到一位同事的求助，他在Yii中使用Memcache（跟PHP代码不在同一台服务器）作为缓存，但是在给缓存增加过期时间后，缓存就不见了，下面是他的代码

	Yii::$app->cache->set($key, $data, 60);

这句代码看起来没有什么问题，也没有什么特别之处，所以我继续去看Yii的源码

源码地址：vendor\yiisoft\yii2\caching\MemCache.php

    protected function setValue($key, $value, $duration)
    {
        //$expire = $duration > 0 ? $duration + time() : 0;
        $expire = $duration > 0 ? $duration : 0;

        return $this->useMemcached ? $this->_cache->set($key, $value, $expire) : $this->_cache->set($key, $value, 0, $expire);
    }

从源代码上面看Yii是将传入的过期时间转为时间戳后传入了Memcache，那传入时间戳是否有问题呢

https://github.com/memcached/memcached/blob/master/doc/protocol.txt

> Some commands involve a client sending some kind of expiration time
> (relative to an item or to an operation requested by the client) to
> the server. In all such cases, the actual value sent may either be
> Unix time (number of seconds since January 1, 1970, as a 32-bit
> value), or a number of seconds starting from current time. In the
> latter case, this number of seconds may not exceed 60*60*24*30 (number
> of seconds in 30 days); if the number sent by a client is larger than
> that, the server will consider it to be real Unix time value rather
> than an offset from current time.

通过查看官方文档可以看到，memcache允许传入时间戳或者从当前时间到过期时间的秒数，当传入的数字小于 60*60*24*30(30天)时就当秒数处理，超出这个秒数就当时间戳处理

所以从这里看出，传入时间戳是没问题的，那有没有可能是memcache的时间与PHP服务器时间不一致呢

	[harry@localhost ~]$ telnet 1.2.3.4 11211
	Trying 1.2.3.4...
	Connected to 1.2.3.4.
	Escape character is '^]'.
	stats
	STAT pid 1467
	STAT uptime 4513364
	STAT time 1471594732
	STAT version 1.4.4
	STAT pointer_size 64
	STAT rusage_user 219.273665
	STAT rusage_system 619.466826
	STAT curr_connections 146
	STAT total_connections 259600
	STAT connection_structures 366
	STAT cmd_get 4370031
	STAT cmd_set 401128
	...

通过telnet连上memcache，然后通过stats指令我们拿到了memcache当前的时间戳，再对比php服务的当前时间，发现是memcache的时间戳快了6.5个小时

也就是说，如果我们的过期时间设置在6.5个小时内，那对memcache来说，都是已经过期的时间，所以缓存就丢失了

为了验证这种情况，我重新设置了两个缓存，一个过期时间6个小时，另外一个过期时间7个小时

	Yii::$app->cache->set($key, $data, 60*60*6);
	Yii::$app->cache->set($key, $data, 60*60*7);

运行后发现，果然过期时间为7小时的key没有消失，但是过期时间为6小时的key则消失了

从目前的情况看，基本可以确认是memcache的当前时间戳与php服务器的时间戳不一致，导致了设置过期时间后，缓存丢失的问题。

那现在要解决的问题就是为什么memcache的时间会错，以及如果改正的问题

我最初的想法是memcache的当前时间应该是跟他所在服务器的时间是一致的，但是所在服务器的时间设置有误，导致memcache的时间也是错误的，所以登录memcache所在的服务器查看当前时间戳

	[harry@localhost ~]$ date +%s
	1471598689

但是用date查询所在服务器的当前时间戳后，发现所在服务器的当前时间戳是没错的

那这个问题就很奇怪了，memcache的时间不跟所在服务器的时间一致，那他的时间是哪来的呢，这时候果断google一下。

https://code.google.com/archive/p/memcached/issues/284

> memcached internally uses a monotonic clock, so the time changing outside of it won't change and break relative object expirations. Except if your clock is off when it starts, it'll be stuck that way.
> 
> I assume that's what's going on here, anyway.

最后google到了这样一段话，memcache有内部时钟，不会受外部时间的影响，那就是说memcache的时间不会受系统时间的影响，按照这个逻辑，那我修改系统时间就不会影响到memcache了，为了验证这个问题，我修改了memcache所在服务器的系统时间，但是我发现memcache的时间也会跟着变的，所以说这段话可能有问题，又或者是我的理解有问题。

最后我们通过重启memcache的方式让memcache的时间恢复正常，但仍不清楚memcache时间出错的原因