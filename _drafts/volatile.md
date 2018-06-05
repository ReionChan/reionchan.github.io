http://www.cnblogs.com/trytocatch/archive/2013/03/21/2974547.html
http://www.infoq.com/cn/articles/java-memory-model-4
http://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html#volatile
https://www.ibm.com/developerworks/cn/java/j-jtp06197.html

你这段关于volatile特性的描述的原始出处来自于：
《The JavaTM Virtual Machine Specification Second Edition》
在线版本在：docs.oracle.com/javase/specs/jvms/se5.0/html/Th...
“8.7 Rules for volatile Variables”这一节的最后一句话，就是原始出处。
请结合上下文（8.7节，最好是整个第8章），来理解《JVM规范第二版》对volatile在java旧内存模型中所具有特性的描述。

Brian Goetz在《JSR 133 (Java Memory Model) FAQ》中，对volatile做了简洁，清晰的描述：
www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-f...

另外，Brian Goetz写过一篇很棒的文章。
这篇文章说明了应该在什么情况下，怎么样使用volatile才是正确的：
www.ibm.com/developerworks/cn/java/j-jtp06197.html
**************************************************************************
第二个问题
对变量的同时更新的问题，要看你是指某个时间点，还是某个时间段。

我们先考虑时间点。
在任意时间点，不可能有两个处理器能同时访问内存。
因为处理器总线会同步多个处理器对内存的并发访问（请参阅我在这个系列的第三篇中，对处理器总线工作机制的描述）。
因此在任意时间点，不可能有两个线程能同时更新X的值。

再考虑时间段。
在某个时间段，两个线程有可能会同时去读/写同一个共享变量。
比如有一个long型的共享变量l被A，B线程并发访问，同时程序没有做任何同步（指广义上的同步）。
在一些32位的处理器中，程序有可能按下列时序执行：
时间点1：首先，A写l的高32位；
时间点2：然后，B写l的高32位；
时间点3：接着，B写l的低32位；
时间点4：最后，A写l的低32位。
在1到4这个时间段，A，B线程在同时在写变量l；但在1-4中的任意一个时间点，只有一个线程能访问变量l。
以这个时序来执行后，l将变成一个“四不像”（高32位是B线程写的，低32位是A线程写的）。
要避免出现这种问题，只需要对这个多线程程序做正确同步（指广义上的同步）即可。
对于正确同步的多线程程序，java内存模型确保不会出现上述问题。


http://www.cnblogs.com/trytocatch/articles/2853742.html