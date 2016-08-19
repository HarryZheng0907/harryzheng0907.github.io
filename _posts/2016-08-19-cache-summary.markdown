---
layout:     post
title:      "Redis缓存使用总结"
date:       2016-08-19 17:37:00
author:     "Harry"
header-img: "img/post-bg-2015.jpg"
tags:
    - Redis
---

# Redis缓存使用总结

### 缓存单条数据

缓存单条数据是缓存使用中最常见也是最简单的场景，但是这个场景下也有一些需要注意的地方

第一个版本

	//获取缓存
	$value = $redis->get($key);
	//如果缓存中不存在则去数据查找缓存
    if (is_null($value)) {
		//通过callback函数从数据库中获取数据
        $value = call_user_func_array($callback, $params);
		//如果存在则存入缓存中
        if ($value) {
            $value = $this->serialize($value);
            $redis->set($key, $value，'EX', 60);
        }
    }

这个写法是使用缓存时非常典型的写法，但是这其中存在一些问题：

- 缓存穿透，当查询的数据不存在时，每次请求都会继续去查数据库，这种情况下缓存就起不到应有的作用
- 有覆盖最新数据的隐患，简单的用set将缓存写入，如果在查询数据跟写入缓存之间，由于数据的修改导致缓存更新，那查询回来的旧数据将覆盖掉新数据

针对以上两点，进行了第二版的改造

	//获取缓存
	$value = $redis->get($key);
	//如果缓存中不存在以及没有空标记则去数据查找缓存
    if (is_null($value) && in_null($redis->get($key.':empty'))) {
		//通过callback函数从数据库中获取数据
        $value = call_user_func_array($callback, $params);
		//如果存在则存入缓存中
        if ($value) {
            $value = $this->serialize($value);
            alue = is_null($redis->set($key, $value, 'EX', $expires, 'NX')) ? $redis->get($key) : $value;
        }else{
			//如果查询数据库仍然不存在，则设置一个空标记，表示数据库中也不存在该记录，防止缓存穿透
			$redis->set($key.':empty',1)
		}
    }

在这一版中，通过设置空标记的方式解决由于数据不存在导致的缓存穿透的问题，并且通过为set指令增加NX标识，来避免覆盖最新数据，
但是要注意的，当数据库有数据插入时，需要删除对应的空标记

> 为set指令增加NX的作用是当要set的key已存在时，不再写入，并且返回null

通过第二版的改造基本上可以满足需求了，但是有很重要的一点，就是存什么数据的问题
假设我们现在要一个API，返回用户信息，用户信息包括基本信息、浏览记录、购买信息。
如果我们将这些信息存在一个Key里面，那当这些信息发生变化的时候，我们都需要更新这个key，那这个问题将变得非常复杂
所以建议当需要存储复合数据时，最好按照数据表存储在不同的key中，需要数据再从不同的key中获取数据然后组合起来

### 缓存分页数据

缓存分页数据由于篇幅较多，不在这里赘述

### 缓存集合数据

##### 应用场景

- 有谁中过奖
- 有多少好友
- 有多少人帮助点赞

##### 具体范例

	private function checkWinningLimit($shop_id, $activity_id, $uid)
	{
	    $redis = Yii::$app->dataCache->getRedis();
	    $cacheKey = Yii::$app->dataCache->getCacheKey('MARKETING_ACTIVITY_PRIZE_WINNERS', $shop_id, $activity_id);
	    $value = $redis->exists($cacheKey);
	    if (!$value) {
	        //判断是否有空标记，有则说明没有人中奖，不需要查数据库直接返回false
	        if($redis->hasEmptyTag('MARKETING_ACTIVITY_PRIZE_WINNERS', [$shop_id, $activity_id])){
	            return false;
	        }
	        //如没有空标记则需要查找数据库
	        $ids = $this->db->getIDs(['shop_id' => $shop_id, 'marketing_activity_id' => $activity_id, $uid]);
	        if (count($ids)){
	            //数据库有数据则更新缓存并且移除空标记
	            $redis->executeCommand('sadd', ArrayHelper::merge([$cacheKey],$ids));
	            $redis->removeEmptyTag('MARKETING_ACTIVITY_PRIZE_WINNERS', [$shop_id, $activity_id]);
	        } else {
	            //数据库没数据则设置空标记，避免下次重复查询
	            $redis->setEmptyTag('MARKETING_ACTIVITY_PRIZE_WINNERS', [$shop_id, $activity_id]);
	            return false;
	        }
	    }
	    return Yii::$app->dataCache->getRedis()->sismember($cacheKey, $uid);
	}

