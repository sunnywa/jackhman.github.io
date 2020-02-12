# GC源码解析

## G1

### Region Size

> **说明**：JDK7和JDK8的Region划分实现略有不同（差异非常小，且只有-Xmx和-Xms的值不一样才有区别），本篇文章讲解的是JDK8中Region的划分实现；如果要了解JDK7的Region划分实现，请参考[JDK7 headpRegion.cpp](https://link.jianshu.com?t=https://github.com/dmlloyd/openjdk/blob/jdk7u/jdk7u/hotspot/src/share/vm/gc_implementation/g1/heapRegion.cpp)

#### 源码分析

G1 Region划分的实现源码在headpRegion.cpp中，摘取部分核心源码如下：

```cpp
// Minimum region size; we won't go lower than that.
// We might want to decrease this in the future, to deal with small
// heaps a bit more efficiently.
#define MIN_REGION_SIZE  (      1024 * 1024 )

// Maximum region size; we don't go higher than that. There's a good
// reason for having an upper bound. We don't want regions to get too
// large, otherwise cleanup's effectiveness would decrease as there
// will be fewer opportunities to find totally empty regions after
// marking.
#define MAX_REGION_SIZE  ( 32 * 1024 * 1024 )

// The automatic region size calculation will try to have around this
// many regions in the heap (based on the min heap size).
#define TARGET_REGION_NUMBER          2048

size_t HeapRegion::max_region_size() {
  return (size_t)MAX_REGION_SIZE;
}

// 这个方法是计算region的核心实现
void HeapRegion::setup_heap_region_size(size_t initial_heap_size, size_t max_heap_size) {
  uintx region_size = G1HeapRegionSize;
  // 是否设置了G1HeapRegionSize参数，如果没有配置，那么按照下面的方法计算；如果设置了G1HeapRegionSize就按照设置的值计算
  if (FLAG_IS_DEFAULT(G1HeapRegionSize)) {
    // average_heap_size即平均堆的大小，(初始化堆的大小即Xms+最大堆的大小即Xmx)/2
    size_t average_heap_size = (initial_heap_size + max_heap_size) / 2;
    // average_heap_size除以期望的REGION数量得到每个REGION的SIZE，与MIN_REGION_SIZE取两者中的更大值就是实际的REGION_SIZE；从这个计算公式可知，默认情况下如果JVM堆在2G（TARGET_REGION_NUMBER*MIN_REGION_SIZE）以下，那么每个REGION_SIZE都是1M；
    region_size = MAX2(average_heap_size / TARGET_REGION_NUMBER, (uintx) MIN_REGION_SIZE);
  }

  // region_size的对数值
  int region_size_log = log2_long((jlong) region_size);
  // 重新计算region_size，确保它是最大的小于或等于region_size的2的N次方的数值，例如重新计算前region_size=33，那么重新计算后region_size=32；重新计算前region_size=16，那么重新计算后region_size=16；
  // Recalculate the region size to make sure it's a power of
  // 2. This means that region_size is the largest power of 2 that's
  // <= what we've calculated so far.
  region_size = ((uintx)1 << region_size_log);

  // 确保计算出来的region_size不能比MIN_REGION_SIZE更小，也不能比MAX_REGION_SIZE更大
  // Now make sure that we don't go over or under our limits.
  if (region_size < MIN_REGION_SIZE) {
    region_size = MIN_REGION_SIZE;
  } else if (region_size > MAX_REGION_SIZE) {
    region_size = MAX_REGION_SIZE;
  }

  // 与MIN_REGION_SIZE和MAX_REGION_SIZE比较后，再次重新计算region_size
  // And recalculate the log.
  region_size_log = log2_long((jlong) region_size);

  ... ...
}
```

> 源码解读：
>  **MIN_REGION_SIZE**：允许的最小的REGION_SIZE，即1M，不可能比1M还小；
>  **MAX_REGION_SIZE**：允许的最大的REGION_SIZE，即32M，不可能比32M更大；限制最大REGION_SIZE是为了考虑GC时的清理效果；
>  **TARGET_REGION_NUMBER**：JVM对堆**期望划分**的REGION数量，而**不是实际划分**的REGION数量；

#### 计算演示

> 1、 验证下面这段源码，即`如果配置了XX:G1HeapRegionSize，那么以配置为准；否则以计算为准`:

```cpp
// JDK8的实现
if (FLAG_IS_DEFAULT(G1HeapRegionSize)) {
  size_t average_heap_size = (initial_heap_size + max_heap_size) / 2;
  region_size = MAX2(average_heap_size / TARGET_REGION_NUMBER, (uintx) MIN_REGION_SIZE);
}


// JDK7的实现
if (FLAG_IS_DEFAULT(G1HeapRegionSize)) {
  // We base the automatic calculation on the min heap size. This
  // can be problematic if the spread between min and max is quite
  // wide, imagine -Xms128m -Xmx32g. But, if we decided it based on
  // the max size, the region size might be way too large for the
  // min size. Either way, some users might have to set the region
  // size manually for some -Xms / -Xmx combos.

  region_size = MAX2(min_heap_size / TARGET_REGION_NUMBER,
                     (uintx) MIN_REGION_SIZE);
}
```

> 这是JDK7和JDK8关于REGION_SIZE计算唯一的区别，事实上当Xmx和Xms的值不一样时，JVM不太好自动计算region_size，JDK7的注释进一步的解释了，且建议某些-Xms/-Xmx组合情况下，用户自己设置REGION_SIZE

- 计算为准
   假设配置JVM参数-Xmx6144m -Xms2048m，那么计算过程如下：

1. average_heap_size=(6144m+2048m)/2=4096m
2. region_size=max(4096m/2048, 1m)=2m
3. region_size_log=21（因为2^21=2*1024*1024<=2m）
4. region_size=2^21=2m（保证region_size的值为2^n）
5. region_size=2m（因为MIN_REGION_SIZE<=2m<=MAX_REGION_SIZE）

- 配置为准
   假设配置JVM参数-Xmx1024m -Xms1024m -XX:G1HeapRegionSize=4m，那么计算过程如下：

1. region_size=4m
2. region_size_log=22（因为2^22<=4m）
3. region_size=2^22=4m
4. region_size=4m（因为MIN_REGION_SIZE<=1m<=MAX_REGION_SIZE）

> 2、 验证下面这段源码，即`region_size的值一定是2^n`:

```cpp
int region_size_log = log2_long((jlong) region_size);
region_size = ((uintx)1 << region_size_log);
```

假设配置JVM参数-Xmx3072m -Xms3072m，那么计算过程如下：

1. average_heap_size=(3072m+3072m)/2=3072m
2. region_size=max(3072m/2048, 1m)=1.5*1024*1024
3. region_size_log=20（因为2^20<1.5*1024*1024<2^21）
4. region_size=2^20=1m（保证region_size的值为2^n）
5. region_size=1m（因为MIN_REGION_SIZE<=1m<=MAX_REGION_SIZE）

> 3、 验证下面这段源码，即`region_size的值一定是在[MIN_REGION_SIZE, MAX_REGION_SIZE]`这个范围:

```cpp
if (region_size < MIN_REGION_SIZE) {
  region_size = MIN_REGION_SIZE;
} else if (region_size > MAX_REGION_SIZE) {
  region_size = MAX_REGION_SIZE;
}
```

假设配置JVM参数-Xmx1024m -Xms1024m  -XX:G1HeapRegionSize=64m，那么计算过程如下：

1. region_size=64m
2. region_size_log=26（因为2^26<=64m）
3. region_size=2^26=64m
4. region_size=32m（因为region_size必须在[MIN_REGION_SIZE, MAX_REGION_SIZE]之间）

#### REGION_SIZE总结

通过上面的分析可知G1垃圾回收时JVM分配REGION的SIZE有如下要求：
 1、如果配置了-XX:G1HeapRegionSize，那么先以配置的值为准；否则以计算为准；
 2、根据第一步计算得到的REGION_SIZE，取不能大于它的最大的2^n的值为第二步计算得到的REGION_SIZE的值
 3、把第二步计算得到的REGION_SIZE和MIN_REGION_SIZE比较，如果比MIN_REGION_SIZE还小，那么MIN_REGION_SIZE就是最终的region_size；否则再把REGION_SIZE和MAX_REGION_SIZE比较，如果比MAX_REGION_SIZE还大，那么MAX_REGION_SIZE就是最终的region_size；如果REGION_SIZE在[MIN_REGION_SIZE, MAX_REGION_SIZE]之间，那么REGIOIN_SIZE就是最终的region_size；

#### 验证方式

通过下面这段源码配置JVM参数即可验证JDK8 G1中REGION_SIZE的计算方式：

```java
import java.util.UUID;

/**
 * @author afei
 */
public class StringTest {
    public static void main(String[] args) throws Exception {
        for (int i=0; i<Integer.MAX_VALUE; i++){
            // 利用UUID不断生成字符串，这些字符串都会在堆中分配，导致不断塞满Eden区引起YoungGC
            UUID.randomUUID().toString();
            if (i>=100000 && i%100000==0){
                System.out.println("i="+i);
                Thread.sleep(3000);
            }
        }
        Thread.sleep(3000);
    }
}
```



> JVM参数：
>
> ``java -XX:+UseG1GC -verbose:gc ${HEAP_OPTS} -XX:+PrintHeapAtGC  StringTest``，其中${HEAP_OPTS}由上面计算演示过程中提供的JVM参数取代即可，例如：
>
> ``java -XX:+UseG1GC -verbose:gc -Xmx6144m -Xms2048m -XX:+PrintHeapAtGC  StringTest``，GC日志如下，从GC日志中可以看出region size为2048k;

```shell
{Heap before GC invocations=12 (full 0):
 garbage-first heap   total 2097152K, used 1255968K [0x0000000640000000, 0x0000000640202000, 0x00000007c0000000)
  region size 2048K, 614 young (1257472K), 1 survivors (2048K)
 Metaspace       used 2903K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 308K, capacity 386K, committed 512K, reserved 1048576K
[GC pause (G1 Evacuation Pause) (young) 1226M->816K(2048M), 0.0024981 secs]
Heap after GC invocations=13 (full 0):
 garbage-first heap   total 2097152K, used 816K [0x0000000640000000, 0x0000000640202000, 0x00000007c0000000)
  region size 2048K, 1 young (2048K), 1 survivors (2048K)
 Metaspace       used 2903K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 308K, capacity 386K, committed 512K, reserved 1048576K
}
```

