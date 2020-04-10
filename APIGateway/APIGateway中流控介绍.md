流控或者叫限流，可以通过控制流量来保护我们的系统不被大流量或者异常流量冲垮，常用的限流算法有：计数器算法、令牌桶算法、漏桶算法。

# 计数器算法

计数器算法最简单，可以实现在指定的时间段内流量不能超过多少，比如同一个ip在1秒内请求次数不能超过100次这种情形。

需要使用两个map，一个用来记录同一个ip访问的次数，一个用来记录同一个ip上次访问的时间戳。防止map无限制增长，可以单独开启一个线程，用来定时清除超过时间窗口的ip数据。

计数器算法可能会产生突刺，请求集中到达处理后，后面时间就会空闲掉。

示例代码如下：

```java
public class IPCounter {

    /**
     * 保存ip访问的次数
     * key：ip
     * value：访问次数
     */
    private Map<String, AtomicInteger> counterMap = new ConcurrentHashMap<>();

    /**
     * 保存ip访问的时间
     * key：ip
     * value：时间戳
     */
    private Map<String, Long> timeMap = new ConcurrentHashMap<>();

    /**
     * 指定的次数
     */
    private int countRule;

    /**
     * 指定的时间，毫秒
     */
    private long timeRule;

    public IPCounter(int countRule, long timeRule) {
        this.countRule = countRule;
        this.timeRule = timeRule * 1000;
    }

    public boolean allow(String ip) {
        Long time = timeMap.get(ip);
        Long now = System.currentTimeMillis();

        // 不存在或者上一个时间窗口已经过去，重置时间和计数器
        if (time == null || (now - time) > timeRule) {
            timeMap.put(ip, now);
            counterMap.put(ip, new AtomicInteger());
        }

        AtomicInteger count = counterMap.get(ip);
        int temp = 1;
        if (count != null) {
            temp = count.incrementAndGet();
        }

        return temp <= countRule;
    }

    public static void main(String[] args) {
        // 10秒不能超过5次
        IPCounter counter = new IPCounter(5, 10);
        String ip = "192.168.1.1";
        System.out.println(counter.allow(ip));
        System.out.println(counter.allow(ip));
        System.out.println(counter.allow(ip));
        System.out.println(counter.allow(ip));
        System.out.println(counter.allow(ip));
        System.out.println(counter.allow(ip));
    }
}
```

# 漏桶算法

漏桶算法，漏桶的容量是固定的，大批流量进来，超过漏桶数量的抛弃掉，进入到漏桶的请求可以匀速流出。

漏桶算法能够限制请求的速率。

# 令牌桶算法

令牌桶算法是以固定的速度往桶里产生令牌，桶满了新的令牌被丢弃或者拒绝，请求到达的时候会先从桶里获取令牌，再继续执行。

令牌桶算法可以限制请求调用速率，也允许一定程度的突发调用。

可以使用guava包中的令牌桶算法限流器。