### 防止超发

##### 应用场景

- 防止红包超发
- 防止抽奖奖品超发
- 防止秒杀商品超发
- 防止点赞奖品超发

##### 具体范例

已防止抽奖奖品超发为例


    private function getWinnerCount($shop_id, $activity_id, $prize_level)
    {
        return Yii::$app->dataCache->getRedis()->getCache('MARKETING_ACTIVITY_PRIZE_WINNER_COUNT', [$shop_id, $activity_id, $prize_level], function($shop_id, $activity_id, $prize_level){
            $this->marketingRecordDao->countMarketingRecord(['shop_id' => $shop_id, 'marketing_activity_id' => $activity_id, 'level' => $prize_level]);
        },[$shop_id, $activity_id, $prize_level]);
    }

具体流程

	//获取已中奖数量
	$winnerCount = $this->getWinnerCount($marketingActivity['shop_id'], $marketingActivity['id'], $level);
	//用奖品数量减去已中奖数量
	$canwinCount = $winPrize['num'] - $winnerCount;
	//判断奖品数量
	if ($canwinCount <= 0) {
	    //如果剩余数量等于0，则说明奖品已发完，此处应报错
	} else {
	    //初步检查有剩余奖品后，需要用incr增加已中奖数量，这一步是防止超发最关键的一步，利用redis的原子性，可以保证不会出现重复增加的问题
	    $winnerCount = yii::$app->dataCache->getRedis()->displayException(true)->incr(
	        ['MARKETING_ACTIVITY_PRIZE_WINNER_COUNT', $marketingActivity['shop_id'], $marketingActivity['id'], $level]
	    );
	    if ($winnerCount > $winPrize['num']) {
	        //incr以后的中奖数量如果大于奖品数量，则表明奖品已超发，此处应报错并将刚刚incr的中奖数量decr回去
			yii::$app->dataCache->getRedis()->displayException(true)->decr(
                            ['MARKETING_ACTIVITY_PRIZE_WINNER_COUNT', $marketingActivity['shop_id'], $marketingActivity['id'], $level]
                        );
	    }
	}

### 缓存热数据

##### 应用场景

- 用户最后登录信息更新（登录册数，最后登录IP）


具体流程

保存登录信息

	//用哈希保存最后登录时间和最后登录IP
	$redis->hmset(['WX_USER_LAST_LOGIN_INFO', $params['id']],
	    'lastlogintime', $params['lastlogintime'],
	    'lastloginip', $params['lastloginip']
	);
	//登录次数+1
	$redis->hincrby(['WX_USER_LAST_LOGIN_INFO', $params['id']], 'login_count', 1);
	//将登录用户的id存入有序集合，以最后登录时间为score
	$redis->zadd(['WX_USER_LAST_LOGIN_COLLECTION'], $params['lastlogintime'], $params['id']);

将登录信息同步到数据库

	//每次从Redis拉取的记录数
	cacheNumber = 1000; 
	//延迟更新秒数，假设设置90分钟，则表示只同步90分钟内未更新的冷数据
    $delayTime = 90 * 60; 
    $time = time() - $delayTime;
    $redis = Yii::$app->dataCache->getRedis();
	//按score排序后，将最早登录的用户一批一批取出来
    while ($ids = $redis->zrangebyscore(['WX_USER_LAST_LOGIN_COLLECTION'], 0, $time, 'LIMIT', 0, $cacheNumber)){
        foreach($ids as $id){
			//拿到用户ID后再获取用户具体的登录信息
            $data = $redis->hmget(['WX_USER_LAST_LOGIN_INFO', $id], 'lastloginip', 'lastlogintime', 'login_count');
            if($data[1]){
                $loginInfo = [
                    'lastloginip' => $data[0],
                    'lastlogintime' => $data[1],
                    'login_count' => new Expression('`login_count` + ' . $data[2])
                ];
				//将最后登录信息更新到数据库中
                Yii::$app->db->createCommand()->update('wx_user_infos', $loginInfo, ['id' => $id])->execute();
				//删除缓存用户信息，避免登录次数计算重复                
				$redis->del(['WX_USER_LAST_LOGIN_INFO', $id]);
            }
        }
		//处理完一批用户后，将其从有序集合中删除
        array_unshift($ids, Yii::$app->dataCache->getCacheKey('WX_USER_LAST_LOGIN_COLLECTION'));
        $redis->executeCommand('zrem', $ids);
    }