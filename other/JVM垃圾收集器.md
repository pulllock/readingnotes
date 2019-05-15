# 垃圾收集器

- Serial收集器
- ParNew收集器
- Parallel Scavenge收集器
- Serial Old收集器
- Parallel Old收集器
- CMS收集器
- G1收集器

## Serial收集器

Serial收集器是单线程收集器，进行垃圾收集时必须暂停其他所有的工作线程，直到收集结束。是Client模式下的默认新生代收集器。

优势：简单高效，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程效率。

## ParNew收集器

ParNew收集器其实是Serial收集器的多线程版本，Server模式下的首选的新生代收集器。ParNew收集器默认开启收集线程数与CPU数量相同。

## Parallel Scavenge收集器

Parallel Scavenge收集器是一个新生代收集器，使用复制算法，是并行的多线程收集器，目标是达到一个可控制的吞吐量。

## Serial Old收集器

Serial Old收集器是Serial收集器的老年代版本，是一个单线程收集器，使用标记-整理算法。主要给Client模式下使用。如果用在Server模式下，有两种用途：1. 与Parallel Scavenge收集器搭配；2. 作为CMS的后备方案，在并发收集发生Concurrent Mode Failure时使用。

## Parallel Old收集器

Parallel Old收集器是Parallel Scavenge收集器的老年代版本，使用多线程和标记-整理算法。在注重吞吐量和CPU敏感的场合，优先考虑Parallel Scavenge + Parallel Old组合。

## CMS收集器

CMS收集器是真正意义上的并发收集器，实现了让垃圾收集线程和用户线程同时工作。尽可能的缩短垃圾收集时用户线程的停顿时间。CMS收集器是基于标记-清除算法实现的，总共分四个步骤：

- 初始标记
- 并发标记
- 重新标记
- 并发清除

初始标记和重新标记需要Stop The World，初始标记仅仅是标记一下GC Roots能关联到的对象，速度很快。并发标记就是进行GC Roots Tracing的过程。重新标记是为了修正并发标记期间因用户程序继续运行而导致的变动的对象，这个阶段停顿时间一般会比初始标记阶段稍长，但远比并发标记时间段。

整个过程耗时最长的是并发标记和并发清除，但这两个过程可以合用户线程一起工作。

缺点：

- CMS收集器对CPU非常敏感，并发阶段会占用一部分线程，导致应用变慢，吞吐量降低。
- CMS无法处理浮动垃圾，可能出现Concurrent Mode Failure失败而导致另一次Full GC产生。
- CMS是基于标记-清除算法实现的，会有大量内存空间碎片问题。

## G1收集器

G1是面向服务端应用的垃圾收集器。
