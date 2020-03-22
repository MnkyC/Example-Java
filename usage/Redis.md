# 锁
## SETNX+EXPIRE 错误做法
SETNX, Key不存在时才能将key设置为value，存在就不做处理

EXPIRE, 给key设置过期时间，自动删除

因为需要两个操作，不具有原子性，锁会无法释放
## SET, Redis 2.6.12开始增加了一系列选项
SET key uuid NX PX 3000，等同于SETNX+EXPIRE，但是保证了原子性
## 用Lua脚本保证原子性
有个非常重要的点，需要保证value的唯一性，可以设置唯一的客户端ID或者随机数UUID，当释放锁的时候先要检查value是否是当前进程的锁，不能把别人的锁释放了

	-- 加锁
	if  redis.call('SETNX', KEYS[1], ARGV[1]) == 1 then
		redis.call('EXPIRE', KEYS[1], ARGV[2])
		return 1
	else
		return 0
	end
	-- 释放锁
	if redis.call('GET', KEYS[1]) == ARGV[1] then
		return redis.call('DEL', KEYS[1])
	else
		return 0
	end
## Redlock和Redission 分布式
### Redlock算法
内部实现也是基于lua脚本，用其保证原子性

假设n个redis节点，相互之间安全独立，不存在主从复制或集群协调，向这些节点请求获取锁

客户端操作流程：

1. 获取当前Unix时间戳，毫秒为单位
2. n个实例依次使用相同的key和唯一的value（如uuid）获取锁。获取锁时，客户端要设置一个网络连接和响应的超时时间（需要小于锁的失效时间，假设锁失效时间10s，超时时间就应该在5-50毫秒之间，避免服务节点已经挂掉了，客户端还在等待，没有响应了就立刻进行下一个实例尝试）
3. 客户端用当前时间减去开始获取锁的时间就是获取锁使用了的时间，**当且仅当大多数(n/2+1)节点都获取到了锁，且使用的总时间小于锁失效的时间，锁才算获取成功**
4. 若获取到了锁，**key的真正有效时间等于有效时间减去获取锁所使用的时间**
5. **若获取锁失败，则需要对所有节点进行解锁操作，即使有些节点没有获取锁成功**，防止这些节点后面不能获取锁

### Redission
Jedis是Redis的Java客户端，阻塞式I/O，方法调用都是同步的，不支持异步，并且不是线程安全的，需要通过连接池使用

Redission也是Redis的Java客户端，底层基于Netty，实现了非阻塞I/O，方法调用是异步的，线程安全，使用Redlock算法
### 唯一ID
**分布式锁非常重要的一点就是保证set的value要有唯一性**
Redission用UUID+threadId来保证唯一性




