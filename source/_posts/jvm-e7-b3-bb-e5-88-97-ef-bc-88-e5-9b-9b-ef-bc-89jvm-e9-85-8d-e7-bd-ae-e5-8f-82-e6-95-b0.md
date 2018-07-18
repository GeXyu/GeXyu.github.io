---
title: JVM系列（四）JVM配置参数
comments: true
tags:
  - JVM系列（四）JVM配置参数
categories:
  - JVM系列（四）JVM配置参数
url: 256.html
id: 256
date: 2017-05-31 11:21:23
---

不管是Minor GC还是Full GC，在GC过程中会导致程序在运行中中断，正确的选择不同的GC策略，调整JVM、GC的参数，可以极大的减少由于GC工作，而导致的程序运行中断方面的问题，进而适当的提高Java程序的工作效率。

新生代GC（Minor GC）：发生在新生代的垃圾收集动作，因为Java对象大多具备朝生夕灭的特点，所以Minor GC非常频繁，回收速度也很快。

老年代GC（Major GC/Full GC）：指发生在老年代的GC动作，出现了Full GC，一般会伴随至少一次Minor GC（但非绝对）。

**GC日志**  

-----------

阅读GC日志是理解Java虚拟机内存问题的基础技能，它只是一些人为的规则。在程序运行中可以通过配置**-XX:+PrintGCDetails**来打印日志输出。

