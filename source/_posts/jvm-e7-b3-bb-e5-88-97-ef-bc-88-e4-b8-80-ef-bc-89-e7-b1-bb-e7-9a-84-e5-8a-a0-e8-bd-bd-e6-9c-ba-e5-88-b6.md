---
title: JVM系列（一）类的加载机制
comments: true
tags:
  - JVM系列（一）类的加载机制
categories:
  - JVM系列（一）类的加载机制
url: 203.html
id: 203
date: 2017-05-12 15:19:00
---

虚拟机把编译后的class文件加载到内存中，并对class文件进行校验，转换解析和初始化。最终执行可以直接被虚拟机使用的Java类型，这就是虚拟机的加载机制。在java语言中，类型的加载链接和初始化都是在运行时期完成，这种策略会稍微增加一些性能的开销，但是为Java程序提供了高度灵活性。JSP的热部署和OSGI技术都是使用了运行时期的类加载。

**类的生命周期**
----------

![1345979728_9052](http://www.zzcode.cn/wp-content/uploads/2017/05/1345979728_9052.jpg)

类加载的过程包括了**加载、验证、准备、解析、初始化**五个阶段。在这五个阶段中，加载、验证、准备和初始化这四个阶段发生的顺序是确定的，而解析阶段则不一定，它在某些情况下可以在初始化阶段之后开始，这是为了支持Java语言的运行时绑定（也成为动态绑定或晚期绑定）。另外注意这里的几个阶段是按顺序开始，而不是按顺序进行或完成，因为这些阶段通常都是互相交叉地混合进行的，通常在一个阶段执行的过程中调用或激活另一个阶段。**当解析在初始化之后执行时，称为动态绑定或者晚期绑定，例如：晚期绑定的多态特性**。

### **1：加载**

主要是查找和加载二进制数据的过程，这里虚拟机会完成3件事：

*   通过一个类的全限定名来获取其定义的二进制字节流。
*   将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
*   在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。

这里加载二进制并没有强制从class文件中获取，也可以从zip包，网络中获取，也可以通过计算生成，譬如动态代理技术，或者是其他文件生成，譬如jsp文件生成对应的class类。加载阶段完成之后虚拟机将二进制字节流按照虚拟机格式存放在方法区中，然后在内存中虚拟出一个Class对象（在HotSport中，Class对象比较特殊，虽然是对象，但是存放在方法区中）。

### **2：链接**

#### 2.1 验证：验证是虚拟机链接的第一步，这一步主要是为了确保Class文件字节流的信息是否符合当前虚拟机的规范，并确保不会危害虚拟机自身的安全。验证阶段大约分为四个阶段的校验：文件格式校验，元数据验证，字节码验证，符号引用验证。

*   文件格式校验：是否符合Class文件规范，并且能被当前虚拟机处理
    
*   元数据验证：对字节码的信息进行语义分析
    
*   字节码验证：通过对数据流和控制流分析，确定程序语义时候合法，符合逻辑。完成之后还会对类的方法体进行验证分析，保证方法在运行时不会危害虚拟机。
    
*   符号引用验证：虚拟机将符号引用转换为直接引用。
    

2.2 准备：正式为类分配内存并设置类变量初始值，这些变量将会在方法区内分配内存。

public int value = 123;
public static int value = 123;
public static final int value = 123;

上面第一条语句在准备阶段过后初始值会被复制为0，在对象被new出来之后才会赋值为123。第二句虽然被static修饰，但是在准备阶段过后，初始值也是0，他会在类初始化时被赋值为123。第三句同时被static final修饰，他会在准备阶段将value赋值为123。

2.3 解析:虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号引用进行。

**符号引用**就是一组符号来描述目标，可以是任何字面量。

**直接引用**就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

### **3：初始化**

初始化阶段是类加载的最后一个阶段，在初始化阶段其实就是初始化类变量和其它资源，某种角度来说就是类构造器Clinit()方法的执行过程，Clinit()大约就下面这几件事：

*   为静态属性和静态语句块属性赋值
*   确保类的构造器中显式调用父类构造器（保证父类比之类初始化早）。
*   初始化一个类时，保证单线程去初始化，其他线程需要堵塞。

### **4：使用**

经过上面处理的ByteCode要不解释执行，要不翻译成机器码，取指执行。

### **5：卸载**

*   有启动类加载器加载的类型在整个运行期间是不可能被卸载的(jvm和jls规范)
*   被系统类加载器和标准扩展类加载器加载的类型在运行期间不太可能被卸载，因为系统类加载器实例或者标准扩展类的实例基本上在整个运行期间总能直接或者间接的访问的到，其达到unreachable的可能性极小.(当然，在虚拟机快退出的时候可以，因为不管ClassLoader实例或者Class(java.lang.Class)实例也都是在堆中存在，同样遵循垃圾收集的规则).
*   被开发者自定义的类加载器实例加载的类型只有在很简单的上下文环境中才能被卸载，而且一般还要借助于强制调用虚拟机的垃圾收集功能才可以做到.可以预想，稍微复杂点的应用场景中(尤其很多时候，用户在开发自定义类加载器实例的时候采用缓存的策略以提高系统性能)，被加载的类型在运行期间也是几乎不太可能被卸载的(至少卸载的时间是不确定的).

**类加载器 **
---------

“通过一个类的全限定名来获取该类的二进制字节流”这个动作放到java虚拟机外部去实现，以便让应用程序自己决定如何获取所需要的类，实现这个动作的代码模块就是“类加载器”。Java的虚拟机叫JVM。当我们在命令行执行 java HelloWord的时候，JVM干了下面几件事，也就是类加载过程：

*   产生一个Bootstrap Loader（引导类加载器）
*   Bootstrap Loader（引导类加载器）自动加载Extended Loader（标准扩展类加载器），并将其父Loader设为Bootstrap Loader。
*   Bootstrap Loader（引导类加载器）自动加载AppClass Loader（系统类加载器），并将其父Loader设为Extended Loader。
*   最后由AppClass Loader加载HelloWorld类。

下面就是各个加载器之间的层次关系，**这里父类加载器并不是通过继承关系来实现的，而是采用组合实现的**

![331425-20160621125943787-249179344](http://www.zzcode.cn/wp-content/uploads/2017/05/331425-20160621125943787-249179344.jpg)

站在Java虚拟机的角度来讲，只存在两种不同的类加载器：**启动类加载器**：它使用C++实现，是虚拟机自身的一部分；所有其他的类加载器：这些类加载器都由Java语言实现，独立于虚拟机之外，并且全部继承自抽象类java.lang.ClassLoader，这些类加载器需要由启动类加载器加载到内存中之后才能去加载其他的类。这些加载器可以分为一下三种类加载器：

*   **启动类加载器**：Bootstrap ClassLoader，负责加载存放在JDK\\jre\\lib(JDK代表JDK的安装目录，下同)下，或被-Xbootclasspath参数指定的路径中的，并且能被虚拟机识别的类库（如rt.jar，所有的java.*开头的类均被Bootstrap ClassLoader加载）。启动类加载器是无法被Java程序直接引用的。
*   **扩展类加载器**：Extension ClassLoader，该加载器由sun.misc.Launcher$ExtClassLoader实现，它负责加载DK\\jre\\lib\\ext目录中，或者由java.ext.dirs系统变量指定的路径中的所有类库（如javax.*开头的类），开发者可以直接使用扩展类加载器。
*   **应用程序类加载器**：Application ClassLoader，该类加载器由sun.misc.Launcher$AppClassLoader来实现，它负责加载用户类路径（ClassPath）所指定的类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

### **双亲委派模型**

从上图中可以看到类加载器之间的层次关系，这种层次关系称为双亲委派模型。（这种关系并不是通过继承实现，而是通过组合实现）。双亲委派模型除了顶层启动类加载器之外，其余的都要有自己的父类加载器。双亲委派的工作过程：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。

在JVM中具体的过程就是：

*   当AppClassLoader加载一个class时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器ExtClassLoader去完成。
*   当ExtClassLoader加载一个class时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给BootStrapClassLoader去完成。
*   如果BootStrapClassLoader加载失败（例如在$JAVA_HOME/jre/lib里未查找到该class），会使用ExtClassLoader来尝试加载；
*   若ExtClassLoader也加载失败，则会使用AppClassLoader来加载，如果AppClassLoader也加载失败，则会报出异常ClassNotFoundException。

#### 双亲委派模型的实现：

public Class<?> loadClass(String name)throws ClassNotFoundException {
            return loadClass(name, false);
    }
    
    protected synchronized Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException {
            // 首先判断该类型是否已经被加载
            Class c = findLoadedClass(name);
            if (c == null) {
                //如果没有被加载，就委托给父类加载或者委派给启动类加载器加载
                try {
                    if (parent != null) {
                         //如果存在父类加载器，就委派给父类加载器加载
                        c = parent.loadClass(name, false);
                    } else {
                    //如果不存在父类加载器，就检查是否是由启动类加载器加载的类，通过调用本地方法native Class findBootstrapClass(String name)
                        c = findBootstrapClass0(name);
                    }
                } catch (ClassNotFoundException e) {
                 // 如果父类加载器和启动类加载器都不能完成加载任务，才调用自身的加载功能
                    c = findClass(name);
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }

### **jvm加载机制**

*   **全盘负责**，当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入。
*   **父类委托**，先让父类加载器试图加载该类，只有在父类加载器无法加载该类时才尝试从自己的类路径中加载该类。
*   **缓存机制**，缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效。

### **自定义类加载器**

通常情况下，我们都是直接使用系统类加载器。但是，有的时候，我们也需要自定义类加载器。比如应用是通过网络来传输 Java 类的字节码，为保证安全性，这些字节码经过了加密处理，这时系统类加载器就无法对其进行加载，这样则需要自定义类加载器来实现。自定义类加载器一般都是继承自 ClassLoader 类，我们应该把加载类的业务逻辑在线findClass()方法中，在从上面loadClass源码里面可以看到如果没有父类加载失败，会调用findClass方法来完成加载。这样我们写出来的自定义类加载器就不会破坏双亲委派模型。这里我们只关注核心内容就是对字节码文件的获取。

public class MyClassLoader extends ClassLoader{
	private String root;
	public void setRoot(String root) {
		this.root = root;
	}
	@Override
	protected Class<?> findClass(String name) throws ClassNotFoundException {
		byte\[\] classData = loadClassData(name);
		if (classData == null) {
			return super.findClass(name);
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
	}
	
	private byte\[\] loadClassData(String className){
		try {
			 String fileName = root + File.separatorChar + className.replace('.', File.separatorChar) + ".class";
			 
			 InputStream input = new FileInputStream(fileName);
			 ByteArrayOutputStream out = new ByteArrayOutputStream();
			 byte\[\] buff = new byte\[1024\];
			 int length = 0;
			 while((length = input.read(buff))>0){
				 out.write(buff, 0, length);
			 }
			 return out.toByteArray();
		} catch (Exception e) {
			e.printStackTrace();
		}
		
		return null; 
	 }
	
	public static void main(String\[\] args) throws Exception {
		MyClassLoader loader = new MyClassLoader();
		loader.setRoot("E:/");
		Object o = loader.loadClass("cn.xiuyu.loader.TestClass");
		System.out.println(o);
		
	}
}