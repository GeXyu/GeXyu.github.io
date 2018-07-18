---
title: JVM系列（五）GC分析
comments: true
tags:
  - JVM系列（五）GC分析
categories:
  - JVM系列（五）GC分析
url: 266.html
id: 266
date: 2017-05-31 18:34:16
---

**Java堆分析**
-----------

大部分的Java程序员，我相信都见过java.lang.OutOfMemoryError这个错误，产生该错误原因大多为jvm空间太小， 程序逻辑不合理。具体可以分为一下几类：

*   对象太大，java堆太小和没有及时回收垃圾导致Java堆的OutOfMemoryError
*   字节码生成技术生成太多的字节码永久去太小和没有允许Class回收导致永久区的OutOfMemoryError
*   线程请求的栈深度大于虚拟机允许的深度或者是虚拟机栈动态扩展时，无法申请到足够的空间都会引起Java栈的OutOfMemoryError
*   当ByteBuffer.allocateDirect()无法从操作系统获得足够的空间，会导致直接内存的OutOfMemoryError

综合上面四点，发生OutOfMemoryError的地方为一下几个区域：Java堆，永久区（方法区），Java栈，直接空间。既然我们知道了发生错误的几个地方，那么就看一下什么样的程序能够产生错误又有什么方法可以解决。

### **Java堆**

public static void main(String args\[\]){
		System.out.println(Runtime.getRuntime().maxMemory()/1024/1024);
		System.out.println(Runtime.getRuntime().totalMemory()/1024/1024);
	    ArrayList<Object> list=new ArrayList<Object>();
	    for(int i=0;i<1024;i++){
	        list.add(new byte\[1024*1024\]);
	    }
	}

GC日志：

881
59
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space at cn.xiuyu.jvm.oom.Demo1.main(Demo1.java:15)

从程序上看 我们需要分配1024个1M的数据，但是我这里虚拟机默认的配置堆的最大为881M，并不能分配1024个空间，所以导致Java堆的OutOfMemoryError。

**解决方案：增大Java堆空间，及时释放内存。**

### **永久区**

//工厂类
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
//测试类
public static void main(String\[\] args) {
		ObjectFactory factory = new ObjectFactory(new Demo2());
		for(int i=0; i<1000000000;i++){
			factory.getObject();
			
		}
	}

GC日志：

\[GC \[PSYoungGen: 0K->0K(51072K)\] 102K->102K(728320K), 0.0000858 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[Full GC \[PSYoungGen: 0K->0K(51072K)\] \[PSOldGen: 102K->102K(677248K)\] 102K->102K(728320K) \[PSPermGen: 2047K->2047K(2048K)\], 0.0031717 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[GC \[PSYoungGen: 0K->0K(52416K)\] 102K->102K(729664K), 0.0002032 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
\[Full GC \[PSYoungGen: 0K->0K(52416K)\] \[PSOldGen: 102K->102K(677248K)\] 102K->102K(729664K) \[PSPermGen: 2047K->2047K(2048K)\], 0.0039734 secs\] \[Times: user=0.00 sys=0.00, real=0.00 secs\] 
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

当我们使用CGLIB，ASM之类的字节码生成类库时，会产生大量的字节码。这些字节码加载到永久区，如果永久区太小，会撑爆永久区，最终导致永久区的OutOfMemoryError。

**解决方法：增大Perm区 允许Class回收**

### **java****栈**

栈溢出指，在创建线程的时候，需要为线程分配栈空间，这个栈空间是向操作系统请求的，如果操作系统无法给出足够的空间，就会抛出OutOfMemoryError。

![TIM图片20170601102641](http://www.zzcode.cn/wp-content/uploads/2017/06/TIM图片20170601102641.png)

//-Xmx1g -Xss1m
public class Demo3 implements Runnable{
	@Override
	public void run() {
		try {
            Thread.sleep(10000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
	}
	public static void main(String\[\] args) {
		for(int i=0;i<100000;i++){
	        new Thread(new Demo3(),"Thread"+i).start();
	        System.out.println("Thread"+i+" created");
	    }
	}
}

GC日志：

Exception in thread "main" java.lang.OutOfMemoryError: 
unable to create new native thread

由于我们给堆分配了太大的空间，当我们创建一直创建新的线程，导致虚拟机需要向操作系统请求空间，当操作系统无法分配足够的空间时，就抛出了OutOfMemoryError。

**解决方法：减少堆内存，减少线程栈大小，增大栈空间**

### **直接内存**

ByteBuffer.allocateDirect()无法从操作系统获得足够的空间

![TIM图片20170601102641](http://www.zzcode.cn/wp-content/uploads/2017/06/TIM图片20170601102641-1.png)

//-Xmx1g -XX:+PrintGCDetails
for(int i=0;i<1024000;i++){
		    ByteBuffer.allocateDirect(1024*1024);
		    System.out.println(i);
		      System.gc();
		}

GC日志：

![TIM图片20170601103144](http://www.zzcode.cn/wp-content/uploads/2017/06/TIM图片20170601103144.png)

![TIM图片20170601103153](http://www.zzcode.cn/wp-content/uploads/2017/06/TIM图片20170601103153.png)

从GC日志可以看到 堆空间的使用很少，没有触发GC，而直接内存需要GC回收，但是直接内存无法引起GC。直接内存使用满时，无法触发GC。如果堆空间很富余，无法触发GC，直接内存可能就会溢出。如果堆空间触发GC，直接内存可以回收。

**解决方案：减少堆内存，有意触发GC**

### **GC****性能监控工具**