![1](http://www.zzcode.cn/wp-content/uploads/2017/05/1-4.png)

![20170531094844](http://www.zzcode.cn/wp-content/uploads/2017/05/20170531094844.png)

然后再控制台会打印出GC 信息

![_20170531094742](http://www.zzcode.cn/wp-content/uploads/2017/05/20170531094742.png)

首先 GC日志的开头 “GC”或者是“Full GC”说明这次垃圾收集的停顿类型，而不是区分为新生代还是老年代的垃圾收集动作，如果有Full GC说明这次GC过程中发生了Stop The World，从图上看显然这次GC中没有发生Stop The World。接下来PSYoungGen 表示使用的垃圾收集器为Parallel Scavenge收集器。

后面方括号5520K->1280K(8192K)表示“GC前该区域已使用的容量 -> GC后该内存区域已使用容量（改内存的总容量）”，而方括号外面的5520K->1280K(18432K) 表示"GC前Java堆已使用的容量->GC后Java堆已使用的容量”

再往后 0.0027784 secs 表示该内存区域GC所占用的时间，单位是时间。而\[Times: user=0.00 sys=0.00, real=0.00 secs\] 这里面对user,sys,real 与Linuxd time命令所输出的时间含义一样，分别表示用户态消耗的CPU时间，内核态消耗的CPU事件和操作开始到结束所经过的撞钟时间（Wall Clock Time）。

Heap以后就是GC堆的详细信息

在PSYoungGen可以看到新生代的Eden，Form， To区域。这里“ PSYoungGen      total 8192K used 6588K”其中total表示**新生代**使用的区域总和=Eden + 一个Survivor区， used  6588K表示使用了6588K。“ \[0x000000000b090000, 0x000000000ba90000, 0x000000000ba90000)”分别表示地址的低边界，当前边界，最高边界。可以通过（最高边界-最低边界）/1024/1024得到区域大小，以M为单位。

PSOldGen表示老年代后面的参数代表和PSYoungGen 的相同。PSPermGen表示方法区，参数和上面两者相同，但是该区域一般不发生GC 但是不代表不发生GC。

Java代码和jvm命令（前面几个参数后面会说到）

//-Xmx20M -Xms20M -Xmn7M -XX:+PrintGCDetails
public class Demo4 {
	public static void main(String\[\] args) {
		byte\[\] b=null;
		for(int i=0;i<10;i++){
		      b=new byte\[1\*1024\*1024\];
		}
	}
}

其他的日志打印命令

*   -XX:+printGC（打印GC的简要信息）
    
*   -XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式，需要与打印GC信息配合使用）
    
*   -XX:+PrintGCDateStamps  输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800 需要与打印GC信息配合使用）
    
*   -XX:+PrintHeapAtGC   在进行GC的前后打印出堆的信息 
    
*   -Xloggc:E:/logs/gc.log  指定位置输出 （E:/logs的gc.log文件）
    
*   -XX:+TraceClassLoading  打印类加载的信息
    
*   -XX:+PrintClassHistogram  按下Ctrl+Break后，打印类的信息（分别显示：序号、实例数量、总大小、类型）
    

**堆分配命令**  

------------

### **-Xmx –Xms 指定最大堆 最小堆**  

//-Xmx20m -Xms20m -XX:+PrintGCDetails
public class Demo1 {
	public static void main(String\[\] args) {
		//运行时的最大内存（byte为单位）
		System.out.println(Runtime.getRuntime().maxMemory() /1024.0 /1024 + "M");	
		System.out.println(Runtime.getRuntime().freeMemory() /1024.0/1024 + "M");	
		System.out.println(Runtime.getRuntime().totalMemory() /1024.0 /1024 + "M");
		
		
		byte\[\] b=new byte\[1\*1024\*1024\];
		System.out.println("---分配了1M空间给数组----");
		System.out.println(Runtime.getRuntime().maxMemory() /1024.0 /1024 + "M");	
		System.out.println(Runtime.getRuntime().freeMemory() /1024.0/1024 + "M");	
		System.out.println(Runtime.getRuntime().totalMemory() /1024.0 /1024 + "M");
		
		byte\[\] b4 =new byte\[4\*1024\*1024\];
		System.out.println("----分配了4M空间给数组---");
		System.out.println(Runtime.getRuntime().maxMemory() /1024.0 /1024 + "M");	
		System.out.println(Runtime.getRuntime().freeMemory() /1024.0/1024 + "M");	
		System.out.println(Runtime.getRuntime().totalMemory() /1024.0 /1024 + "M");
		
		
		System.gc();
		System.out.println("------GC后-----");
		System.out.println(Runtime.getRuntime().maxMemory() /1024.0 /1024 + "M");	
		System.out.println(Runtime.getRuntime().freeMemory() /1024.0/1024 + "M");	
		System.out.println(Runtime.getRuntime().totalMemory() /1024.0 /1024 + "M");
	}
}

运行的结果

17.8125M
5.6421966552734375M
5.9375M
---分配了1M空间给数组----
17.8125M
4.642173767089844M
5.9375M
----分配了4M空间给数组---
17.8125M
4.70465087890625M
10.0M
------GC后-----
17.8125M
4.741142272949219M
10.0M

我们结合代码看一下，首先我们设置了Java堆最大为20M 和 最小堆（初始值）为5M，我们通过runtime接口得到运行时的数据分别是最大空间，空闲空间和总空间。我们第一次打印分别是17M 5M和5.9M。

当我们给分配1M空间时，我们的总空间可以容纳1M空间，所以第二次打印17M，4M，5.9M，可以看出空闲空间变小了因为分配了1M空间，所以说Java分配空间时Java会尽可能维持在最小堆。

当我们再分配4M空间时，我们的空闲空间分配不了4M空间，总空间会扩容。所以分配4M空间后Java堆的情况分别是：17M，4M，10M。

当我们GC一次后，可以看到空闲空间变大了

### **Xmn指定新生代大小**  

这次我们把Java堆的最大设置为20M，初始值也为20M 把新生代大小设置为1M，执行下面代码看一下GC日志。

//-Xmx20m -Xms20m -Xmn1m -XX:+PrintGCDetails
public static void main(String\[\] args) {
		byte\[\] b=null;
		for(int i=0;i<10;i++){
		      b=new byte\[1\*1024\*1024\];
		}
	}

打印日志

Heap
 PSYoungGen      total 896K, used 305K \[0x000000000b920000, 0x000000000ba20000, 0x000000000ba20000)
  eden space 768K, 39% used \[0x000000000b920000,0x000000000b96c560,0x000000000b9e0000)
  from space 128K, 0% used \[0x000000000ba00000,0x000000000ba00000,0x000000000ba20000)
  to   space 128K, 0% used \[0x000000000b9e0000,0x000000000b9e0000,0x000000000ba00000)
 PSOldGen        total 19456K, used 10240K \[0x000000000a620000, 0x000000000b920000, 0x000000000b920000)
  object space 19456K, 52% used \[0x000000000a620000,0x000000000b0200f0,0x000000000b920000)
 PSPermGen       total 21248K, used 2972K \[0x0000000005220000, 0x00000000066e0000, 0x000000000a620000)
  object space 21248K, 13% used \[0x0000000005220000,0x00000000055072e0,0x00000000066e0000)

可以从GC日志中看到因为新生代太小不能够分配1M空间： 1，没有发生GC    2，对象直接分配到老年代

**如果我们把-Xmn（新生代大小）设置的大一些,执行下面的代码（同样的代码）**

//-Xmx20M -Xms20M -Xmn15m -XX:+PrintGCDetails
public static void main(String\[\] args) {
		byte\[\] b=null;
		for(int i=0;i<10;i++){
		      b=new byte\[1\*1024\*1024\];
		}
	}

GC日志：

Heap
 PSYoungGen      total 13440K, used 10931K \[0x000000000ad20000, 0x000000000bc20000, 0x000000000bc20000)
  eden space 11520K, 94% used \[0x000000000ad20000,0x000000000b7ccf38,0x000000000b860000)
  from space 1920K, 0% used \[0x000000000ba40000,0x000000000ba40000,0x000000000bc20000)
  to   space 1920K, 0% used \[0x000000000b860000,0x000000000b860000,0x000000000ba40000)
 PSOldGen        total 5120K, used 0K \[0x000000000a620000, 0x000000000ab20000, 0x000000000ad20000)
  object space 5120K, 0% used \[0x000000000a620000,0x000000000a620000,0x000000000ab20000)
 PSPermGen       total 21248K, used 2972K \[0x0000000005220000, 0x00000000066e0000, 0x000000000a620000)
  object space 21248K, 13% used \[0x0000000005220000,0x00000000055072e0,0x00000000066e0000)

新生代可空间比较大可以容纳10M的数组所以没有发生GC，全部分配在eden而且老年代没有使用。

**如果我们把新生代大小设置为7M执行上面的代码**

 -Xmx20M -Xms20M -Xmn7M -XX:+PrintGCDetails

GC日志：

\[GC \[PSYoungGen: 4418K->256K(6272K)\] 4418K->1280K(19584K), 0.0079350 secs\] \[Times: user=0.00 sys=0.00, real=0.01 secs\] 
\[GC \[PSYoungGen: 5433K->288K(6272K)\] 6457K->2336K(19584K), 0.0026655 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
Heap
 PSYoungGen      total 6272K, used 1419K \[0x000000000b380000, 0x000000000ba80000, 0x000000000ba80000)
  eden space 5376K, 21% used \[0x000000000b380000,0x000000000b49aed0,0x000000000b8c0000)
  from space 896K, 32% used \[0x000000000b9a0000,0x000000000b9e8000,0x000000000ba80000)
  to   space 896K, 0% used \[0x000000000b8c0000,0x000000000b8c0000,0x000000000b9a0000)
 PSOldGen        total 13312K, used 2048K \[0x000000000a680000, 0x000000000b380000, 0x000000000b380000)
  object space 13312K, 15% used \[0x000000000a680000,0x000000000a880030,0x000000000b380000)
 PSPermGen       total 21248K, used 2981K \[0x0000000005280000, 0x0000000006740000, 0x000000000a680000)
  object space 21248K, 14% used \[0x0000000005280000,0x00000000055697e8,0x0000000006740000)

可以看出发生了2次新生代GC，因为Survivor区太小，所以部分对象需要老年代担保，所以老年代使用了15%。

### **-XX:SurvivorRatio**  

**-XX:SurvivorRatio**命令是设置两个Survivor区和eden的比，譬如：-XX:SurvivorRatio8表示 两个Survivor :eden=2:8，即一个Survivor占年轻代的1/10。

我们在原来的jvm命令基础再添加-XX:SurvivorRatio=2 参数，表示Survivor与Eden比例为2：2即两个区域一样大。

-Xmx20M -Xms20M -Xmn7M -XX:+PrintGCDetails -XX:SurvivorRatio=2 

GC日志

\[GC \[PSYoungGen: 3423K->1280K(5376K)\] 3423K->1280K(18688K), 0.0069522 secs\] \[Times: user=0.00 sys=0.00, real=0.01 secs\] 
\[GC \[PSYoungGen: 4390K->1280K(5376K)\] 4390K->1280K(18688K), 0.0018699 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[GC \[PSYoungGen: 4377K->1312K(5376K)\] 4377K->1312K(18688K), 0.0007972 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
Heap
 PSYoungGen      total 5376K, used 2407K \[0x000000000b270000, 0x000000000b970000, 0x000000000b970000)
  eden space 3584K, 30% used \[0x000000000b270000,0x000000000b381f78,0x000000000b5f0000)
  from space 1792K, 73% used \[0x000000000b5f0000,0x000000000b738018,0x000000000b7b0000)
  to   space 1792K, 0% used \[0x000000000b7b0000,0x000000000b7b0000,0x000000000b970000)
 PSOldGen        total 13312K, used 0K \[0x000000000a570000, 0x000000000b270000, 0x000000000b270000)
  object space 13312K, 0% used \[0x000000000a570000,0x000000000a570000,0x000000000b270000)
 PSPermGen       total 21248K, used 2981K \[0x0000000005170000, 0x0000000006630000, 0x000000000a570000)
  object space 21248K, 14% used \[0x0000000005170000,0x00000000054597e8,0x0000000006630000)

可以从GC日志中看到进行了3次新生代的GC ，from区和to区增大，from+to = eden

### **-XX:NewRatio=1**  

**-XX:NewRatio**命令是设置新生代（eden+2*s）和老年代（不包含永久区）的比值，譬如-XX:NewRatio**=**4 表示 新生代:老年代=1:4，即年轻代占堆的1/5

我们再次添加一个参数 **-XX:NewRatio=1**表示新生代与老年代的比例为1：1

 -Xmx20M -Xms20M -XX:NewRatio=1 -XX:SurvivorRatio=2  -XX:+PrintGCDetails  

GC日志：

\[GC \[PSYoungGen: 4419K->1280K(7680K)\] 4419K->1280K(17920K), 0.0008120 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[GC \[PSYoungGen: 5430K->1280K(7680K)\] 5430K->1280K(17920K), 0.0011638 secs\] \[Times: user=0.00 sys=0.02, real=0.00 secs\] 
Heap
 PSYoungGen      total 7680K, used 3466K \[0x000000000af40000, 0x000000000b940000, 0x000000000b940000)
  eden space 5120K, 42% used \[0x000000000af40000,0x000000000b162ab0,0x000000000b440000)
  from space 2560K, 50% used \[0x000000000b6c0000,0x000000000b800018,0x000000000b940000)
  to   space 2560K, 0% used \[0x000000000b440000,0x000000000b440000,0x000000000b6c0000)
 PSOldGen        total 10240K, used 0K \[0x000000000a540000, 0x000000000af40000, 0x000000000af40000)
  object space 10240K, 0% used \[0x000000000a540000,0x000000000a540000,0x000000000af40000)
 PSPermGen       total 21248K, used 2981K \[0x0000000005140000, 0x0000000006600000, 0x000000000a540000)
  object space 21248K, 14% used \[0x0000000005140000,0x00000000054297e8,0x0000000006600000)

进行了两次新生代的GC。比例分配，新生代 老年代对半开，对象全部留在新生代。

**如果我们在此基础上减少Survivor区的大小，即把-XX:SurvivorRatio的值增大**

-Xmx20M -Xms20M -XX:NewRatio=1 -XX:SurvivorRatio=3  -XX:+PrintGCDetails

GC日志：

\[GC \[PSYoungGen: 5520K->1280K(8192K)\] 5520K->1280K(18432K), 0.0008124 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
Heap
 PSYoungGen      total 8192K, used 6588K \[0x000000000afc0000, 0x000000000b9c0000, 0x000000000b9c0000)
  eden space 6144K, 86% used \[0x000000000afc0000,0x000000000b4ef2c8,0x000000000b5c0000)
  from space 2048K, 62% used \[0x000000000b5c0000,0x000000000b700048,0x000000000b7c0000)
  to   space 2048K, 0% used \[0x000000000b7c0000,0x000000000b7c0000,0x000000000b9c0000)
 PSOldGen        total 10240K, used 0K \[0x000000000a5c0000, 0x000000000afc0000, 0x000000000afc0000)
  object space 10240K, 0% used \[0x000000000a5c0000,0x000000000a5c0000,0x000000000afc0000)
 PSPermGen       total 21248K, used 2981K \[0x00000000051c0000, 0x0000000006680000, 0x000000000a5c0000)
  object space 21248K, 14% used \[0x00000000051c0000,0x00000000054a97e8,0x0000000006680000)

进行了1次GC，老年代未使用，新空间使用率更高

### **-XX:+HeapDumpOnOutOfMemoryError**  

-XX:+HeapDumpOnOutOfMemoryError发生OOM时导出堆到文件，可以与**-XX:+HeapDumpPath**（导出路径）配合使用。

我们可以设置下面jvm的命令，并执行下面的代码

//-Xmx20m -Xms5m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:/a.dump
public static void main(String\[\] args) {
		List list = new ArrayList<Object>();
		for(int i=0;i<25; i++){
			list.add(new byte\[1\*1024\*1024\]);
		}
	}

控制台输出

java.lang.OutOfMemoryError: Java heap space
Dumping heap to E:/a.dump ...
Heap dump file created \[19910369 bytes in 0.030 secs\]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at cn.xiuyu.jvm.head.Demo8.main(Demo8.java:14)

可以用Java自带的工具VisualVM来看一下jump文件

![20170531150621](http://www.zzcode.cn/wp-content/uploads/2017/05/20170531150621.png)

当然，你也可以再添加一个参数**-XX:OnOutOfMemoryError=D:/rintstack.bat %p**，当发生OOM的时候执行一个脚本，重启程序或者是发送邮件都是可以的。

### **堆总结**  

根据实际事情调整新生代和幸存代的大小，官方推荐新生代占堆的3/8，幸存代占新生代的1/10，在OOM时，记得Dump出堆，确保可以排查现场问题。

**永久区分配参数**  

--------------

**-XX:PermSize **表示永久区初始内存分配大小

**-XX:MaxPermSize **表示对永久区分配的内存的最大上限。

在使用CGLIB等一些类库的时候（CGLIB就是“字节码生成”，javac可以说是这些类库的老祖宗。），会产生大量的字节码，这些字节码可能会撑爆永久区造成OutOfMemoryError  

可以设置XX:PermSize和-XX:MaxPermSize参数试永久区撑爆，同时还能看到永久区的GC。

首先需要写一个产生对象的工厂对象

public class ObjectFactory implements MethodInterceptor{

	private Object target;
	public ObjectFactory(Object target){
		this.target = target;
	}
	public Object getObject(){
		Enhancer en = new Enhancer();
		en.setSuperclass(target.getClass());
		en.setCallback(this);
		return en.create();
	}
	
	@Override
	public Object intercept(Object proxy, Method method, Object\[\] args,
			MethodProxy arg3) throws Throwable {	
		return method.invoke(proxy, args);
	}

}

然后写一个main方法，运行Main前，设置JVM参数

//-XX:+PrintGCDetails -XX:PermSize=1M -XX:MaxPermSize=1M
public class Demo9 {
	public static void main(String\[\] args) {
		ObjectFactory factory = new ObjectFactory(new Demo9());
		for(int i=0; i<1000000000;i++){
			factory.getObject();
			
		}
	}
}

GC日志：

\[GC \[PSYoungGen: 0K->0K(39296K)\] 103K->103K(716544K), 0.0002266 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[Full GC \[PSYoungGen: 0K->0K(39296K)\] \[PSOldGen: 103K->103K(677248K)\] 103K->103K(716544K) \[PSPermGen: 2046K->2046K(2048K)\], 0.0025604 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[GC \[PSYoungGen: 0K->0K(39296K)\] 103K->103K(716544K), 0.0000936 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[Full GC \[PSYoungGen: 0K->0K(39296K)\] \[PSOldGen: 103K->103K(677248K)\] 103K->103K(716544K) \[PSPermGen: 2047K->2047K(2048K)\], 0.0027156 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] [https://www.viagrapascherfr.com/generique-viagra-tarif/](https://www.viagrapascherfr.com/generique-viagra-tarif/) 
\[GC \[PSYoungGen: 0K->0K(41728K)\] 103K->103K(718976K), 0.0002800 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[Full GC \[PSYoungGen: 0K->0K(41728K)\] \[PSOldGen: 103K->102K(677248K)\] 103K->102K(718976K) \[PSPermGen: 2047K->2047K(2048K)\], 0.0044607 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
Error occurred during initialization of VM
java.lang.OutOfMemoryError: PermGen space
	at java.lang.StringCoding.encode(StringCoding.java:266)
	at java.lang.String.getBytes(String.java:947)
	at java.lang.ClassLoader$NativeLibrary.load(Native Method)
	at java.lang.ClassLoader.loadLibrary0(ClassLoader.java:1778)
	at java.lang.ClassLoader.loadLibrary(ClassLoader.java:1695)
	at java.lang.Runtime.loadLibrary0(Runtime.java:823)
	at java.lang.System.loadLibrary(System.java:1030)
	at java.lang.System.initializeSystemClass(System.java:1077)

从日志中可以看到由于产生大量的类，撑爆了永久区。同时还可以看到永久区的回收日志，说明永久去是回收的。

**栈大小分配**  

------------

**-Xss** 设置栈帧的深度，以K为单位

可以通过减小栈帧深度，来人为的造成StackOverflowError。

设置**-Xss128K -XX:+PrintGCDetails**参数来运行下面的代码

public class Demo10 {
	private static int count = 0;
	public static void recursion(long a,long b,long c){
		long e=1,f=2,g=3,h=4,i=5,k=6,q=7,x=8,y=9,z=10;
		count++;
		recursion(a,b,c);
	}
	public static void main(String\[\] args) {
		try{
			recursion(0L,0L,0L);
		}catch (StackOverflowError e) {
			System.out.println(count);
			e.printStackTrace();
		}
		
	}
}

GC日志：

307
java.lang.StackOverflowError
	at cn.xiuyu.jvm.head.Demo10.recursion(Demo10.java:9)

可以看到，递归调用了307次就抛出了StackOverflowError，**如果把栈帧深度设置为256K，那么**

762
java.lang.StackOverflowError
	at cn.xiuyu.jvm.head.Demo10.recursion(Demo10.java:9)