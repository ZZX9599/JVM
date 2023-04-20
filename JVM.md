# JVM

# 1:JVM的内存结构

![image-20220727170952160](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220727170952160.png)



## 1.1:程序计数器

作用：是记住下一条jvm指令的执行地址 

特点 

- 是`线程私有`的 
- 不会存在内存溢出

![image-20220727171139817](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220727171139817.png)

程序计数器就是为了记录上面的编号的，Java源代码被编译为class文件，也就是二进制的字节码文件

但是CPU不能执行这些二进制的class字节码，CPU只认识机器码，需要解释器执行为机器码给CPU执行

也就是还要先让JVM把class文件解释为平台支持的机器码，这个时候才可以被操作系统执行

每个线程都拥有自己的程序计数器



## 1.2:虚拟机栈

虚拟机栈是每个线程所需要的内存空间

虚拟机栈由多个栈帧组成【栈帧实际上就是一个个的方法】

栈帧：每个方法需要的内存空间

活动栈帧：每个虚拟机栈正在运行的那个栈帧



### 1:虚拟机栈的问题辨析 

- 垃圾回收是否涉及栈内存？  

  答：否，每次用完就弹出栈，自动回收，不需要立即回收

  

- 栈内存分配越大越好吗？    

  答：否，只是可以增加方法的调用个数，但是这样的话，线程数反而减少了

  

- 方法内的局部变量是否线程安全？ 

  答：如果方法内局部变量没有逃离方法的作用访问它是线程安全的，例如下面的例子

  ```java
  public void test(){
      int a=1;
      a++;
      System.out.println(a);
  }
  ```

  如果是局部变量引用了对象，并逃离方法的作用范围

  例如作为参数或者返回值，这样其他的线程可以访问到这个实例。这样就需要考虑线程安全

  ```java
  public int test(){
      int a=1;
      a++;
      return a;
  }
  ```

- 这个时候多个线程并发执行，实际上每个线程的栈都会开辟一个栈帧

  把a放进去，不是共享的，线程安全

- 如果是a是静态变量 static 修饰，那就是类的属性了，那么两个线程得到的结果，有一个就是3了



### 2:栈内存溢出

- 栈帧过多【方法过多】栈内存固定，栈帧过多就会溢出，常见于递归的溢出，转换 json 的互相套用
- 栈帧过大【栈帧所占用的内存太大】直接导致内存的溢出，不常见，只是有这个可能
- 设置栈内存大小方法 VM -options : -Xss256k



### 3:线程运行诊断

因为一个线程就是一个虚拟机栈，所以线程的运行和虚拟机栈息息相关

**如何定位：**

- 用top定位哪个进程对cpu的占用过高【定位进程】
- ps H -eo pid,tid,%cpu | grep 进程id【定位某个进程号的线程】
- jstack 进程id，可以根据线程id 找到有问题的线程，进一步定位到问题代码的源码行号
- 注意：jstack的线程显示的是16进制，记得换算一下



如何定位死锁：

- 还是使用jstack，会提示产生死锁的代码行号



## 1.3:本地方法栈

什么是本地方法：不是java代码编写的方法，java代码有限制，有时候不能跟操作系统底层打交道

可以通过本地方法【c，c++等语言编写的】，这一类方法被我们称为本地方法，这些方法运行使用的内存

叫做本地方法栈，`无论是java自带的api，还是框架等，都会调用这些本地方法`，例如Object类

使用ctrl+F12，查看全部的方法，看到有很多的方法声明为native，一般没有方法实现，这类实际上底层都是

c或者c++的实现，还有hashcode，wait等方法，其实都是`native修饰`的本地方法



## 1.4:堆

前面三个都是线程私有的，堆是一个线程共享的区域，通过new创建的对象都会使用堆内存

- 它是线程共享的，堆中对象都需要考虑线程安全的问题，一般就是使用new 创建的对象
- 有垃圾回收机制【堆里没有引用指向的对象在合适的时候就会被当作垃圾释放】

### 1:组成

堆内存的组成大致分为三个区域

1：新生代

2：老年代

3：永久代【JDK8被替换为了元空间】

### 2:堆内存溢出

堆内存的对象数量太多，而且都有指向使用，这个时候不能被回收，那么就可能导致堆内存溢出

示例：

```java
public class Test03 {
    public static void main(String[] args) {
        List list=new ArrayList<>();
        String str="周志雄";
        int i=0;
        while (true){
            list.add(str);
            str+=str;
            i++;
            System.out.println(i);
        }
    }
}
```

```elixir
异常信息:Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

配置jvm内存结构最大的堆大小 VM options : -Xmx2048m

配置jvm内存初始堆大小 VM options：-Xms10248m

排查堆内存问题，最好设置小一点，更加容易暴露出现问题



### 3:堆内存诊断

- jps工具             查看当前系统中有哪些 java 进程，并且显示进程编号
- jmap工具         查看堆内存占用情况 jmap - heap 进程id，只能查询一瞬间，不能连续监测
- jconsole工具    图形界面的，多功能的监测工具，可以连续监测，弥补jmap的不足



演示：

```java
public class Test03 {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("阶段一");
        Thread.sleep(60000);

        System.out.println("阶段二");
        //堆内存占用增加了10MB
        byte[] bytes = new byte[1024 * 1024 * 10];
        Thread.sleep(60000);
        //取消执行，没有了对象引用，是一个无效的对象，堆内存会被回收掉
        bytes=null;
        System.gc();

        System.out.println("阶段三");
        Thread.sleep(1000000L);
    }
}
```

阶段一的时候在终端使用jps命令查看Java进程，得到进程号 pid

阶段一的时候使用jmap -heap pid查看堆内存情况

阶段二的时候使用jmap -heap pid查看堆内存情况

阶段三的时候使用jmap -heap pid查看堆内存情况



进行对比发现阶段二比阶段一多了大概10M，阶段三比阶段二少了十几M

直接在终端使用jconole来连接进程，可以查看到堆内存，线程，死锁等信息



案例：在垃圾回收之后，堆内存占用仍然很高

使用 jvisualvm 可以来连接上去，也能看到内存，线程等信息，有一个强大的功能是堆Dump

可以抓取堆的快照信息和堆内存最大的对象，以及对象的信息等等

就能够知道堆内存到底是被哪个对象消耗了



## 1.5:方法区

### 1:方法区概念

方法区也是一块线程共享的区域【这一点和堆一样】

当虚拟机要使用一个类时，它需要读取并解析 Class 文件获取相关信息，再将信息存入到方法区

方法区会存储已被虚拟机加载的 **类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据**

存储什么东西呢？存储每个类的结构信息【成员变量，方法数据，以及成员方法和构造器方法等】

总之方法区就是存储类的信息的内存，同时还有运行时常量池

在虚拟机启动的时候被创建，逻辑上是堆的组成部分，在概念上定义了方法区，相当于是规范

但是不同的jvm厂商是否把方法区做成了堆的组成，这个是不一致的，并不强制堆的位置

hot-spot在jdk1.8以前，实现方式是永久代【这个时候方法区就是堆的一部分】

jdk1.8之后，实现是元空间，不再是堆的内存，而是操作系统的内存，方法区也可能会内存溢出



### 2:方法区的组成

jdk1.6【永久代】===》储存  类的信息Class，类加载器ClassLoader，运行时常量池，和String Table

<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220728110029368.png" alt="image-20220728110029368" style="zoom:67%;" />



jdk1.8【元空间，不占用堆，直接使用操作系统内存，StringTable被移动到了堆内存里】

![image-20220728110053099](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220728110053099.png)



### 3:方法区内存溢出

实例：

```java
public class Test04 extends ClassLoader{
    //extends ClassLoader 表示可以用来加载类的二进制字节码
    public static void main(String[] args) {
        int j=0;
        try{
            Test04 test04=new Test04();
            for(int i=0;i<10000;i++,j++){
                //ClassWriter：生成类的二进制字节码
                ClassWriter classWriter=new ClassWriter(0);
                classWriter.visit(Opcodes.V1_8,Opcodes.ACC_PUBLIC,
                        "Class"+i,null,"java/lang/Object",null);
                byte[] bytes = classWriter.toByteArray();
                test04.defineClass("Class"+i,bytes,0,bytes.length);
            }
        }finally {
            System.out.println(j);
        }
    }
}

```

经过测试，发现没有溢出，原因是因为我们1.8之后使用的元空间，操作系统的内存，很难溢出.直接设置方法区的大小：-XX:MaxMetaspaceSize=8m

```elixir
Exception in thread "main" java.lang.OutOfMemoryError: Compressed class space
```



### 4:方法区的内存溢出

什么时候会造成方法去的内存溢出呢？常见存在下面两种情况

- **spring**
- **mybatis**

为什么以上两种可能会造成内存溢出，只要是在运行期间动态生成二进制字节码文件来生成类

就可能会导致方法区的内存溢出，反正动态加载生成类都有可能产生这种情况



### 5:常量池

**常量池是在Java代码编译之后形成的字节码文件里面，属于一张静态的表，运行的时候才会加入到运行时常量池**



在学习运行时常量池之前先来看看常量池的概念

常量池：给指令提供常量符号来查找，把虚拟指令需要的东西查找出来，拼接在一起形成完整指令

很抽象，这里演示一下：

```java
public class Test05 {
    public static void main(String[] args) {
        System.out.println("ZZX");
    }
}
```

使用：javap -v Test05.class进行反编译得到的结果如下：

```apl
========================================类的基本信息=========================================
Classfile /E:/Code/JVM/out/production/day01/com/zzx/Test05.class
  Last modified 2022-7-28; size 529 bytes
  MD5 checksum a282a2c72bc9330f571787f70e246925
  Compiled from "Test05.java"
public class com.zzx.Test05
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
  
==========================================常量==============================================  
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // ZZX
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // com/zzx/Test05
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/zzx/Test05;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               Test05.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               ZZX
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               com/zzx/Test05
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
  
  
=======================================方法定义============================================
{
  public com.zzx.Test05();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/zzx/Test05;

==========================================main方法===========================================  	
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String ZZX
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "Test05.java"
```

**注意看：先getstatic 找到#2的指令，去Constant pool里面找，得到**

**#2 = Fieldref     #21.#22。这个时候又去找#21，#22，这样依次对比找下去，就能拼接成**

**System.out.println("ZZX")**



**总结：常量池，就是一张表，当java文件被编译成class文件之后，就会生成所说的class常量池，虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量等信息**



### 6:运行时常量池

运行时常量池：常量池是 *.class 文件中的

当该类被加载，它在编译时期生成的常量池的都会放入运行时常量池

并把里面的符号地址变为真实地址，当类加载到内存中后，jvm就会将class常量池中的内容存放到运行时常量池中，由此可知，运行时常量池也是每个类都有一个

这是刚刚类的常量池信息，如果这个类在运行的话，这些就会被放进运行时常量池，而且运行时不是根据#1，#2来查找，而是会转化为真实的内存地址来查找

```apl
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // ZZX
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // com/zzx/Test05
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/zzx/Test05;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               Test05.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               ZZX
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               com/zzx/Test05
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
```



### 7:常量池和串池关系

```java
public class Test06 {
    /**
     * 常量池的信息【编译时生成】在运行时都会被加载到运行时常量池
     * 这个时候a,b,ab都是常量池的符号，还不是java字符串对象
     * 当执行到了String s1="a"的时候，也就是ldc #2的时候，会把a符号变成"a"字符串对象
     * 当执行到了String s2="b"的时候，也就是ldc #3的时候，会把b符号变成"b"字符串对象
     * 当执行到了String s3="ab"的时候，也就是ldc #4的时候，会把ab符号变成"ab"字符串对象
     * 当代码执行涉及到 "str" 的时候，字符串str就会被放入字符串常量池【执行ldc操作】
     * 上面的对象会被放进StringTable["a","b","ab"]，最开始是空的
     */
    public static void main(String[] args) {
        String s1="a";
        String s2="b";
        String s3="ab";
    }
}
```



```apl
Constant pool:
   #1 = Methodref          #6.#24         // java/lang/Object."<init>":()V
   #2 = String             #25            // a
   #3 = String             #26            // b
   #4 = String             #27            // ab
   #5 = Class              #28            // com/zzx/Test06
   #6 = Class              #29            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/zzx/Test06;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               s1
  #19 = Utf8               Ljava/lang/String;
  #20 = Utf8               s2
  #21 = Utf8               s3
  #22 = Utf8               SourceFile
  #23 = Utf8               Test06.java
  #24 = NameAndType        #7:#8          // "<init>":()V
  #25 = Utf8               a
  #26 = Utf8               b
  #27 = Utf8               ab
  #28 = Utf8               com/zzx/Test06
  #29 = Utf8               java/lang/Object
  
  public static void main(java.lang.String[]);
   descriptor: ([Ljava/lang/String;)V
   flags: ACC_PUBLIC, ACC_STATIC
   Code:
     stack=1, locals=4, args_size=1
        0: ldc           #2                  // String a
        2: astore_1
        3: ldc           #3                  // String b
        5: astore_2
        6: ldc           #4                  // String ab
        8: astore_3
        9: return
```



### 8:字符串变量拼接

```java
public class Test06 {
    /**
     * 通过分析下面的反编译语句，会new一个StringBuilder，然后执行init，也就是无参构造
     * 然后把s1从LocalVariableTable读取出来，也就是StringBuilder的append，同理，s2也一样
     * 然后调用toString方法，会new一个新的String对象放进堆内存
     */
    public static void main(String[] args) {
        String s1="a";
        String s2="b";
        String s3="ab";
        String s4=s1+s2;
    }
}
```

字节码：

```apl
stack=2, locals=5, args_size=1
         0: ldc           #2                  // String a
         2: astore_1
         3: ldc           #3                  // String b
         5: astore_2
         6: ldc           #4                  // String ab
         8: astore_3
         9: new           #5                  // class java/lang/StringBuilder
        12: dup
        13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        16: aload_1
        17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        20: aload_2
        21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        27: astore        4
        29: return
        
 #astore就是存入本地，aload就是加载   
 LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      30     0  args   [Ljava/lang/String;
            3      27     1    s1   Ljava/lang/String;
            6      24     2    s2   Ljava/lang/String;
            9      21     3    s3   Ljava/lang/String;
           29       1     4    s4   Ljava/lang/String;       
```

思考：System.out.println(s3==s4)的结果

false，因为s3是在StringTable里的值，而s4会new一个新的String对象，new的对象在堆内存，所以是false



### 9:编译期优化

```java
public class Test06 {
    public static void main(String[] args) {
        String s1="a";
        String s2="b";
        String s3="ab";
        String s4=s1+s2;
        //并不是到常量池先找"a"，再找"b"，而是直接到常量池找"ab"
        String s5="a"+"b";
        System.out.println(s3==s5);
    }
}
```

字节码：

```apl
stack=2, locals=6, args_size=1
         0: ldc           #2                  // String a
         2: astore_1
         3: ldc           #3                  // String b
         5: astore_2
         6: ldc           #4                  // String ab
         8: astore_3
         9: new           #5                  // class java/lang/StringBuilder
        12: dup
        13: invokespecial #6                  // Method java/lang/StringBuilder."<init>":()V
        16: aload_1
        17: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        20: aload_2
        21: invokevirtual #7                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
        24: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
        27: astore        4
        29: ldc           #4                  // String ab
        31: astore        5
        33: return
```



**可以看到第六行和第29行都是去StringTable里去找#4**

**六行执行的时候，还没有ab，就会把ab字符串对象放进**

**StringTable里面，然后第29行再次执行的时候**

**直接去StringTable找到了"ab"，都是一个对象，所以是true**



**对比以上String s4=s1+s2 和 String s5="a"+"b"，为什么一个是false，一个是true，其实我们可以这样**

**想，s1和s2属于变量，是可能改变的值，所以我们不能把他直接放进常量池，而应该创建对象放进堆里面**

**而String s5="a"+"b"相当于在编译的时候都确定了已经是常量了，就没有必要再创建对象了，直接去**

**找，找到了就直接引用，找不到的话，再创建这个对象放进StringTable**



### 10:字符串延迟加载

```java
public class Test06 {

    public static void main(String[] args) {  //2047
        String s1="a";  //2048
        String s2="b";  //2049
        String s3="ab"; //2050
        String s4="a";  //2050
        String s5="b";  //2050
        String s6="ab"; //2050
    }
}
```

debug模式运行，查看Memory就可以看到字符串常量池里字符串的数量，我们发现只有执行到了对应的代码，才会创建字符串对象存进StringTable里面，这也验证了只有代码执行到对应的字面量，才会入池



### 11:intern的使用

**intern 1.8:将这个字符串对象尝试放入串池，如果有则并不会放入**

**如果没有则放入【移动到】串池，会把串池中的对象返回**

```java
public class Test08 {
    public static void main(String[] args) {
        /**
         * StringTable["a","b"]
         */
        //String x="ab";
        //注意这行代码并不涉及字面量"ab"，所以字符串常量池并没有"ab"，除非代码涉及到"ab"
        String s=new String("a")+new String("b");
        //把这个字符串放进字符串常量池，有的话就不放入
        //没有的话就把这个对象s放进字符串常量池
        //返回的结果都是字符串常量池对象
        //所以 intern和s都是字符串常量池里面的"ab"

        /**
         * StringTable["a","b","ab"]
         */
        String intern = s.intern();

        //打印的是字符串常量池的"ab"
        System.out.println(intern);
        //打印的是字符串常量池的"ab"
        System.out.println(s);
        //都是StringTable的内容 true
        System.out.println(s=="ab");
        //都是StringTable的内容 true
        System.out.println(intern=="ab");
    }
}
```



intern方法：会把字符串对象放进常量池，分为两种情况

- 1:字符串常量池已经有了这个值，那么就不会放入了
- 2:字符串常量池没有，就会把对象放入【移动】常量池
- 返回的结果都是字符串常量池的对象

**自己测试先执行String x="ab"  答案是false  true**



**intern 1.6:将这个字符串对象尝试放入串池，如果有则并不会放入**

**如果没有会把此对象复制一份， 放入串池， 会把串池中的对象返回**



**差别：一个是移动，一个是复制**

**String intern = s.intern()  **

**在Intern1.8  s移动过去，成为了StringTable的对象**

**在Intern1.6  intern是复制到StringTable里面去的，是StringTable的对象，但是s依然在堆里**



### 12:StringTable位置

在String1.6，StringTable在方法区里面，方法区在永久代，所以串池是永久代的一部分

在String1.8，方法区在元空间，但是StringTable不在方法区了，串池转到了堆，是堆的一部分

永久代的回收效率很低，需要Full GC才会执行垃圾回收，所以被优化更改了

StringTable为什么要放到堆

因为永久代的回收频率比较低，只在FullGC的时候才会被回收，FullGC只会在老年代或者永久代空间不足时才会触发。如果有大量的字符串被创建，放在永久代，由于永久代的回收频率低，会导致很多无用的字符串得不到及时的回收，而导致永久代空间不足。如果放到堆里，能够及时回收内存。

<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220728110029368.png" alt="image-20220728110029368" style="zoom: 67%;" />



![image-20220728110053099](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220728110053099.png)

JDK1.8中字符串常量池和运行时常量池逻辑上属于方法区，但是实际存放在堆内存中

因此既可以说两者存放在堆中，也可以说两则存在于方法区中，这就是造成误解的地方



在jdb1.6测试，字符串特别多而且存在引用，报错永久代溢出

在jdb1.8测试，字符串特别多而且存在引用，报错等同于堆内存溢出



### 13:StringTable垃圾回收

字符串常量池实际上也会执行垃圾回收的，并不是不执行垃圾回收

```java
public class StringGCTest {
    /**
     * -Xms15m -Xmx15m -XX:+PrintGCDetails
     */
    public static void main(String[] args) {
        for (int i = 0; i < 100000; i++) {
            String.valueOf(i).intern();
        }
    }
}
```



```
[GC (Allocation Failure) [PSYoungGen: 4096K->488K(4608K)] 4096K->727K(15872K), 0.0021643 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 4608K, used 3655K [0x00000000ffb00000, 0x0000000100000000, 0x0000000100000000)
  eden space 4096K, 77% used [0x00000000ffb00000,0x00000000ffe17cc0,0x00000000fff00000)
  from space 512K, 95% used [0x00000000fff00000,0x00000000fff7a020,0x00000000fff80000)
  to   space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
 ParOldGen       total 11264K, used 239K [0x00000000ff000000, 0x00000000ffb00000, 0x00000000ffb00000)
  object space 11264K, 2% used [0x00000000ff000000,0x00000000ff03bc40,0x00000000ffb00000)
 Metaspace       used 3327K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 362K, capacity 388K, committed 512K, reserved 1048576K
SymbolTable statistics:
Number of buckets       :     20011 =    160088 bytes, avg   8.000
Number of entries       :     13583 =    325992 bytes, avg  24.000
Number of literals      :     13583 =    600768 bytes, avg  44.229
Total footprint         :           =   1086848 bytes
Average bucket size     :     0.679
Variance of bucket size :     0.682
Std. dev. of bucket size:     0.826
Maximum bucket size     :         6

===========================================重点============================================
StringTable statistics:
#桶的个数
Number of buckets       :     60013 =    480104 bytes, avg   8.000
#键值对个数
Number of entries       :     58728 =   1409472 bytes, avg  24.000
#字符串常量池字符串的个数
Number of literals      :     58728 =   3369144 bytes, avg  57.369
#占用字节数
Total footprint         :           =   5258720 bytes
Average bucket size     :     0.979
Variance of bucket size :     0.776
Std. dev. of bucket size:     0.881
Maximum bucket size     :         5
```

我们存了十万，实际上只加入了五万多个，而且执行了垃圾回收





### 14:StringTable性能调优



StringTable的实现数据结构是Hash结构

- 调整 -XX:StringTableSize=桶个数 StringTable底层是数组+链表，桶的个数就是数组长度，如果你的String特别多的话，适当增加桶个数，可以减少hash碰撞的概率
- 考虑将字符串对象放入字符串常量池，也就是使用intern把堆内存的String对象放进StringTable，例如你有100万个字符串，全部放入字符串常量池【堆内存内】，就很大，字符串这么多，难免这个字符串有很多重复的，我们可以在代码层面直接使用Intern去重再存入StringTable里面，就能减少很多的内存消耗



## 1.6:直接内存

注意：不属于JVM，只是这里提一下

直接内存是 Java 堆外、直接向系统申请的内存区间，不是虚拟机运行时数据区的一部分

也不是《Java 虚拟机规范》中定义的内存区域

- 常见于 NIO 操作时，用于数据缓冲区分配
- 创建和回收成本较高，但读写性能对比普通IO，高很多
- 不受 JVM 内存回收管理，但是JVM垃圾回收会间接的回收这块区域



java本身不具备调用磁盘的能力，必须调用操作系统的函数【本地方法才可以，native修饰】



### 1:IO对比NIO操作

**普通的io操作：java缓冲区的内容存在了两份，系统缓冲区和java缓冲区都存放了，造成不必要的复制**

**文件先从磁盘读取到系统缓冲区，但是java代码不能读取系统缓冲区的内容，所有需要在java堆内存创建**

**一块java的缓冲区，java代码再读取java的缓冲区**

**java代码在这个时候才能读取到java缓冲区的内容，造成了不必要的复制**

![image-20220730092758604](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220730092758604.png)



**NIO操作文件：直接把java堆内存和系统内存合为了一份，少了一次缓冲区复制操作**

**使用NIO操作之后，java的堆内存和系统内存之间存在交集，java代码可以直接访问这一块的交集部分**



![image-20220730092935536](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220730092935536.png)



### 2:直接内存溢出

**测试：NIO使用ByteBuffer来分配空间，每次循环就分配500MB**

**报错：Direct buffer memory**



### 3:内存释放

```java
public class Test09 {
    public static void main(String[] args) throws IOException {
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024 * 1024 * 1024);
        System.out.println("成功");
        System.in.read();
        System.out.println("释放");
        byteBuffer=null;
        System.gc();
        System.in.read();
    }
}
```

不是说直接内存不被JVM管理吗？

实际上：直接内存的释放必须主动通过`UnSafe对象的方法`来管理

UnSafe是一个非常底层的类，方法也都是跟系统底层打交道的方法了

ByteBuffer里面的方法的源码实际上是调用了UnSafe的分配【allocateMemory】和释放【freeMemory】方法

但是JVM执行垃圾回收的时候，会跟UnSafe的对象的释放方法关联到一起



### 4:分配和回收原理 

使用了 Unsafe 对象完成直接内存的分配内存和回收，并且回收需要主动调用 freeMemory 方法 

ByteBuffer 的实现类内部，使用了 Cleaner （虚引用）来监测 ByteBuffer 对象

虚引用的特点：一旦关联的对象并不存在了，就会把虚引用对象放进引用队列里面，虚引用对象会带有直接内存的地址，然后引用队列里面会处理掉这块内存，使用的是UnSafe.freeMemory来释放内存

则一旦 NIO的 ByteBuffer 对象被垃圾回收，那么就会由 ReferenceHandler 线程通过 Cleaner 的 clean 方法

调用 freeMemory 来释放直接内存

**释放直接内存：UnSafe.freeMemory**



### 5:显示禁用垃圾回收的影响

使用JVM参数：-XX:+DissableExplicitGC ===>标志自动将所有的system.gc()调用转换成一个空操作

让System.gc()无效，因为jvm执行一个堆内存的“全部清扫”，比一个常规的GC操作要昂贵好几个数量级

这样可以极大地提升效率，但是这样的话，我们就不能在执行System.gc()的时候间接的释放直接内存了

这样就容易造成jvm内存充足，但是直接内存不足的情况，但是JVM没有执行GC，则直接内存也没有执行垃

圾回收，解决方案：还是自己来管理UnSafe对象，见Netty



# 2:JVM垃圾回收

## 2.1:如何判断对象可以回收



### 1:引用计数法

只要一个对象被引用，就让计数加一，然后不被引用了，计数减一，当计数为0的时候，就达到了回收条件



**问题：**

![image-20220730100236078](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220730100236078.png)

假如这两个都是无用的对象了，但由于循环的引用，使用引用计数法的时候，就不能被回收【python前期使用】



### 2:可达性分析算法

- java 虚拟机中的垃圾回收器采用可达性分析来探索所有存活的对象 
- 扫描堆中的对象，看是否能够沿着 GC Root对象 为起点的引用链找到该对象，找不到，表示可以回收 
- 哪些对象可以作为 GC Root



过程：首先确定根对象【可以理解为肯定不能被作为垃圾回收的对象】然后扫描堆内存所有的对象，看这些对象是否被根对象直接或者间接使用【如果存在直接或间接引用，就不能被删除，反之就是可以作为立即来回收的对象】





因为可达性算法的特性，所以可以在gc接触不到的地方作为GC Roots，可理解为肯定不会被作为垃圾的对象

**什么可以作为根对象GC ROOT呢？**

- a.java虚拟机栈中的引用的对象。
- b.方法区中的类静态属性引用的对象。 （一般指被static修饰的对象，加载类的时候就加载到内存中）
- c.方法区中的常量引用的对象
- d.本地方法栈中的JNI（native方法）引用的对象



### 3:java五种引用



**强引用使用实线，虚线分别代表其他的**引用

![image-20220730102608682](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220730102608682.png)



**1：强引用：**

我们平时用的 = 赋值都是强引用

只要能沿着根对象找下去，找得到的就不会被回收，例如A1对象

什么时候A1被回收？只有从GC ROOT到B对象这条线对A1的强引用都断开了，就会被回收

就是C->A1和B->A1都断开了，则A1可以被回收



**2：软引用：**

A2对象被GC ROOT软引用了，被B直接引用，存在强引用，肯定不会回收，但假如B对象和A2的强引用关系消失了，只存在了一个软引用，那么A2是可以被回收的，GC ROOT认为不重要，`在内存不足`时执行垃圾回收的时可以被回收



**3：弱引用：**

A3对象被GC ROOT软引用了，被B直接引用，假如B对象和A3的强引用关系消失了，只存在了一个弱引用，那么A2是可以被回收的，GC ROOT认为不重要，在垃圾回收时，`不管内存充不充足`都直接回收



**对比：只存在软引用和弱引用的时候，在内存不足的时候会回收的是软引用，弱引用直接回收**

**软引用和弱引用本身也占用内存，可以配合引用队列，这样的话软引用和弱引用自身会进入引用队列**

**可对引用队列做进一步处理回收，当然也可以不配合引用队列使用**



**4：虚引用：**

```java
public class Test09 {
    public static void main(String[] args) throws IOException {
        // NIO的ByteBuffer使用的是直接内存
        ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024 * 1024 * 1024);
        System.out.println("成功");
        System.in.read();
        System.out.println("释放");
        byteBuffer=null;
        System.gc();
        System.in.read();
    }
}
```

注意：虚引用必须配合引用队列使用

虚引用引用的对象，都带有一块直接内存，直接内存见上面的笔记【属于操作系统】不能被JVM管理，当虚引用的对象不存在直接引用了【byteBuffer=null】就会把虚引用对象放进引用队列里面，虚引用对象会带有直接内存的地址，然后引用队列里面会处理掉这块内存，使用的是UnSafe.freeMemory来释放内存，所以System.gc()实际上并不是它来释放内存，而是提示引用队列ByteBuffer这个对象没有引用了，应该放进引用队列，从而拿到直接内存的地址，从而执行UnSafe.freeMemory来释放直接内存



**5：终结器引用:**

注意：终结器引用必须配合引用队列使用

当对象重写了终结方法finalize，属于Object类的方法，这个时候会创建一个终结器引用，当不存在直接引用的时候，会把终结器引用放进引用队列里，由一个优先级很低的线程来执行释放资源，正是因为优先级很低，所以很可能很久都不执行，就释放不了资源，所以我们并不推荐使用这个来释放垃圾



### 4:软引用的使用



**以下对new byte对象全部都是强引用，不会被垃圾回收**

**List直接关联byte[]**

```java
public class Test09 {
    public static void main(String[] args) throws IOException {
        List list=new ArrayList();

        for(int i=0;i<10;i++){
            list.add(new byte[1024*1024*1000]);
        }

        System.in.read();
    }
}
```

**Exception in thread "main" java.lang.OutOfMemoryError: Java heap space**



**软引用实现SoftReference**

**设置堆内存为20MB**

**List关联SoftReference，SoftReference关联byte[]**

```java
public class Test09 {
    public static void main(String[] args) throws IOException {
        List<SoftReference<byte[]>> list=new ArrayList();
        
        //软引用对象
        SoftReference<byte[]> softReference=null;
       
        for(int i=0;i<10;i++){
            softReference= new SoftReference<>(new byte[1024 * 1024 * 5000]);
            list.add(softReference);
        }

        System.out.println(list.size());
        for(SoftReference<byte[]> soft:list){

            System.out.println(soft.get());
        }

        System.in.read();
    }
}
```

**结果：不报错了，证明在内存不足的时候回收了软引用的对象，最后输出的时候存在很多null**



**问题：虽然软引用对象里的byte[]数组对象以及被移除了，但是软引用还被list引用着**

**我们要借助引用队列来把软引用对象和list的连接断开**

```java
public class Test09 {
    public static void main(String[] args) throws IOException {
        List<SoftReference<byte[]>> list=new ArrayList();

        //软引用对象
        SoftReference<byte[]> softReference=null;

        //引用队列
        ReferenceQueue<byte[]> referenceQueue=new ReferenceQueue<>();

        for(int i=0;i<10;i++){
            /**
             * 构造方法传递引用队列，关联起来
             * 当软引用所关联的对象在内存不足回收的时候，软引用自身会被加入到引用队列
             */
            softReference= new SoftReference<>(new byte[1024 * 1024 * 5000],referenceQueue);
            list.add(softReference);
        }

        /**
         * 拿到引用队列最先进来的，并且移除
         */
        Reference<? extends byte[]> poll = referenceQueue.poll();

        //不为空，证明存在被回收的byte数组对象，对应的软引用已经被移动到队列
        while (poll!=null){
            list.remove(poll);
            poll=referenceQueue.poll();
        }

        System.out.println(list.size());

        for(SoftReference<byte[]> soft:list){
            System.out.println(soft.get());
        }
        System.in.read();
    }
}
```



### 5:弱引用的使用

**使用和SoftReference一样的方式，对象是WeakReference，一样可以使用引用队列的对象来消除无效连接**

**不同点就是软引用在内存不足的时候回收，而弱引用会直接回收**



## 2.2:垃圾回收算法



### 1:标记清除算法



**存在两个阶段**

- **标记无效的内存**
- **清除无效的内存**



<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220802155257488.png" alt="image-20220802155257488" style="zoom: 50%;" />



**先把需要的标记为灰色，二阶段进行清除**

**优点：清除操作快，垃圾回收快**

**缺点：容易产生内存碎片，不会对内存再进行整理，例如再来一个数组，就不能插入到上面的任意一个空白位置，但是实际上所有白色的位置累加起来是足够添加数组的。这就是内存碎片**



### 2:标记整理算法



**存在两个阶段**

- **标记无效的内存**
- **清除无效的内存+整理内存碎片**



前面的一个阶段和标记清除是一致的

第二阶段和标记清楚的第二个阶段不同



![image-20220802155931375](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220802155931375.png)

**优点：不存在内存碎片问题**

**缺点：内存地址变化，需要重塑对象的内存地址执行，效率低下**



### 3:内存复制算法



**存在三个阶段**

- **标记有效的存活的内存**
- **复制有效的内存到另一块内存TO**
- **清空FROM，交换FROM和TO的位置**

![image-20220802160133170](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220802160133170.png)



![image-20220802160533765](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220802160533765.png)

**注意：上面的From和To加起来的总和就是之前的标记清除和标记整理算法的总内存区域，只是拆分成了两份**

**优点：这样就能保证TO的永远是空，FROM永远都是有效的，而且不存在内存碎片**

**缺点：需要双倍的内存空间**



**以上三种垃圾回收算法在JVM里都存在，并不是就使用一种，都是结合着使用**



### 4:分代回收算法

当前虚拟机的垃圾收集都采用分代收集算法，这种算法没有什么新的思想，只是根据对象存活周期的不同将内存分为几块

一般将 java 堆分为新生代和老年代，这样我们就可以根据各个年代的特点选择合适的垃圾收集算法



**延伸面试问题：** HotSpot 为什么要分为新生代和老年代？



比如在新生代中，每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率比较高，没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集

## 2.3:分代垃圾回收

<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220802160813734.png" alt="image-20220802160813734"  />

- 对象首先分配在伊甸园区域
- 新生代空间不足时，触发 minor gc，伊甸园和 from 存活的对象使用 copy 复制算法复制到 to 中，存活的对象年龄加 1并且交换 from to
- minor gc 会引发 stop the world，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行
- 当对象寿命超过阈值时，会晋升至老年代，最大寿命是15（4bit）
- 当老年代空间不足，会先尝试触发 minor gc，如果之后空间仍不足，那么触发 full gc，STW的时间更长
- 经常使用的对象在老年代
- 用完就可以丢弃的在新生代



```apl
1：最开始对象出生在伊甸园，伊甸园快满了的时候，就会触发一次新生代的垃圾回收【新生代垃圾回收一般称为 Minor GC】
2：然后根据可达性分析算法来进行垃圾回收，然后根据复制算法，把有用的对象复制到 幸存区To  寿命加一
3：伊甸园没用的对象就会被回收，然后交换From 和 To【From就是有用的，To又变成了空的】
4：继续放新的对象进入伊甸园
5：伊甸园又满了，继续触发【Minor GC】
6：把伊甸园有用的对象拿到 TO 里去，寿命加一，再看幸存区From里有用的，移动到TO，寿命再加一，变为二
7：再次进行交换From和To【To又变成了空的】
8：当幸存区的对象寿命达到了阈值，超过了15，就会被晋升到老年代【垃圾回收频率很低】
9：当老年代也满了之后，触发【Full GC】直接把新生代和老年代都进行清理

注意：以上进行垃圾回收的时候，用户线程那些是不能再运行的，所以我们也不能太频繁的进行垃圾回收
```



## 2.4:JVM相关的参数

![image-20220802162713273](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220802162713273.png)



- SurvivorRatio:默认是8  八份给伊甸园，剩下两份给幸存区From和To
- 新生代的GC就是叫做GC
- 老年代的GC叫做Full GC
- 新生代的空间实在是不足的话，直接会跑到老年代，不一定非要达到15才回到老年代



## 2.5:大对象

例如设置堆内存为20MB，新生代10MB，老年代10MB，新生代的From和To各自为1MB

**我们直接new一个byte数组[8MB]，因为新生代一共10MB，幸存区from和to默认占用了2MB，伊甸园初始化的时候也占用了一部分，所有8MB放不下，这个时候会把对象直接放进老年代，不会触发新生代的GC**

```java
public class Test11 {
    public static void main(String[] args) {
        List list=new ArrayList();
        Thread thread = new Thread(() -> {
            list.add(new byte[1024 * 1024 * 8]);
            list.add(new byte[1024 * 1024 * 8]);
        });
        thread.start();
        System.out.println("主线程正常");
    }
}
```

**结果：发现主线程正常输出**

**思考：堆内存共享，但是新生代放不下，老年代也只能放下一个8MB的空间，为什么不报错**

**原因：当一个线程出现了异常，会把这个线程所占用的内存全部释放掉，所以主线程没问题，这个时候内存足够**



## 2.6:垃圾回收器

**如果说收集算法是内存回收的方法论，那么垃圾收集器就是内存回收的具体实现。**

虽然我们对各个收集器进行比较，但并非要挑选出一个最好的收集器。因为直到现在为止还没有最好的垃圾收集器出现，更加没有万能的垃圾收集器，**我们能做的就是根据具体应用场景选择适合自己的垃圾收集器**。试想一下：如果有一种四海之内、任何场景下都适用的完美收集器存在，那么我们的 HotSpot 虚拟机就不会实现那么多不同的垃圾收集器了



### 1:串行:Serial

- 堆内存小
- 单线程
- 适用于堆内存小，个人电脑使用



**新生代采用标记-复制算法，老年代采用标记-整理算法**



**打开串行垃圾回收器的JVM参数：**

**-XX:+UseSerialGC = Serial + SerialOld**



**Serial【新生代，采用内存复制算法】**

**Serial：串行垃圾收集器，作用于新生代，是指使用单线程进行垃圾回收，采用复制算法，新生代基本都是复制算法**



**SerialOld【老年代，采用标记整理算法】**

**Serial old**：执行老年代垃圾回收的串行收集器，内存回收算法使用的是**标记-整理算法**，同样也采用了串行回收和 STW 机制



![image-20220803094911621](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220803094911621.png)



**在垃圾回收的时候，其他的用户线程会在安全点停下来**



**serial和serialOld都属于单线程的垃圾回收，因为是串行方式**



****



### 2:吞吐量优先:ParallelGC

- 单次的垃圾回收不一定很短，但是总时间短，也就是回收次数少点，平均来讲：单位时间的STW最短

- 多线程
- 堆内存较大，多核CPU
- 适合服务器电脑



**打开吞吐量优先垃圾回收器的JVM参数：**

**-XX:+UseParallelGC 和 -XX:+UseParallelOldGC【JDK8默认开启】**

**-XX:GCTimeRatio=ratio**

**-XX:MaxGCPauseMillis=ms **

**-XX:ParallelGCThreads=n**



**UseParallelGC【代表是新生代垃圾回收器，复制算法】**

**UseParallelOldGC【代表使用的是老年代垃圾回收器，采用标记整理算法】**

**这两个都是Paral  代表并行的【用户线程不能运行】多个线程的，回收的线程数量跟CPU的核心数量有关**

**GCTimeRatio=ratio【垃圾收集花费的时间与垃圾收集之外花费的时间】**

**假设-XX:GCTimeRatio=19，则垃圾收集时间为1/(1+19)，默认值为99，即1%时间用于垃圾收集，一般为19】**

**MaxGCPauseMillis=ms【垃圾回收的最大暂停时间，默认200】 **

**ParallelGCThreads=n【指定垃圾回收器的线程数】**

![image-20220803095614393](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220803095614393.png)



****



### 3:响应时间优先:CMS

- 尽可能的降低单次STW的时间，用户体验好

- 多线程
- 堆内存较大，多核CPU
- 适合服务器电脑
- 一款基于标记清除的垃圾回收器，可以并发



CMS 全称 Concurrent Mark Sweep，是一款**并发的、使用标记-清除**算法、针对老年代的垃圾回收器，其最大特点是**让垃圾收集线程与用户线程同时工作**

CMS 收集器的关注点是尽可能缩短垃圾收集时用户线程的停顿时间，停顿时间越短（**低延迟**）越适合与用户交互的程序，良好的响应速度能提升用户体验

**打开响应时间优先垃圾回收器的JVM参数：**



**-XX:+UseConcMarkSweepGC 和 -XX:+UseParNewGC ~ SerialOld【碎片过多的时候会变为SerialOld】**

**UseConcMarkSweepGC：标记清除算法的并发的垃圾回收器，并发：垃圾回收的时候用户线程也可以抢占CPU**

**-XX:ParallelGCThreads=n【并行执行的线程，默认是CPU核心】**

**-XX:ConcGCThreads=threads【垃圾回收的线程个数，假设是1，则其他的CPU分给用户线程】**

**-XX:CMSInitiatingOccupancyFraction=percent**

**【其他用户线程运行的时候产生新的垃圾，叫做浮动垃圾，比如设置为80%，则老年代在内存为80%的时候就执行了垃圾回收，防止其他用户线程在我们执行垃圾回收的时候产生的垃圾没有地方存放】**

**-XX:+CMSScavengeBeforeRemark【是否可以退化为SerialIOld的开关】**

![image-20220803100723233](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220803100723233.png)



**标记的时候，用户线程停止，很快，然后用户线程和垃圾回收线程一起执行。重新标记的时候也会暂停**

**标记结束之后，用户线程依然运行。所以只有在标记的时候会暂停，正符合【响应时间优先】的字眼**



**分为以下四个流程：**

- **初始标记：使用 STW 出现短暂停顿，仅标记一下 GC Roots 能直接关联到的对象，速度很快**
- **并发标记：进行 GC Roots 开始遍历整个对象图，在整个回收过程中耗时最长，不需要 STW，可以与用户线程并发运行**
- **重新标记：修正并发标记期间因用户程序继续运作而导致标记产生变动的那一部分对象的标记记录，比初始标记时间长但远比并发标记时间短，需要 STW（不停顿就会一直变化，采用写屏障 + 增量更新来避免漏标情况）**
- **并发清除：清除标记为可以回收对象，不需要移动存活对象，所以这个阶段可以与用户线程同时并发的**

**Mark Sweep 会造成内存碎片，不把算法换成 Mark Compact 的原因：Mark Compact 算法会整理内存，导致用户线程使用的对象的地址改变，影响用户线程继续执行**

**在整个过程中耗时最长的并发标记和并发清除过程中，收集器线程都可以与用户线程一起工作，不需要进行停顿**



**缺点：**

- 吞吐量降低：在并发阶段虽然不会导致用户停顿，但是会因为占用了一部分线程而导致应用程序变慢，CPU 利用率不够高

- CMS 收集器**无法处理浮动垃圾**，可能出现 Concurrent Mode Failure 导致另一次 Full GC 的产生

  浮动垃圾是指并发清除阶段由于用户线程继续运行而产生的垃圾（产生了新对象），这部分垃圾只能到下一次 GC 时才能进行回收。由于浮动垃圾的存在，CMS 收集需要预留出一部分内存，不能等待老年代快满的时候再回收。如果预留的内存不够存放浮动垃圾，就会出现 Concurrent Mode Failure，这时虚拟机将临时启用 Serial Old 来替代 CMS，导致很长的停顿时间

- 标记 - 清除算法导致的空间碎片，往往出现老年代空间无法找到足够大连续空间来分配当前对象，不得不提前触发一次 Full GC；为新对象分配内存空间时，将无法使用指针碰撞（Bump the Pointer）技术，而只能够选择空闲列表（Free List）执行内存分配



**CMS存在的最大问题：1：吞吐量下降  2：并发的过程无法清除浮动垃圾  3：CMS存在内存碎片，所以有可能在并发的过程出现并发失败，这个时候就会退化为SerialOld的垃圾回收器，进行一次全面的垃圾回收，单线程的，时间花费会变得很长，不符合响应时间优先的策略**



****



### 4:G1垃圾回收器

定义：Garbage First 

2004 论文发布 

2009 JDK 6u14 体验 

2012 JDK 7u4 官方支持 

2017 JDK 9 默认 

适用场景 同时注重吞吐量（Throughput）和低延迟（Low latency）

默认的暂停目标是 200 ms 

适合超大堆内存，会将堆划分为多个大小相等的 Region 【每个区域都可以作为独立的伊甸园，幸存区，老年代等】

整体上是 标记+整理 算法，两个区域之间是 复制 算法



相关 JVM 参数：在JDK9及其之后就不用开了，已经默认打开 

-XX:+UseG1GC 

-XX:G1HeapRegionSize=size 

-XX:MaxGCPauseMillis=time



#### 1:G1的特点

G1（Garbage-First）是一款面向服务端应用的垃圾收集器，**应用于新生代和老年代**、整体上采用标记-整理算法、在两个区域之间采用的是复制算法。软实时、低延迟、可设定目标（最大 STW 停顿时间）的垃圾回收器，用于代替 CMS，适用于较大的堆（>4 ~ 6G），在 JDK9 之后默认使用 G1



#### 2:垃圾回收阶段

![image-20220823145559402](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823145559402.png)

**Young Collection:新生代垃圾回收**

**Concurrent Mark:老年代内存不足的时候，触发新生代垃圾回收并进行并发标记**

**Mixed Collection:混合收集【对所有区域都进行一次垃圾回收，比较彻底】**



Young Collection：会STW

![image-20220823151702056](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823151702056.png)



E代表伊甸园

S代表幸存区

O代表老年代

E满了之后采用复制算法到幸存区

S的对象比较多了，把需要的对象，会通过复制算法移动到S，达到年龄的，通过复制算法到O

幸存区满了之后采用复制算法到老年代

****



Young Collection + CM【并发标记】：

在 Young GC 时会进行 GC Root 的初始标记【找到根对象GC ROOT】：会STW 

老年代占用堆空间比例达到阈值时，进行并发标记【找到跟GC ROOT有关的】（不会 STW），由下面的 JVM 参数决定

-XX:InitiatingHeapccupancyPercent=percent【默认45%】

也就是老年代占比达到堆内存的45%，就会进行并发标记了

![image-20220823152130212](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823152130212.png)



****



Mixed Collection

会对 E、S、O 进行全面垃圾回收

最终标记（Remark）会 STW【标记的是在并发过程中产生的垃圾】 

拷贝存活（Evacuation）会 STW【注意：并不是所有的老年代垃圾的都被复制清理了】

![image-20220823152517202](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823152517202.png)



**注意：上面的拷贝存活，为什么没有把全部的O都指向一个O，原因是因为O如果比较多的话，回收时间太长**

**默认有一个参数的限制：XX:MaxGCPauseMillis=ms，最大的暂停时间**

**所以在老年代的垃圾很多的时候，就会优先回收垃圾多的区域，并没有一次性回收完，因为时间达不到，但是如果垃圾较少，就可以一次性回收掉**



#### 3:Full GC

- SerialGC 
  - 新生代内存不足发生的垃圾收集 - minor gc 
  - 老年代内存不足发生的垃圾收集 - full gc 
- ParallelGC 
  - 新生代内存不足发生的垃圾收集 - minor gc 
  - 老年代内存不足发生的垃圾收集 - full gc 
- CMS 
  - 新生代内存不足发生的垃圾收集 - minor gc 
  - 老年代内存不足 
    - 情况一：并发收集垃圾够的话，就不算Full GC
    - 情况二：并发失败，退化为SerialOld，属于Full GC
- G1 
  - 新生代内存不足发生的垃圾收集 - minor gc 
  - 老年代内存不足
    - 情况一：MaxGCPauseMillis=ms【默认45%】就会出现并发标记和混合收集阶段，如果回收速度大于并发产生的速度，这个时候还不算是Full GC，属于并发收集
    - 情况二：并发收集的速度没跟上的话，就退化为串行化的收集，属于Full GC



#### 4:Young Collection跨代引用



存在的问题：在Young Collection的时候，会查找根对象，来判断是否和根对象存在关联，但是老年代也存在根对象，老年代的根对象相对来讲是比较多的，一个个的对比是否存在引用，是非常消耗时间的，所以在建立引用的时候，就把老年代的对象标记为脏卡对象，查看GC ROOT存在的引用关系，我只需要去看脏卡的那一些对象即可



![image-20220823155928611](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823155928611.png)



![image-20220823160339362](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823160339362.png)



- 卡表与 Remembered Set【记忆集】 

- 在引用变更时通过 post-write barrier【写屏障，异步操作】 + dirty card queue 

- concurrent refinement threads 更新 Remembered Set



#### 5:remark



![image-20220823160544115](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220823160544115.png)



**以上代表并发标记阶段，黑色的代表处理完了，会被留下的对象，灰色代表正在处理，没有引用的白色最后会被处理为白色作为垃圾，将会被回收**

**存在的问题：因为在并发标记阶段的时候，垃圾回收线程和用户线程是并发执行的，可能在垃圾回收线程执行的时候，那个没引用的白色的被作为了垃圾，标记为了黑色，代表处理结束了，但是在之后的用户线程又添加了引用，在之后的并发清理的时候就不会再处理这个对象了，所以我们应该再进一步做检查**

**JVM会在对象引用发生改变的时候，就会触发写屏障，写屏障执行的时候，就会把这个将要改变引用的对象加入队列，表示还未处理完成，并发标记结束后，在重新标记的阶段，会STW，就会把队列的对象全部拿出来，做进一步检查**



#### 6:JDK 8u20 字符串去重

-XX:+UseStringDeduplication

优点：节省大量内存 

缺点：略微多占用了 cpu 时间，新生代回收时间略微增加

```java
String s1 = new String("hello"); // char[]{'h','e','l','l','o'}
String s2 = new String("hello"); // char[]{'h','e','l','l','o'}
```

- 将所有新分配的字符串放入一个队列 
- 当新生代回收时，G1并发检查是否有字符串重复 
- 如果它们值一样，让它们引用同一个 char[] 
- 注意，与 String.intern() 不一样 
  - String.intern() 关注的是字符串对象 
  - 而字符串去重关注的是 char[] 在 JVM 内部，使用了不同的字符串表



#### 7:JDK 8u40 并发标记类卸载

所有对象都经过并发标记后，就能知道哪些类不再被使用，当一个类加载器的所有类都不再使用

则卸载它所加载的所有类 

-XX:+ClassUnloadingWithConcurrentMark 默认启用



#### 8:JDK 8u60 回收巨型对象

- 一个对象大于 region 的一半时，称之为巨型对象 
- G1 不会对巨型对象进行拷贝 
- 回收时被优先考虑 
- G1 会跟踪老年代所有 incoming引用，这样老年代 incoming 引用为0 的巨型对象就可以在新生代垃圾回收时处理掉



#### 9:JDK 9 并发标记起始时间的调整

- 并发标记必须在堆空间占满前完成，否则退化为 FullGC【在以前是单线程，JDK高版本是多线程了】 

- JDK 9 之前需要使用 -XX:InitiatingHeapOccupancyPercent，在JDK 9 可以动态调整

  - -XX:InitiatingHeapOccupancyPercent 用来设置初始值
  - 进行数据采样并动态调整
  - 总会添加一个安全的空档空间

  

#### 10:G1的优点

* 并发与并行：
  * 并行性：G1 在回收期间，可以有多个 GC 线程同时工作，有效利用多核计算能力，此时用户线程 STW
  * 并发性：G1 拥有与应用程序交替执行的能力，部分工作可以和应用程序同时执行，因此不会在整个回收阶段发生完全阻塞应用程序的情况
  * 其他的垃圾收集器使用内置的 JVM 线程执行 GC 的多线程操作，而 G1 GC 可以采用应用线程承担后台运行的 GC 工作，JVM 的 GC 线程处理速度慢时，系统会**调用应用程序线程加速垃圾回收**过程
* **分区算法**：
  * 从分代上看，G1  属于分代型垃圾回收器，区分年轻代和老年代，年轻代依然有 Eden 区和 Survivor 区。从堆结构上看，**新生代和老年代不再物理隔离**，不用担心每个代内存是否足够，这种特性有利于程序长时间运行，分配大对象时不会因为无法找到连续内存空间而提前触发下一次 GC
  * 将整个堆划分成约 2048 个大小相同的独立 Region 块，每个 Region 块大小根据堆空间的实际大小而定，整体被控制在 1MB 到 32 MB之间且为 2 的 N 次幂，所有 Region 大小相同，在 JVM 生命周期内不会被改变。G1 把堆划分成多个大小相等的独立区域，使得每个小空间可以单独进行垃圾回收
  * **新的区域 Humongous**：本身属于老年代区，当出现了一个巨型对象超出了分区容量的一半，该对象就会进入到该区域。如果一个 H 区装不下一个巨型对象，那么 G1 会寻找连续的 H 分区来存储，为了能找到连续的 H 区，有时候不得不启动 Full GC
  * G1 不会对巨型对象进行拷贝，回收时被优先考虑，G1 会跟踪老年代所有 incoming 引用，这样老年代 incoming 引用为 0 的巨型对象就可以在新生代垃圾回收时处理掉



## 2.7:垃圾回收调优



### 1:调优领域 

- 内存 
- 锁竞争 
- cpu 
- 占用 io



### 2:确定目标 

【低延迟】还是【高吞吐量】，选择合适的回收器 

- CMS
- G1
- ZGC 
- ParallelGC Zing



### 3:最快的 GC 

答案是不发生 GC 

查看 FullGC 前后的内存占用，考虑下面几个问题 

- 数据是不是太多？ resultSet = statement.executeQuery("select * from 大表 limit n") 

- 数据表示是否太臃肿？ 

  - 对象图
  - 对象大小 new Object()-->最小都是16字节，例如包装类型  Integer 24字节  int 4字节 

- 是否存在内存泄漏？ static Map map =   然后不断的往map里面添加数据，就很容易GC或者Out of Memory

  可以采用下面的方式：

  - 软 
  - 弱 
  - 缓存的方式尽可能采用第三方缓存实现  redis等



### 4:新生代调优



新生代的特点 

- 所有的 new 操作的内存分配非常廉价 
- TLAB thread-local allocation buffer
- TLAB【每个线程在伊甸园的私有区域。线程的独立内存区域，不存在多个线程之间的内存冲突】 
- 死亡对象的回收代价是零 
- 大部分对象用过即死 
- Minor GC 的时间远远低于 Full GC



**问题：新生代的内存越大越好吗？**

**回答：不是，这样的话，老年代的内存就会相对更少了，这样就很容易触发Full GC，Full GC的话，停顿的时间会更长。所以并不是新生代越大越好。官方推荐在堆内存的25%~50%之间**

**经过测试，新生代的内存和吞吐量的曲线图，横坐标为内存，纵坐标为吞吐量，结果为先上升后下降**

**但是新生代的内存还是设置的相对大一点比较好，新生代之间采用复制算法，先标记，后复制**

**复制的时间比标记的时间长，但是在新生代的内存的对象大部分都是被回收，不需要复制的**

**所以大部分时间还是花费在标记上**



**新生代的各个区域设置多大合适呢？**

- 新生代能容纳所有【并发量 * (请求-响应)】的数据
- 幸存区大到能保留【当前活跃对象+需要晋升对象】
- 晋升阈值配置得当，让长时间存活对象尽快晋升 
- -XX:MaxTenuringThreshold=threshold【晋升的次数，默认是15】
- -XX:+PrintTenuringDistribution【垃圾回收时显示详情】





### 5:老年代调优



以CMS为例：

- CMS 的老年代内存越大越好 
- 先尝试不做调优，如果没有 Full GC，那么证明老年代比较正常，应该先尝试调优新生代 
- 观察发生到了Full GC，能缺点给是老年代内存占用，将老年代内存预设调大 1/4 ~ 1/3
  - -XX:CMSInitiatingOccupancyFraction=percent



### 6:GC调优案例



**案例一：Full GC 和 Minor GC频繁**

原因可能性分析：高峰期，大量用户进入，创建了大量的对象进入新生代区域，造成新生代紧张，也就会造成晋升阈值降低，大量的对象进入老年代，然后老年代也会频繁的进行Full GC，造成恶性循环

解决方案：试着增加新生代内存，也就增加了幸存区的内存，也就不容易造成晋升阈值降低而跑到老年代



**案例二：请求高峰期发生 Full GC，单次暂停时间特别长(CMS)**

原因可能性分析：我们知道在初始标记和并发标记的时间是比较短的，而重新标记的时间是比较长的，所以我们可以查看CMS的日志，很可能是因为在重新标记阶段【这个时间段，会扫描整个堆内存(新生代+老年代)】响应时间比较长

解决方案：我们可以尝试在重新标记的时候，把一些新生代的垃圾进行回收，这样就可以减少在重新标记阶段的时间，这样查找和对比的对象数量就会降低，就可以把响应时间提上去了  参数：XX:+CMSScavengeBeforeRemark



# 3:类加载和字节码技术



## 3.1:学习目标

- 类文件结构 
- 字节码指令
- 编译期处理
- 类加载阶段
- 类加载器 
- 运行期优化



![image-20220826084802744](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826084802744.png)

上图整体介绍：

写好的`java源文件`进行`javac编译`之后，成为`class文件`，然后经过类加载器被加载装入JVM的运行时数据区【运行时数据区也就是java内存结构】，然后通过执行引擎进行解释执行，对于一些热点代码，还会被执行引擎的`JIT`即使编译器进行优化处理



- 类加载器：用于装载字节码文件（.class文件）
- 运行时数据区：用于分配存储空间
- 执行引擎：执行字节码文件或本地方法
- 垃圾回收器：用于对 JVM 中的垃圾内容进行回收



## 3.2:类文件结构【了解】

```java
ClassFile {
	u4 				magic;						
    u2 				minor_version;						
    u2 				major_version;						
    u2 				constant_pool_count;
    cp_info			constant_pool[constant_pool_count-1];
    u2	 			access_flags;
    u2 				this_class;
    u2 				super_class;
    u2 				interfaces_count;
    u2 				interfaces[interfaces_count];
    u2 				fields_count;
    field_info 		fields[fields_count];
    u2 				methods_count;
    method_info 	methods[methods_count];
    u2 				attributes_count;
    attribute_info 	attributes[attributes_count];
}
```



| 类型           | 名称                | 说明                 | 长度    | 数量                  |
| -------------- | ------------------- | -------------------- | ------- | --------------------- |
| u4             | magic               | 魔数，识别类文件格式 | 4个字节 | 1                     |
| u2             | minor_version       | 副版本号(小版本)     | 2个字节 | 1                     |
| u2             | major_version       | 主版本号(大版本)     | 2个字节 | 1                     |
| u2             | constant_pool_count | 常量池计数器         | 2个字节 | 1                     |
| cp_info        | constant_pool       | 常量池表             | n个字节 | constant_pool_count-1 |
| u2             | access_flags        | 访问标识             | 2个字节 | 1                     |
| u2             | this_class          | 类索引               | 2个字节 | 1                     |
| u2             | super_class         | 父类索引             | 2个字节 | 1                     |
| u2             | interfaces_count    | 接口计数             | 2个字节 | 1                     |
| u2             | interfaces          | 接口索引集合         | 2个字节 | interfaces_count      |
| u2             | fields_count        | 字段计数器           | 2个字节 | 1                     |
| field_info     | fields              | 字段表               | n个字节 | fields_count          |
| u2             | methods_count       | 方法计数器           | 2个字节 | 1                     |
| method_info    | methods             | 方法表               | n个字节 | methods_count         |
| u2             | attributes_count    | 属性计数器           | 2个字节 | 1                     |
| attribute_info | attributes          | 属性表               | n个字节 | attributes_count      |



Class 文件格式采用一种类似于 C 语言结构体的方式进行数据存储，这种结构中只有两种数据类型：无符号数和表

* 无符号数属于基本的数据类型，以 u1、u2、u4、u8 来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照 UTF-8 编码构成字符串
* 表是由多个无符号数或者其他表作为数据项构成的复合数据类型，表都以 `_info` 结尾，用于描述有层次关系的数据，整个 Class 文件本质上就是一张表，由于表没有固定长度，所以通常会在其前面加上个数说明

获取方式：

* HelloWorld.java 执行 `javac -parameters -d . HellowWorld.java`指令
* 写入文件指令 `javap -v xxx.class >xxx.txt`
* IDEA 插件 jclasslib 



### 1:魔数

魔数：每个 Class 文件开头的 4 个字节的无符号整数称为魔数（Magic Number）是 Class 文件的标识符，

代表这是一个能被虚拟机接受的有效合法的 Class 文件，

* Java的class文件魔数值固定为 0xCAFEBABE【咖啡宝贝】不符合则会抛出错误

* 使用魔数而不是扩展名来进行识别主要是基于安全方面的考虑，因为文件扩展名可以随意地改动

版本：4 个字节，5 6两个字节代表的是编译的副版本号 minor_version，而 7 8 两个字节是编译的主版本号 major_version

* 不同版本的 Java 编译器编译的 Class 文件对应的版本是不一样的，高版本的 Java 虚拟机可以执行由低版本编译器生成的 Class 文件，反之 JVM 会抛出异常 `java.lang.UnsupportedClassVersionError`



### 2:常量池

常量池又叫做：`Class常量池`，在编译的时候就生成了这个常量池，常量池中常量的数量是不固定的

为什么不固定？因为后面涉及到的字符串信息还可能被加入到常量池中

constant_pool 是一种表结构，以1 ~ constant_pool_count - 1为索引

表明有多少个常量池表项。表项中`存放编译时期生成的各种字面量和符号引用`

这部分内容将在后面被`类加载后进入方法区的运行时常量池`



* 字面量（Literal） ：基本数据类型、字符串类型常量、声明为 final 的常量值等

* 符号引用（Symbolic References）：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符

  * 全限定名：com/test/Demo 这个就是类的全限定名，仅仅是把包名的 `.` 替换成 `/`，为了使连续的多个全限定名之间不产生混淆，在使用时最后一般会加入一个 `;` 表示全限定名结束

  * 简单名称：指没有类型和参数修饰的方法或者字段名称，比如字段 x 的简单名称就是 x

  * 描述符：用来描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值

  

| 标志符 | 含义                                                      |
| ------ | --------------------------------------------------------- |
| B      | 基本数据类型 byte                                         |
| C      | 基本数据类型 char                                         |
| D      | 基本数据类型 double                                       |
| F      | 基本数据类型 float                                        |
| I      | 基本数据类型 int                                          |
| J      | 基本数据类型 long                                         |
| S      | 基本数据类型 short                                        |
| Z      | 基本数据类型 boolean                                      |
| V      | 代表 void 类型                                            |
| L      | 对象类型，比如：`Ljava/lang/Object;`，不同方法间用`;`隔开 |
| [      | 数组类型，代表一维数组。比如：`double[][][] is [[[D`      |



常量类型和结构：

| 类型                             | 标志(或标识) | 描述                   |
| -------------------------------- | ------------ | ---------------------- |
| CONSTANT_utf8_info               | 1            | UTF-8编码的字符串      |
| CONSTANT_Integer_info            | 3            | 整型字面量             |
| CONSTANT_Float_info              | 4            | 浮点型字面量           |
| CONSTANT_Long_info               | 5            | 长整型字面量           |
| CONSTANT_Double_info             | 6            | 双精度浮点型字面量     |
| CONSTANT_Class_info              | 7            | 类或接口的符号引用     |
| CONSTANT_String_info             | 8            | 字符串类型字面量       |
| CONSTANT_Fieldref_info           | 9            | 字段的符号引用         |
| CONSTANT_Methodref_info          | 10           | 类中方法的符号引用     |
| CONSTANT_InterfaceMethodref_info | 11           | 接口中方法的符号引用   |
| CONSTANT_NameAndType_info        | 12           | 字段或方法的符号引用   |
| CONSTANT_MethodHandle_info       | 15           | 表示方法句柄           |
| CONSTANT_MethodType_info         | 16           | 标志方法类型           |
| CONSTANT_InvokeDynamic_info      | 18           | 表示一个动态方法调用点 |

18 种常量没有出现 byte、short、char，boolean 的原因：编译之后都可以理解为 Integer



### 3:访问标识

访问标识（access_flag），又叫访问标志、访问标记，该标识用两个字节表示，用于识别一些类或者接口层次的访问信息，包括这个 Class 是类还是接口，是否定义为 public类型，是否定义为 abstract类型等

* 类的访问权限通常为 ACC_ 开头的常量
* 每一种类型的表示都是通过设置访问标记的 32 位中的特定位来实现的，比如若是 public final 的类，则该标记为 `ACC_PUBLIC | ACC_FINAL`
* 使用 `ACC_SUPER` 可以让类更准确地定位到父类的方法，确定类或接口里面的 invokespecial 指令使用的是哪一种执行语义，现代编译器都会设置并且使用这个标记

| 标志名称       | 标志值 | 含义                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 标志为 public 类型                                           |
| ACC_FINAL      | 0x0010 | 标志被声明为 final，只有类可以设置                           |
| ACC_SUPER      | 0x0020 | 标志允许使用 invokespecial 字节码指令的新语义，JDK1.0.2之后编译出来的类的这个标志默认为真，使用增强的方法调用父类方法 |
| ACC_INTERFACE  | 0x0200 | 标志这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 是否为 abstract 类型，对于接口或者抽象类来说，次标志值为真，其他类型为假 |
| ACC_SYNTHETIC  | 0x1000 | 标志此类并非由用户代码产生（由编译器产生的类，没有源码对应） |
| ACC_ANNOTATION | 0x2000 | 标志这是一个注解                                             |
| ACC_ENUM       | 0x4000 | 标志这是一个枚举                                             |



### 4:索引集合

类索引、父类索引、接口索引集合

* 类索引用于确定这个类的全限定名

* 父类索引用于确定这个类的父类的全限定名，Java 语言不允许多重继承，所以父类索引只有一个，除了Object 之外，所有的 Java 类都有父类，因此除了 java.lang.Object 外，所有 Java 类的父类索引都不为0

* 接口索引集合就用来描述这个类实现了哪些接口
  * interfaces_count 项的值表示当前类或接口的直接超接口数量
  * interfaces[] 接口索引集合，被实现的接口将按 implements 语句后的接口顺序从左到右排列在接口索引集合中

| 长度 | 含义                         |
| ---- | ---------------------------- |
| u2   | this_class                   |
| u2   | super_class                  |
| u2   | interfaces_count             |
| u2   | interfaces[interfaces_count] |



### 5:字段表

字段 fields 用于描述接口或类中声明的变量，包括类变量以及实例变量，但不包括方法内部、代码块内部声明的局部变量以及从父类或父接口继承。字段叫什么名字、被定义为什么数据类型，都是无法固定的，只能引用常量池中的常量来描述

fields_count（字段计数器），表示当前 class 文件 fields 表的成员个数，用两个字节来表示

fields[]（字段表）：

* 表中的每个成员都是一个 fields_info 结构的数据项，用于表示当前类或接口中某个字段的完整描述

* 字段访问标识：

  | 标志名称      | 标志值 | 含义                       |
  | ------------- | ------ | -------------------------- |
  | ACC_PUBLIC    | 0x0001 | 字段是否为public           |
  | ACC_PRIVATE   | 0x0002 | 字段是否为private          |
  | ACC_PROTECTED | 0x0004 | 字段是否为protected        |
  | ACC_STATIC    | 0x0008 | 字段是否为static           |
  | ACC_FINAL     | 0x0010 | 字段是否为final            |
  | ACC_VOLATILE  | 0x0040 | 字段是否为volatile         |
  | ACC_TRANSTENT | 0x0080 | 字段是否为transient        |
  | ACC_SYNCHETIC | 0x1000 | 字段是否为由编译器自动产生 |
  | ACC_ENUM      | 0x4000 | 字段是否为enum             |

* 字段名索引：根据该值查询常量池中的指定索引项即可

* 描述符索引：用来描述字段的数据类型、方法的参数列表和返回值

  | 字符        | 类型      | 含义                    |
  | ----------- | --------- | ----------------------- |
  | B           | byte      | 有符号字节型树          |
  | C           | char      | Unicode字符，UTF-16编码 |
  | D           | double    | 双精度浮点数            |
  | F           | float     | 单精度浮点数            |
  | I           | int       | 整型数                  |
  | J           | long      | 长整数                  |
  | S           | short     | 有符号短整数            |
  | Z           | boolean   | 布尔值true/false        |
  | V           | void      | 代表void类型            |
  | L Classname | reference | 一个名为Classname的实例 |
  | [           | reference | 一个一维数组            |

* 属性表集合：属性个数存放在 attribute_count 中，属性具体内容存放在 attribute 数组中，一个字段还可能拥有一些属性，用于存储更多的额外信息，比如初始化值、一些注释信息等

  ```java
  ConstantValue_attribute{
      u2 attribute_name_index;
      u4 attribute_length;
      u2 constantvalue_index;
  }
  ```

  对于常量属性而言，attribute_length 值恒为2



### 6:方法表

方法表是 methods 指向常量池索引集合，其中每一个 method_info 项都对应着一个类或者接口中的方法信息，完整描述了每个方法的签名

* 如果这个方法不是抽象的或者不是 native 的，字节码中就会体现出来
* methods 表只描述当前类或接口中声明的方法，不包括从父类或父接口继承的方法
* methods 表可能会出现由编译器自动添加的方法，比如初始化方法 <cinit> 和实例化方法 <init>

**重载（Overload）**一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名，特征签名就是一个方法中各个参数在常量池中的字段符号引用的集合，因为返回值不会包含在特征签名之中，因此 Java 语言里无法仅仅依靠返回值的不同来对一个已有方法进行重载。但在 Class 文件格式中，特征签名的范围更大一些，只要描述符不是完全一致的两个方法就可以共存

methods_count（方法计数器）：表示 class 文件 methods 表的成员个数，使用两个字节来表示

methods[]（方法表）：每个表项都是一个 method_info 结构，表示当前类或接口中某个方法的完整描述

* 方法表结构如下：

  | 类型           | 名称             | 含义       | 数量             |
  | -------------- | ---------------- | ---------- | ---------------- |
  | u2             | access_flags     | 访问标志   | 1                |
  | u2             | name_index       | 字段名索引 | 1                |
  | u2             | descriptor_index | 描述符索引 | 1                |
  | u2             | attrubutes_count | 属性计数器 | 1                |
  | attribute_info | attributes       | 属性集合   | attributes_count |

* 方法表访问标志：

  | 标志名称      | 标志值 | 含义                       |
  | ------------- | ------ | -------------------------- |
  | ACC_PUBLIC    | 0x0001 | 字段是否为 public          |
  | ACC_PRIVATE   | 0x0002 | 字段是否为 private         |
  | ACC_PROTECTED | 0x0004 | 字段是否为 protected       |
  | ACC_STATIC    | 0x0008 | 字段是否为 static          |
  | ACC_FINAL     | 0x0010 | 字段是否为 final           |
  | ACC_VOLATILE  | 0x0040 | 字段是否为 volatile        |
  | ACC_TRANSTENT | 0x0080 | 字段是否为 transient       |
  | ACC_SYNCHETIC | 0x1000 | 字段是否为由编译器自动产生 |
  | ACC_ENUM      | 0x4000 | 字段是否为 enum            |



***



### 7:属性表

属性表集合，指的是 Class 文件所携带的辅助信息，比如该 Class 文件的源文件的名称，以及任何带有 `RetentionPolicy.CLASS` 或者 `RetentionPolicy.RUNTIME` 的注解，这类信息通常被用于 Java 虚拟机的验证和运行，以及 Java 程序的调试。字段表、方法表都可以有自己的属性表，用于描述某些场景专有的信息

attributes_ count（属性计数器）：表示当前文件属性表的成员个数

attributes[]（属性表）：属性表的每个项的值必须是 attribute_info 结构

* 属性的通用格式：

  ```java
  ConstantValue_attribute{
      u2 attribute_name_index;	//属性名索引
      u4 attribute_length;		//属性长度
      u2 attribute_info;			//属性表
  }
  ```

* 属性类型：

  | 属性名称                              | 使用位置           | 含义                                                         |
  | ------------------------------------- | ------------------ | ------------------------------------------------------------ |
  | Code                                  | 方法表             | Java 代码编译成的字节码指令                                  |
  | ConstantValue                         | 字段表             | final 关键字定义的常量池                                     |
  | Deprecated                            | 类、方法、字段表   | 被声明为 deprecated 的方法和字段                             |
  | Exceptions                            | 方法表             | 方法抛出的异常                                               |
  | EnclosingMethod                       | 类文件             | 仅当一个类为局部类或者匿名类是才能拥有这个属性，这个属性用于标识这个类所在的外围方法 |
  | InnerClass                            | 类文件             | 内部类列表                                                   |
  | LineNumberTable                       | Code 属性          | Java 源码的行号与字节码指令的对应关系                        |
  | LocalVariableTable                    | Code 属性          | 方法的局部变量描述                                           |
  | StackMapTable                         | Code 属性          | JDK1.6 中新增的属性，供新的类型检查检验器检查和处理目标方法的局部变量和操作数有所需要的类是否匹配 |
  | Signature                             | 类，方法表，字段表 | 用于支持泛型情况下的方法签名                                 |
  | SourceFile                            | 类文件             | 记录源文件名称                                               |
  | SourceDebugExtension                  | 类文件             | 用于存储额外的调试信息                                       |
  | Syothetic                             | 类，方法表，字段表 | 标志方法或字段为编泽器自动生成的                             |
  | LocalVariableTypeTable                | 类                 | 使用特征签名代替描述符，是为了引入泛型语法之后能描述泛型参数化类型而添加 |
  | RuntimeVisibleAnnotations             | 类，方法表，字段表 | 为动态注解提供支持                                           |
  | RuntimelnvisibleAnnotations           | 类，方法表，字段表 | 用于指明哪些注解是运行时不可见的                             |
  | RuntimeVisibleParameterAnnotation     | 方法表             | 作用与 RuntimeVisibleAnnotations 属性类似，只不过作用对象为方法 |
  | RuntirmelnvisibleParameterAnniotation | 方法表             | 作用与 RuntimelnvisibleAnnotations 属性类似，作用对象哪个为方法参数 |
  | AnnotationDefauit                     | 方法表             | 用于记录注解类元素的默认值                                   |
  | BootstrapMethods                      | 类文件             | 用于保存 invokeddynanic 指令引用的引导方式限定符             |



## 3.3:javap



自己分析字节码，实在是太痛苦，所以提供了javap指令



javap 反编译生成的字节码文件，根据 class 字节码文件，反解析出当前类对应的 code 区 （字节码指令）、局部变量表、异常表和代码行偏移量映射表、常量池等信息

用法：javap <options> <classes>

```sh
-help  --help  -?        输出此用法消息
-version                 版本信息
-public                  仅显示公共类和成员
-protected               显示受保护的/公共类和成员
-package                 显示程序包/受保护的/公共类和成员 (默认)
-p  -private             显示所有类和成员
						 #常用的以下三个
-v  -verbose             输出附加信息
-l                       输出行号和本地变量表
-c                       对代码进行反汇编	#反编译

-s                       输出内部类型签名
-sysinfo                 显示正在处理的类的系统信息 (路径, 大小, 日期, MD5 散列)
-constants               显示最终常量
-classpath <path>        指定查找用户类文件的位置
-cp <path>               指定查找用户类文件的位置
-bootclasspath <path>    覆盖引导类文件的位置
```



```java
package com.zzx;

/**
 * @author: 周志雄
 * @Description: javap
 * @date: 2022-08-26 8:56
 * @ClassName: TestHelloWorld
 */

public class TestJavaP {
    public static void main(String[] args) {
        System.out.println("Hello world");
    }
}
```



```apl
Classfile /E:/Code/JVM/out/production/class-structure/com/zzx/TestJavaP.class     类文件信息
  Last modified 2022-8-26; size 546 bytes       #修改时间，大小
  MD5 checksum 21b746b5d9a5388c38b9edae177546c7     #MD5签名
  Compiled from "TestJavaP.java"     #源文件名
  
public class com.zzx.TestJavaP  #全限定类名
  minor version: 0    #小版本
  major version: 52   #大版本  52代表JDK8
  flags: ACC_PUBLIC, ACC_SUPER        #public,增强的方法调用父类方法
  
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // Hello world
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // com/zzx/TestJavaP
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Lcom/zzx/TestJavaP;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               TestJavaP.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               Hello world
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               com/zzx/TestJavaP
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
  
{
  public com.zzx.TestJavaP();
    descriptor: ()V  	 #无参数，返回值Void
    flags: ACC_PUBLIC	 #public
    Code:
      stack=1, locals=1, args_size=1   #操作数栈深度1，局部变量表1
         0: aload_0       #把局部变量0加载到操作数栈
         1: invokespecial #调用方法                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:  #属性
        line 10: 0
      LocalVariableTable: #本地变量表
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/zzx/TestJavaP;

  public static void main(java.lang.String[]);  #main方法
    descriptor: ([Ljava/lang/String;)V  #String[]数组，返回值Void
    flags: ACC_PUBLIC, ACC_STATIC  #public static
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 12: 0
        line 13: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature  #作用范围0-9，参数args，string[]数组
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "TestJavaP.java"

```



## 3.4:图解javap

```java
package com.zzx;

/**
 * @author: 周志雄
 * @Description: Test
 * @date: 2022-08-26 9:30
 * @ClassName: Test01
 */

public class Test01 {
    public static void main(String[] args) {
        int a=10;
        int b=Short.MAX_VALUE+1;
        int c=a+b;
        System.out.println(c);
    }
}
```

使用javap反编译查看

```apl
Classfile /E:/Code/JVM/out/production/class-structure/com/zzx/Test01.class
  Last modified 2022-8-26; size 596 bytes
  MD5 checksum f220565addb605b7422f66343cdfbeda
  Compiled from "Test01.java"
public class com.zzx.Test01
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#25         // java/lang/Object."<init>":()V
   #2 = Class              #26            // java/lang/Short
   #3 = Integer            32768
   #4 = Fieldref           #27.#28        // java/lang/System.out:Ljava/io/PrintStream;
   #5 = Methodref          #29.#30        // java/io/PrintStream.println:(I)V
   #6 = Class              #31            // com/zzx/Test01
   #7 = Class              #32            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               LocalVariableTable
  #13 = Utf8               this
  #14 = Utf8               Lcom/zzx/Test01;
  #15 = Utf8               main
  #16 = Utf8               ([Ljava/lang/String;)V
  #17 = Utf8               args
  #18 = Utf8               [Ljava/lang/String;
  #19 = Utf8               a
  #20 = Utf8               I
  #21 = Utf8               b
  #22 = Utf8               c
  #23 = Utf8               SourceFile
  #24 = Utf8               Test01.java
  #25 = NameAndType        #8:#9          // "<init>":()V
  #26 = Utf8               java/lang/Short
  #27 = Class              #33            // java/lang/System
  #28 = NameAndType        #34:#35        // out:Ljava/io/PrintStream;
  #29 = Class              #36            // java/io/PrintStream
  #30 = NameAndType        #37:#38        // println:(I)V
  #31 = Utf8               com/zzx/Test01
  #32 = Utf8               java/lang/Object
  #33 = Utf8               java/lang/System
  #34 = Utf8               out
  #35 = Utf8               Ljava/io/PrintStream;
  #36 = Utf8               java/io/PrintStream
  #37 = Utf8               println
  #38 = Utf8               (I)V
{
  public com.zzx.Test01();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/zzx/Test01;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: bipush        10
         2: istore_1
         3: ldc           #3                  // int 32768
         5: istore_2
         6: iload_1
         7: iload_2
         8: iadd
         9: istore_3
        10: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
        13: iload_3
        14: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
        17: return
      LineNumberTable:
        line 12: 0
        line 13: 3
        line 14: 6
        line 15: 10
        line 16: 17
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      18     0  args   [Ljava/lang/String;
            3      15     1     a   I
            6      12     2     b   I
           10       8     3     c   I
}
SourceFile: "Test01.java"

```



### 1:常量池载入运行时常量池

仅仅列举了部分，注意：比较小的数字 [-1 ，5] 之间的数字是不会存在常量池的，会直接跟字节码绑定

![image-20220826093312880](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826093312880.png)



### 2:方法字节码载入方法区

![image-20220826093524015](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826093524015.png)



### 3:main线程运行

stack=2，locals=4，本地变量4个，栈的最大深度是2

![image-20220826093707260](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826093707260.png)





### 4:执行引擎开始执行字节码

bipush 10

- 将一个 byte 压入操作数栈（其长度会补齐 4 个字节）
- 类似的指令还有 sipush 将一个 short 压入操作数栈（其长度会补齐 4 个字节） 
- ldc 将一个 int 压入操作数栈 
- ldc2_w 将一个 long 压入操作数栈（分两次压入，因为 long 是 8 个字节） 
- 这里小的数字都是和字节码指令存在一起，如果数字超过 short 最大值的数字会存入了常量池

![image-20220826093848175](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826093848175.png)





istore_1 

- 将操作数栈顶数据弹出，存入局部变量表的 slot 1

![image-20220826093934553](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826093934553.png)



ldc #3 

- 从常量池加载 #3 数据到操作数栈 
- 注意 Short.MAX_VALUE 是 32767，所以 32768 = Short.MAX_VALUE + 1 实际是在编译期间计算好的

![image-20220826094008264](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094008264.png)





istore_2

![image-20220826094043640](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094043640.png)



iload_1

![image-20220826094105167](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094105167.png)



iload_2

![image-20220826094122511](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094122511.png)



iadd

![image-20220826094148201](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094148201.png)



istore_3

![image-20220826094217671](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094217671.png)



getstatic #4

![image-20220826094243857](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094243857.png)

![image-20220826094326460](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094326460.png)



iload_3

![image-20220826094353308](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094353308.png)



invokevirtual #5

- 找到常量池 #5 项 
- 定位到方法区 java/io/PrintStream.println:(I)V 方法 
- 生成新的栈帧（分配 locals、stack等） 
- 传递参数，执行新栈帧中的字节码

![image-20220826094456912](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094456912.png)



- 执行完毕，弹出栈帧 

- 清除 main 操作数栈内容



![image-20220826094526863](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220826094526863.png)



return 

- 完成 main 方法调用
- 弹出 main 栈帧 程序结束



### 5:小结

- bipush：将byte数字压入操作数栈
- istore：将操作数栈的数据弹出，再存入局部变量表，这个时候操作数栈没有这个数据了
- ldc：从常量池加载数据到操作数栈，注意看到后面有个#，代表去运行常量池找
- iload：把局部变量表的数据复制一份，加载到操作数栈，操作数栈和局部变量表都有这个数据
- iadd：把操作数栈前两个数据相加
- 当数字在 -1到5之间的时候，数字直接和字节码绑定
- 当数字超过short的最大范围 32767 的时候，则数字会存入常量池里面



## 3.5:字节码分析a++和++a



```java
package com.zzx;

/**
 * @author: 周志雄
 * @Description: Test
 * @date: 2022-08-26 9:30
 * @ClassName: Test01
 */

public class Test01 {
    public static void main(String[] args) {
        int a=10;
        int b=a++;
        int c=++a;

        System.out.println(b);
        System.out.println(c);
    }
}
```



**着重看这里的字节码部分：**

```apl
stack=2, locals=4, args_size=1
         0: bipush        10
         2: istore_1
         3: iload_1
         4: iinc          1, 1
         7: istore_2
         8: iinc          1, 1
        11: iload_1
        12: istore_3
        13: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        16: iload_2
        17: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        20: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        23: iload_3
        24: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        27: return
```



**注意：iinc的操作是在局部变量表上进行，不是在操作数栈运行**



**发现a++是先执行iload,再执行iinc**



**++a是先执行iinc，再执行iload**



## 3.6:练习-判断结果

```java
public class Test01 {
    public static void main(String[] args) {
        int i = 0;
        int x = 0;
        while (i < 10) {
            x = x++;
            i++;
        }
        System.out.println(x); // 结果是 0
    }
}
```

数字比较小的时候，使用的`iconst`指令

```apl
Code:
      stack=2, locals=3, args_size=1
         0: iconst_0
         1: istore_1
         
         2: iconst_0
         3: istore_2
         
         4: iload_1
         5: bipush        10
         
         7: if_icmpge     21
         
        10: iload_2
        11: iinc          2, 1
        
        14: istore_2
        15: iinc          1, 1
        18: goto          4
        21: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
        24: iload_2
        25: invokevirtual #3                  // Method java/io/PrintStream.println:(I)V
        28: return

```

**注意：inic是在局部变量表上进行，不是在操作数栈运行**



## 3.7:构造方法

### 1:cinit



cinit是整个类的构造方法，主要是执行静态代码块

```java
package com.zzx;

/**
 * @author: 周志雄
 * @Description: Test
 * @date: 2022-08-26 9:30
 * @ClassName: Test01
 */

public class Test01 {
    static int i = 10;
    static {
        i = 20;
    }
    static {
        i = 30;
    }
    public static void main(String[] args) {
    }
}
```



编译器会按`从上至下`的顺序，收集`所有 static 静态代码块和静态成员赋值`的代码

然后把上面涉及的所有代码合并为一个特殊的方法`<cinit>()V`

(cinit)方法会在`类加载的初始化阶段`被调用

```apl
Code:
      stack=1, locals=0, args_size=0
         0: bipush        10
         2: putstatic     #2                  // Field i:I
         5: bipush        20
         7: putstatic     #2                  // Field i:I
        10: bipush        30
        12: putstatic     #2                  // Field i:I
        15: return
```

**putstatic就是给静态变量赋值的意思**

**所以输出的话，答案为30**



### 2:init

cinit是整个类的构造方法，主要是执行静态代码块

init就是每个实例的构造方法



```java
package com.zzx;

/**
 * @author: 周志雄
 * @Description: Test
 * @date: 2022-08-26 9:30
 * @ClassName: Test01
 */

public class Test01 {
    private String a = "s1";
    {
        b = 20;
    }
    private int b = 10;
    {
        a = "s2";
    }
    public Test01(String a, int b) {
        this.a = a;
        this.b = b;
    }

    public static void main(String[] args) {
        Test01 d=new Test01("s3",30);
        System.out.println(d.a);
        System.out.println(d.b);
    }
}
```



编译器会按`从上至下`的顺序，收集所有 `{} 代码块和成员变量赋值`的代码，`形成新的构造方法`

但`原始构造方法内的代码总是在最后`，我们自己的构造方法权重更大



```apl
Code:
      stack=4, locals=2, args_size=1
         0: new           #6                  // class com/zzx/Test01
         3: dup
         4: ldc           #7                  // String s3
         6: bipush        30
         8: invokespecial #8                  // Method "<init>":(Ljava/lang/String;I)V
        11: astore_1
        12: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
        15: aload_1
        16: getfield      #3                  // Field a:Ljava/lang/String;
        19: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        22: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
        25: aload_1
        26: getfield      #4                  // Field b:I
        29: invokevirtual #11                 // Method java/io/PrintStream.println:(I)V
        32: return
```



所以：执行Test01 d=new Test01("s3"，30)`并不是直接就执行构造方法`了，而是内部组成了一个新的构造方法

先执行了这个`由 {} 代码块和为成员变量赋值的代码块的构造方法`，再执行我们的构造方法

- a="s1"
- b=20
- b=10
- a="s2"
- 最后才是我们的构造方法执行



注意：上面出现了`dup`的字节码，解释一下这个

当我们new对象的时候，我们就在堆内存里面创建了一个对象，我们这个例子是`Test01`的对象，并且把堆

内存的这个对象的地址压入了操作数栈，`dup`代表把这个栈顶的数据进行复制操作，这里也就是存在了两

个`Test01`对象的引用，这两个对象引用是完全一样的，我们知道会执行构造方法，这个时候会消耗一个对

象引用，这个时候还剩下一个，我们的代码是`Test01 d=new Test01("s3",30)`，还剩下一个从操作数栈到

局部变量表的`xstore`操作，剩下的这个对象引用就是执行这个的，这就是`dup`的作用



## 3.8:方法调用

```java
public class Demo3_9 {
	public Demo3_9() { }
    //私有方法
	private void test1() { }
    //final修饰的
	private final void test2() { }
    //普通成员方法
	public void test3() { }
    //静态方法
	public static void test4() { }
    
    //主方法
	public static void main(String[] args) {
		Demo3_9 d = new Demo3_9();
		d.test1();
		d.test2();
		d.test3();
		d.test4();
		Demo3_9.test4();
	}
}
```



```apl
0: new #2 // class cn/itcast/jvm/t3/bytecode/Demo3_9
3: dup
4: invokespecial #3 // Method "<init>":()V
7: astore_1
8: aload_1
9: invokespecial #4 // Method test1:()V
12: aload_1
13: invokespecial #5 // Method test2:()V
16: aload_1
17: invokevirtual #6 // Method test3:()V
20: aload_1
21: pop
22: invokestatic #7 // Method test4:()V
25: invokestatic #7 // Method test4:()V
28: return
```



**注意：**

- invokespecial可以`直接确定`绑定了哪个方法，静态的，private的,final的，效率更高，不需要多次查找

- invokevirtual在编译的时候，不能确定是调用哪个类的哪个方法，可能是重写的父类的方法，可能是父类的，不确定，`会涉及到对象头的查找`，所以效率会差点

- new 是创建【对象】，给对象分配堆内存，执行成功会将【对象引用】压入操作数栈

- dup 是复制操作数栈栈顶的内容，本例即为【对象引用】，为什么需要两份引用呢，一个是要配合 invokespecial 调用该对象的构造方法会消耗掉栈顶一个引用，另一个要配合 astore_1 赋值给局部变量

- 最终方法（final）私有方法（private）构造方法都是由 `invokespecial` 指令来调用，属于静态绑定

- 普通成员方法是由 `invokevirtual` 调用，属于动态绑定，即支持多态

- 成员方法与静态方法调用的另一个区别是，执行方法前`是否需要【对象引用】`

- 比较有意思的是 d.test4() 是通过【对象引用】调用一个静态方法，可以看到在调用 `invokestatic 之前执

  行了 pop 指令，也就是上面的20，21行字节码，把【对象引用】从操作数栈弹掉了，因为不需要对象

- 还有一个执行 invokespecial 的情况是通过 super 调用父类方法



**所以执行静态方法，不要使用对象.静态方法，因为使用对象来执行静态方法，会把对象入栈，但是又会被弹出去，实际上还多出了两条不必要的指令，效率是没有`类.静态方法`高的**



## 3.9:多态原理



### 1:执行原理

Java 虚拟机中关于方法重写的判定基于方法描述符，如果子类定义了与父类中非私有、非静态方法同名的方法，只有当这两个方法的参数类型以及返回类型一致，Java 虚拟机才会判定为重写

理解多态：

- 多态有编译时多态和运行时多态，即静态绑定和动态绑定
- 前者是通过方法重载实现，后者是通过重写实现（子类覆盖父类方法，虚方法表）
- 虚方法：运行时动态绑定的方法，对比静态绑定的非虚方法调用来说，虚方法调用更加耗时

方法重写的本质：

1. 找到操作数栈的第一个元素**所执行的对象的实际类型**，记作 C

2. 如果在类型 C 中找到与描述符和名称都相符的方法，则进行访问**权限校验**（私有的），如果通过则返回这个方法的直接引用，查找过程结束；如果不通过，则返回 java.lang.IllegalAccessError 异常

   IllegalAccessError：表示程序试图访问或修改一个属性或调用一个方法，这个属性或方法没有权限访问，一般会引起编译器异常。如果这个错误发生在运行时，就说明一个类发生了不兼容的改变

3. 找不到，就会按照继承关系从下往上依次对 C 的各个父类进行第二步的搜索和验证过程

4. 如果始终没有找到合适的方法，则抛出 java.lang.AbstractMethodError 异常



### 2:虚方法表

在虚拟机工作过程中会频繁使用到动态绑定，每次动态绑定的过程中都要重新在类的元数据中搜索合适目标，影响到执行效率。为了提高性能，JVM 采取了一种用**空间换取时间**的策略来实现动态绑定，在每个**类的方法区**建立一个虚方法表（virtual method table），实现使用索引表来代替查找，可以快速定位目标方法

* invokevirtual 所使用的虚方法表（virtual method table，vtable），执行流程
  1. 先通过栈帧中的对象引用找到对象，分析对象头，找到对象的实际 Class
  2. Class 结构中有 vtable，查表得到方法的具体地址，执行方法的字节码
* invokeinterface 所使用的接口方法表（interface method table，itable）

虚方法表会在类加载的链接阶段被创建并开始初始化，类的变量初始值准备完成之后，JVM 会把该类的方法表也初始化完毕

虚方法表的执行过程：

* 对于静态绑定的方法调用而言，实际引用将指向具体的目标方法
* 对于动态绑定的方法调用而言，实际引用则是方法表的索引值，也就是方法的间接地址。Java 虚拟机获取调用者的实际类型，并在该实际类型的虚方法表中，根据索引值获得目标方法内存偏移量（指针）

为了优化对象调用方法的速度，方法区的类型信息会增加一个指针，该指针指向一个记录该类方法的方法表。每个类中都有一个虚方法表，本质上是一个数组，每个数组元素指向一个当前类及其祖先类中非私有的实例方法

方法表满足以下的特质：

* 其一，子类方法表中包含父类方法表中的**所有方法**，并且在方法表中的索引值与父类方法表种的索引值相同
* 其二，**非重写的方法指向父类的方法表项，与父类共享一个方法表项，重写的方法指向本身自己的实现**。所以这就是为什么多态情况下可以访问父类的方法。

### 3:内联缓存

内联缓存：是一种加快动态绑定的优化技术，能够缓存虚方法调用中**调用者的动态类型以及该类型所对应的目标方法**。在之后的执行过程中，如果碰到已缓存的类型，便会直接调用该类型所对应的目标方法；反之内联缓存则会退化至使用基于方法表的动态绑定

多态的三个术语：

* 单态 (monomorphic)：指的是仅有一种状态的情况
* 多态 (polymorphic)：指的是有限数量种状态的情况，二态（bimorphic）是多态的其中一种
* 超多态 (megamorphic)：指的是更多种状态的情况，通常用一个具体数值来区分多态和超多态，在这个数值之下，称之为多态，否则称之为超多态

对于内联缓存来说，有对应的单态内联缓存、多态内联缓存：

* 单态内联缓存：只缓存了一种动态类型以及所对应的目标方法，实现简单，比较所缓存的动态类型，如果命中，则直接调用对应的目标方法。
* 多态内联缓存：缓存了多个动态类型及其目标方法，需要逐个将所缓存的动态类型与当前动态类型进行比较，如果命中，则调用对应的目标方法

为了节省内存空间，**Java 虚拟机只采用单态内联缓存**，没有命中的处理方法：

* 替换单态内联缓存中的纪录，类似于 CPU 中的数据缓存，对数据的局部性有要求，即在替换内联缓存之后的一段时间内，方法调用的调用者的动态类型应当保持一致，从而能够有效地利用内联缓存
* 劣化为超多态状态，这也是 Java 虚拟机的具体实现方式，这种状态实际上放弃了优化的机会，将直接访问方法表来动态绑定目标方法，但是与替换内联缓存纪录相比节省了写缓存的额外开销

虽然内联缓存附带内联二字，但是并没有内联目标方法



### 4:总结

当执行 invokevirtual 指令时

- 先通过栈帧中的对象引用找到对象
- 分析对象头，找到对象的实际 Class 
- Class 结构中有 vtable，它在类加载的链接阶段就已经根据方法的重写规则生成好了 
- 查表得到方法的具体地址 
- 执行方法的字节码



面试回答：执行`invokevirtual`，首先是通过栈帧的对象引用来找到对象，通过找到的对象来分析对象的

对象头，从而得到实际的Class信息，因为Class信息里面包含了`vtable`也就是虚方法表，虚方法表再类加

载的时候就已经确定了重写规则，就能够找到方法的具体地址，从而执行方法的字节码



## 3.10:异常处理



### 1:单个try catch

```java
public class Demo3_11_1 {
	public static void main(String[] args) {
	int i = 0;
		try {
			i = 10;
		} catch (Exception e) {
			i = 20;
		}
	}
}
```

```apl
public static void main(java.lang.String[]);
	descriptor: ([Ljava/lang/String;)V
	flags: ACC_PUBLIC, ACC_STATIC
	Code:
	stack=1, locals=3, args_size=1
	0: iconst_0
	1: istore_1
	2: bipush 10
	4: istore_1
	5: goto 12
	8: astore_2
	9: bipush 20
	11: istore_1
	12: return
	
	Exception table:
	from  to    target    type
	2     5       8       Class java/lang/Exception
	
	LineNumberTable: ...
	LocalVariableTable:
	Start Length Slot Name Signature
	9       3      2    e   Ljava/lang/Exception;
	0       13     0   args [Ljava/lang/String;
	2       11 	   1    i   I
	StackMapTable: ...
	MethodParameters: ...
}
```

- 可以看到多出来一个 `Exception table` 的结构，[from, to) 是前闭后开的检测范围，一旦这个范围内的字节码执行出现异常，则通过 type 匹配异常类型，如果一致，进入 target 所指示行号 
- 8 行的字节码指令 astore_2 是将异常对象引用存入局部变量表的 slot 2 位置，也就是说异常对象也会占用一个局部变量表的位置



### 2:多个try catch

```java
public static void main(String[] args) {
	int i = 0;
	try {
		i = 10;
	} catch (ArithmeticException e) {
		i = 30;
	} catch (NullPointerException e) {
		i = 40;
	} catch (Exception e) {
		i = 50;
	}
}
```



![image-20220828175419045](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220828175419045.png)



**因为异常出现时,只能进入Exception table中一个分支，所以局部变量表slot2位置被共用，不需要多个slot**



### 3:multi-catch

```java
public static void main(String[] args) {
	try {
		Method test = Demo3_11_3.class.getMethod("test");
		test.invoke(null);
	} catch (NoSuchMethodException | IllegalAccessException |InvocationTargetException e) {
		e.printStackTrace();
	}
}
```



![image-20220828175634835](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220828175634835.png)

**没什么区别，也就是target一样，其他的跟多个try..catch一样**



## 3.11:finally

```java
public class Demo3_11_4 {
	public static void main(String[] args) {
		int i = 0;
		try {
			i = 10;
		} catch (Exception e) {
			i = 20;
		} finally {
			i = 30;
		}
	}
}
```



![image-20220828180051166](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220828180051166.png)

![image-20220828180119264](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220828180119264.png)



**所以即使是catch捕获不到的exception，也会执行finally，会执行到catch any来执行final代码**



final代码块被放入了哪些位置？

可以看到 finally 中的代码被复制了 3 份

分别放入 

- try 流程
- catch 流程
- catch 剩余的异常类型流程



分析字节码，发现在`try,catch,catch剩余部分`都会执行 finally 内部的字节码



## 3.12:finally面试题



### 1:finally出现了return

```java
public static void main(String[] args) {
	int result = test();
	System.out.println(result);
}
public static int test() {
	try {
		return 10;
	} finally {
		return 20;
	}
}
```



**结果：20**



![image-20220828213418162](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220828213418162.png)



- 由于 finally 中的 ireturn 被插入了所有可能的流程，因此返回结果肯定以 finally 的为准 

- 至于字节码中第 2 行，似乎没啥用，且留个伏笔，看下个例子 

- 跟上例中的 finally 相比，发现存在了异常表，但是字节码并没有任何的 `athrow` 了，实际上：如

  果在 finally 中出现了 return，会吞掉异常信息，即代码出现异常了，永远捕获不到



**总结就是：在finally里面写了retuen，即使你的代码出现了异常，都不会抛出异常，会被吞掉异常**

**还会误以为是正常执行的代码**



### 2:finally对返回值的影响

```java
public class Test {
	public static void main(String[] args) {
		int result = test();
		System.out.println(result);
	}
	public static int test() {
		int i = 10;
		try {
			return i;
		} finally {
			i = 20;
	}
}
```



![image-20220828213914605](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220828213914605.png)

我们发现

1：当`finally里没有return`，但是`try语句块里面出现了return`，在try执行的时候，会把数据复制到两个槽位，因为一个槽位的数据可能在finally代码块被改动，所以会复制到一个新的槽位，来固定返回值，防止修改返回值

2：当`finally里存在return`，同时`try语句块也出现了return`，在try执行的时候，就不会固定我们的返回

值，因为finally存在了return，返回的结果肯定以finally为准了，所以没必要固定返回值



## 3.13:synchronized

```java
public static void main(String[] args) {
	Object lock = new Object();
	synchronized (lock) {
		System.out.println("ok");
	}
}
```



![image-20220829152218114](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220829152218114.png)

![image-20220829152232982](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220829152232982.png)



**方法级别的 synchronized 不会在字节码指令中有所体现**



## 3.14:编译期处理[语法糖]

```
所谓的语法糖 ，其实就是指 java 编译器把 *.java 源码编译为 *.class 字节码的过程中
自动生成和转换的一些代码，就是一个优化的手段
主要是为了减轻程序员的负担，算是 java 编译器给我们的一个额外福利（给糖吃）

以下代码的分析，借助了 javap 工具，idea 的反编译功能，idea 插件 jclasslib 等工具。另外， 编译器转换的结果直接就是 class 字节码，只是为了便于阅读，给出了 几乎等价 的 java 源码方式，并不是编译器还会转换出中间的 java 源码
```



### 1:默认构造器

```java
public class Candy1 {
}
```

编译成class后的代码：

```java
public class Candy1 {
	// 这个无参构造是编译器帮助我们加上的
	public Candy1() {
		super(); // 即调用父类Object的无参构造方法，即调用 java/lang/Object."<init>":()V
	}
}
```



### 2:自动拆装箱

这个特性是 JDK 5 开始加入的

```java
public class Candy2 {
	public static void main(String[] args) {
		Integer x = 1;
		int y = x;
	}
}
```

这段代码在 JDK 5 之前是无法编译通过的

```java
public class Candy2 {
	public static void main(String[] args) {
		Integer x = Integer.valueOf(1);
		int y = x.intValue();
	}
}
```



### 3:泛型集合取值

泛型也是在 JDK 5 开始加入的特性，但 java 在编译泛型代码后会执行 `泛型擦除` 的动作

即泛型信息 在编译为字节码之后就丢失了，实际的类型都当做了 Object 类型来处理：

```java
public class Candy3 {
	public static void main(String[] args) {
		List<Integer> list = new ArrayList<>();
		list.add(10); // 实际调用的是 List.add(Object e)
		Integer x = list.get(0); // 实际调用的是 Object obj = List.get(int index);
	}
}
```

**什么是泛型擦除：在编译的字节码里面，操作的实际上都是Object，而不是对应的泛型**



**对应关键字节码如下**

![image-20220829161907539](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220829161907539.png)



**checkcast会自动检查并转换**



![image-20221114194456929](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20221114194456929.png)



擦除的是`字节码上的泛型信息`，也就是`字节码存储到方法区的信息都是Object类型`，但是

LocalVariableTypeTable仍然保留了方法参数泛型的信息

有泛型的，会生成LocalVariableTypeTable表

![image-20220829162208251](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220829162208251.png)



### 4:可变参数

可变参数也是 JDK 5 开始加入的新特性

```java
public class Candy4 {
	public static void foo(String... args) {
		String[] array = args; // 直接赋值
		System.out.println(array);
	}
	public static void main(String[] args) {
		foo("hello", "world");
	}
}

```

**String... 参数最后会成为数组**

注意:如果调用了foo(),则等价代码为 foo(new String[]{}) 

创建了一个空的数组，而不是传递null进去



### 5:foreach循环

```java
public class Candy5_1 {
	public static void main(String[] args) {
		int[] array = {1, 2, 3, 4, 5}; // 数组赋初值的简化写法也是语法糖
		for (int e : array) {
			System.out.println(e);
		}
	}
}
```



**会编译为：**

```java
public class Candy5_1 {
	public Candy5_1() {
	}
	public static void main(String[] args) {
		int[] array = new int[]{1, 2, 3, 4, 5};
		for(int i = 0; i < array.length; ++i) {
			int e = array[i];
			System.out.println(e);
		}
	}
}
```



**集合的循环**

```java
public class Candy5_2 {
	public static void main(String[] args) {
		List<Integer> list = Arrays.asList(1,2,3,4,5);
		for (Integer i : list) {
			System.out.println(i);
		}
	}
}
```

**实际被编译器转换为对迭代器的调用：**

```java
public class Candy5_2 {
	public Candy5_2() {
	}
	public static void main(String[] args) {
		List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
		Iterator iter = list.iterator();
		while(iter.hasNext()) {
			Integer e = (Integer)iter.next();
			System.out.println(e);
		}
	}
}
```



注意：foreach 循环写法，能够配合`数组`，以及所有`实现了 Iterable 接口的集合类`一起使用，其中

Iterable 用来获取集合的迭代器（ Iterator ）



### 6:switch字符串

在原生的switch，是不支持字符串这些的

从 JDK 7 开始，switch 可以作用于字符串和枚举类，这个功能其实也是语法糖

```java
public class Candy6_1 {
	public static void choose(String str) {
		switch (str) {
			case "hello": {
			System.out.println("h");
			break;
			}
			case "world": {
				System.out.println("w");
				break;
			}
		}
	}
}
```



**编译后的：**

```java
public class Candy6_1 {
	public Candy6_1() {
	}
	public static void choose(String str) {
		byte x = -1;
		switch(str.hashCode()) {
			case 99162322: // hello 的 hashCode
			if (str.equals("hello")) {
				x = 0;
			}
			break;
			case 113318802: // world 的 hashCode
			if (str.equals("world")) {
				x = 1;
			}
		}
		switch(x) {
			case 0:
			System.out.println("h");
			break;
			case 1:
			System.out.println("w");
		}
	}
}

```



```
可以看到，执行了两遍 switch
第一遍是根据字符串的 hashCode 和 equals 将字符串的转换为相应byte 类型
第二遍才是利用 byte 执行进行比较。
为什么第一遍时必须既比较 hashCode，又利用 equals 比较呢？
hashCode 是为了提高效率，减少可能的比较；而 equals 是为了防止 hashCode 冲突
```



**例如 BM 和 C. 这两个字符串的hashCode值都是 2123**

**Hashcode相同的时候，还会再执行一次判断**

```
case 2123: //hashCode 值可能相同，需要进一步用 equals 比较
if (str.equals("C.")) {
	x = 1;
} else if (str.equals("BM")) {
	x = 0;
}
```



**小结：**

**字符串的switch实际上是拆分成两个整数的switch分支**

**第一个Switch：**

1：比较我们传入的String的HashCode和Case的字符串的HashCode，来得到对应的整数值

2：如果出现了Hash碰撞，则再比较equals()，来得到对应的整数值

**第二个Switch：**对这个第一个Switch得到的整数值进行再进行switch来判断执行哪个分支



### 7:switch枚举

```java
enum Sex {
	MALE, FEMALE
}
```



```java
public class Candy7 {
	public static void foo(Sex sex) {
		switch (sex) {
		case MALE:
			System.out.println("男"); break;
		case FEMALE:
			System.out.println("女"); break;
		}
	}
}
```

**转换后会生成一个静态内部类**

```java
/**
* 生成一个合成类（仅 jvm 使用，对我们不可见）
* 用来映射枚举的 ordinal 与数组元素的关系
* 枚举的 ordinal 表示枚举对象的序号，从 0 开始
* 即 MALE 的 ordinal()=0，FEMALE 的 ordinal()=1
*/
static class $MAP {
	// 数组大小即为枚举元素个数，里面存储case用来对比的数字
	static int[] map = new int[2];
	static {
		map[Sex.MALE.ordinal()] = 1;
		map[Sex.FEMALE.ordinal()] = 2;
	}
}

public static void foo(Sex sex) {
	int x = $MAP.map[sex.ordinal()];
	switch (x) {
		case 1:
			System.out.println("男");
			break;
		case 2:
			System.out.println("女");
			break;
		}
	}
}
```



### 8:枚举类

JDK 7 新增了枚举类

```java
enum Sex {
	MALE, FEMALE
}
```

**枚举类的值本质上就是一个Class【Sex】**

**MALE FEMALE就是Class【Sex】的两个实例对象**

**普通类的实例个数是无限的，可以随时new，但是枚举的是有限的**



转换后代码：

```java
public final class Sex extends Enum<Sex> {
	public static final Sex MALE;
	public static final Sex FEMALE;
	private static final Sex[] $VALUES;
    
	static {
		MALE = new Sex("MALE", 0);
		FEMALE = new Sex("FEMALE", 1);
		$VALUES = new Sex[]{MALE, FEMALE};
	}
	private Sex(String name, int ordinal) {
		super(name, ordinal);
	}
	public static Sex[] values() {
		return $VALUES.clone();
	}
	public static Sex valueOf(String name) {
		return Enum.valueOf(Sex.class, name);
	}
}
```

**被final修饰，不能被继承，枚举的值实际上就是一个对象**



### 9:try-with-resources

JDK 7 开始新增了对需要关闭的资源处理的特殊语法 try-with-resources

以前的话，要自己加上finally来确定资源一定被关闭

其实我们只要按照它的规则来写，就可以不写finally

```java
try(资源变量 = 创建资源对象){
} catch( ) {
}
```

其中`资源对象需要实现 AutoCloseable 接口`，例如 InputStream 、 OutputStream  Connection 、 Statement  ResultSet 等接口都实现了 AutoCloseable，那么就可以使用 try-with-resources 可以不用写 finally 语句块，编译器会帮助生成关闭资源代码

```java
public class Candy9 {
	public static void main(String[] args) {
		try(InputStream is = new FileInputStream("d:\\my.txt")) {
			System.out.println(is);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```



**编译后:**

```java
public class Candy9 {
	public Candy9() {
	}
	public static void main(String[] args) {
		try {
			InputStream is = new FileInputStream("d:\\my.txt");
			System.out.println(is);
		} catch (Throwable e1) {
			// t 是我们代码出现的异常
			t = e1;
			throw e1;
		} finally {
			// 判断了资源不为空
			if (is != null) {
				// 如果我们代码有异常
				if (t != null) {
						try {
							is.close();
						} catch (Throwable e2) {
							// 如果 close 出现异常，作为被压制异常添加
							t.addSuppressed(e2);
						}
					} else {
						// 如果代码没有异常,close 出现的异常就是最后 catch 块中的 e
						is.close();
					}
				}
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}
```

**仔细读源码之后，就会发现他已经自动添加了对资源的判断，无论如何对会执行资源的关闭**

```apl
为什么要设计一个 addSuppressed(Throwable e) （添加被压制异常）的方法呢？
是为了防止异常信息的丢失（想想 try-with-resources 生成的 fianlly 中如果抛出了异常)
```

**看下面的实例:**

```java
public class Test6 {
	public static void main(String[] args) {
		try (MyResource resource = new MyResource()) {
			int i = 1/0;
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
class MyResource implements AutoCloseable {
	public void close() throws Exception {
		throw new Exception("close 异常");
	}
}
```



try-with-resources会保留程序的异常和关闭资源出现的异常，都可以被捕获

```
java.lang.ArithmeticException: / by zero
at test.Test6.main(Test6.java:7)
Suppressed: java.lang.Exception: close 异常
at test.MyResource.close(Test6.java:18)
at test.Test6.main(Test6.java:6)
```



### 10:方法重写时的桥接方法

我们都知道，方法重写时对返回值分两种情况： 

- 父子类的返回值完全一致 
- 子类返回值可以是父类返回值的子类

严格来说：其实只有返回值完全一致的时候才算是重写

**如下所示:**

```java
class A {
	public Number m() {
		return 1;
	}
}

class B extends A {
	@Override
	// 子类 m 方法的返回值是 Integer 是父类 m 方法返回值 Number 的子类
	public Integer m() {
		return 2;
	}
}
```



**为什么返回值不一致，也算重写呢？其实也是java编译器做的优化**

**对于子类，java 编译器会做如下处理：**

**会生成一个新的真正意义上的重写方法【合成方法，也叫做桥接方法】**

**在这个新的方法的内部再调用我们写的方法返回父类方法返回值的子类，再向上转型得到返回值**



```java
class B extends A {
	public Integer m() {
		return 2;
	}
	// 此方法才是真正重写了父类 public Number m() 方法
	public synthetic bridge Number m() {
		// 调用 public Integer m()
		return m();
	}
}
```



### 11:匿名内部类

```java
public class Candy11{
    public static void main(String[] args){	
		Runnable runnable = new Runnable() {
			@Override
			public void run() {
				System.out.println("ok");
			}
		};
	}
}
```

**转换后代码：**

```java
// 额外生成的类
final class Candy11$1 implements Runnable {
	Candy11$1() {
	}
	
	public void run() {
		System.out.println("ok");
	}
}

```

**会生成一个新的类实现Runnable接口**

```java
public class Candy11 {
	public static void main(String[] args) {
		Runnable runnable = new Candy11$1();
	}
}
```



**引用局部变量的匿名内部类:源码**

```java
public class Candy11 {
	public static void test(final int x) {
		Runnable runnable = new Runnable() {
			@Override
			public void run() {
				System.out.println("ok:" + x);
			}
		};
	}
}
```



**编译生成的**

```java
// 额外生成的类
final class Candy11$1 implements Runnable {
	int val$x;
	Candy11$1(int x) {
		this.val$x = x;
	}
	public void run() {
		System.out.println("ok:" + this.val$x);
	}
}
```

**会多一个属性来存储，生成一个新的构造方法，进行赋值，匿名内部类用的实际上是自己的属性，并不是外面方法所传入的属性**

```java
public class Candy11 {
	public static void main(String[] args) {
         final int x = 10;
		Runnable runnable = new Candy11$1();
	}
}
```

**这同时解释了为什么匿名内部类引用局部变量时，局部变量必须是 final 的**

**因为在创建 Candy11$1 对象时，将 x 的值赋值给了 Candy11$1 对象的 val属性** 

**所以X不应该再发生变了，如果变化了，那么x属性没有机会再跟着一起变化,因为类已经编译好了**

**这样才能保证匿名内部类里面的值和外部的值是一致的**



## 3.15:类加载阶段



### 1:加载阶段

将类编译`生成的字节码载入方法区`中，内部采用C++的`instanceKlass`描述java类

java是无法直接访问instanceKlass的，需要一个转换的过程

它的重要 field 有

![image-20220830095854154](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220830095854154.png)

- _java_mirror就是 java 和 instanceKlass 之间的桥梁
- 如果这个类还有父类没有加载，先加载父类 

- 加载和链接可能是交替运行的



注意 

- instanceKlass 这样的【元数据】是存储在方法区（1.8 后的元空间内），但_java_mirror 是存储在堆中 
- 可以通过前面介绍的 HSDB 工具查看

![image-20220830100312609](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220830100312609.png)

**字节码载入方法区，实际上载入的是C++的数据结构，存在一个_java_mirror**

**class无法直接访问Klass里面的属性，需要通过_java_mirror来实现转换**

**class类对象在堆，Klass在方法区【1.8之后属于元空间】**

**_java_mirror和类.class互相持有对方的地址,这样就可以互相转换访问了**

**我们new了很多个对象，每个对象都有一个对象头，存储了Class对象的地址**

**Class对象又储存了Klass的地址，通过_java_mirror进行转换来拿到各种的属性，方法等**

**实际上都是在方法区里的Klass里面来拿到的信息**



### 2:连接-验证

验证类是否符合 JVM规范，安全性检查

**我们可以使用工具想办法把Hello.class的字节码的魔数改变，不为CA FE BA BY**

**这个时候使用java Hello.class**

**发现出错了**

```elixir
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main" java.lang.ClassFormatError: Incompatible magic value
```



### 3:连接-准备

连接的准备阶段`主要是为 static 变量分配空间，设置默认值`

- static变量在JDK 7之前存储于instanceKlass末尾，从JDK7开始，存储于_java_mirror末尾 
- static变量分配空间和赋值是两个步骤，分配空间在准备阶段完成【现阶段】，赋值在初始化阶段完成 
- 如果static变量是final的基本类型，以及字符串常量，那么编译阶段值就确定了，赋值在准备阶段完成 
- 如果static变量是final的，但属于引用类型，那么赋值也会在初始化阶段完成

![image-20220830101702709](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220830101702709.png)

**在JDK1.8后，静态变量储存在类对象里面**

**在JDK1.6之前，静态变量存储在instanceKlass里面**



```java
public class Test{
	static int a;
}
```

**执行反编译：发现之后只有a的空间分配，但是没有赋值，赋值的话需要有Cinit**

```java
public class Test{
    static int a;
    static int b=10;
}
```

**javap编译发现：出现了static{}--->就是cinit，就会执行putStatic来执行赋值**

**所以：static的变量，分配空间和赋值是两个阶段**

```java
public class Test{
    static int a;
    static int b=10;
    static final int c=20;
    static final String name="ZZX"
}
```

**javap查看，发现static{}里没有putstaic给c和name赋值的情况，表明final修饰的static基本类型，以及字符串常量在初始化的时候就已经赋值了**

```java
public class Test{
	static int a;
    static int b=10;
    static final int c=20;
    static final String name="ZZX"
    static final Object obj=new Object();
}
```

**javap查看，发现static{}里没有putstaic给c和name赋值的情况，但是存在putstaic给obj的情况，表明final修饰的static基本类型，以及字符串常量在初始化的时候就已经赋值了，但是如果是属于引用类型，那么赋值也会等到初始化阶段完成**



### 4:连接-解析

解析：将常量池中的符号引用解析为直接引用

```java
public class Load2 {
    public static void main(String[] args)throws ClassNotFoundException,IOException {
		ClassLoader classloader = Load2.class.getClassLoader();
		// loadClass 方法不会导致类的解析和初始化
		Class<?> c = classloader.loadClass("cn.itcast.jvm.t3.load.C");
		// new C();
		System.in.read();
	}
}

class C {
	D d = new D();
}

class D {
}
```

**使用LoadClass进行类加载的时候，默认都是懒惰的，加载类C的时候是不会加载类D的**

**使用new C()**

**注释掉：Class<?> c = classloader.loadClass("cn.itcast.jvm.t3.load.C")**

**发现类D已经存在**



### 5:初始化

(cinit)V 方法 

初始化即调用 (cinit)V ，虚拟机会保证这个类的『构造方法』的线程安全



初始化发生的时机

概括得说，类初始化是【懒惰的】

- 执行了main 方法，所在的类，总会被首先初始化 
- 首次`访问这个类的静态变量或静态方法`时，不是final修饰或者是final修饰的引用对象 
- 子类初始化，如果父类还没初始化，会引发父类的初始化 
- 子类访问父类的静态变量，只会触发父类的初始化，子类并不会初始化 
- Class.forName，参数二不是false的话会导致初始化 
- new 会导致初始化



不会导致类初始化的情况

- 访问类的 static final 静态常量（基本类型和字符串）不会触发初始化 ，在连接的准备阶段完成
- 使用`类对象.class` 不会触发初始化【类对象.class在类加载时就会绑定_java_mirror】
- 创建该`类的数组`不会触发初始化
- 类加载器的loadclass方法不会初始化
- Class.forname的参数二为false时



**实验：未初始化的情况 **

```java
public class Test02 {
    public static void main(String[] args) throws ClassNotFoundException {
        //访问类的 static final 静态常量（基本类型和字符串）不会触发初始化
        System.out.println(A.a);

        //类对象.class 不会触发初始化【类对象.class在类加载时就会绑定_java_mirror】
        System.out.println(A.class);

        //创建该类的数组不会触发初始化
        Class[] classes = {A.class};

        //类加载器的loadclass方法不会初始化【只会加载】
        ClassLoader classLoader=Thread.currentThread().getContextClassLoader();
        classLoader.loadClass("com.zzx.A");

        //Class.forname的参数二为false时
        Class.forName("com.zzx.A",false,classLoader);
    }
}

class A{
    public static final int a=10;
    static {
        System.out.println("初始化");
    }
}

```

**发现以上情况都未打印初始化**



### 6:类加载-练习一

从字节码分析，使用 a，b，c 这三个常量是否会导致 E 初始化

```java
public static void main(String[] args) {
	System.out.println(E.a);
	System.out.println(E.b);
	System.out.println(E.c);
	}
}

class E {
	public static final int a = 10;
	public static final String b = "hello";
	public static final Integer c = 20;
    static{
        System.out.println("init");
    }
}
```

**a,b不会，c会导致类加载，执行javap反编译发现**

**调用C的话，会使用自动装箱操作，会调用invokestatic --> Integer.valueOf**

**然后putStatic赋值给c**



### 7:类加载-练习二

典型应用 - 完成懒惰初始化单例模式

单例模式：在整个JVM里面，只有一个这个类的对象

```java
public final class Singleton {
    // 限制为只能自己使用构造方法，其他类的方法里面不能创建我的对象
	private Singleton() { }
    
	// 使用内部类中保存单例
	private static class LazyHolder {
		static final Singleton INSTANCE = new Singleton();
	}
    
	// 第一次调用 getInstance 方法，才会导致内部类加载和初始化其静态成员
	public static Singleton getInstance() {
		return LazyHolder.INSTANCE;
	}
}
```

以上的实现特点是： 

- 懒惰实例化【不是类加载的时候就创建】
- 初始化时的线程安全是有保障的



## 3.16:类加载器



**以 JDK 8 为例：**

![image-20220830111818009](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220830111818009.png)

**为什么要这么多类加载器，其实就是为了各司其职，性能较好，专一**

**比如加载Student.class  Bootstrap不会加载，再让Extension加载，也没有，不加载**

**最后被Application加载，这种被称为双亲委派机制**

**应用程序类加载器 --> 拓展类加载器 --> Bootstrap类加载器【启动类加载器】**

**Extension使用getParent或者采用Bootstrap类加载器加载的类，使用getClassLoader得到的结果为null，**

**因为是C++的，无法直接访问**



### 1:启动类加载器

**用 Bootstrap 类加载器加载类：**

```java
package com.zzx
class F {
	static {
		System.out.println("bootstrap F init");
	}
}

public class Load5_1 {
	public static void main(String[] args) throws ClassNotFoundException {
	Class<?> aClass = Class.forName("com.zzx.F");
	System.out.println(aClass.getClassLoader());
	}
}
```



```elixir
java -Xbootclasspath/a:.com.zzx.Load5
-Xbootclasspath 表示设置 bootclasspath
其中 /a:. 表示将当前目录追加至 bootclasspath 之后
可以用这个办法替换核心类
java -Xbootclasspath:<new bootclasspath>
java -Xbootclasspath/a:<追加路径>
java -Xbootclasspath/p:<追加路径>
```



```elixir
结果：

bootstrap F init
null

为什么是null，因为Bootstrap类加载器是C++的
其他两个会打印AppClassLoader和ExtClassLoader
所以只要得到的结果是null，代表为Bootstrap类加载器
```



### 2:扩展类加载器

```java
package com.zzx
class G {
	static {
		System.out.println("classpath G init");
	}
}
public class Load5_2 {
	public static void main(String[] args) throws ClassNotFoundException {
		Class<?> aClass = Class.forName("com.zzx.G");
		System.out.println(aClass.getClassLoader());
	}
}
```



**默认情况肯定是在ClassPath下，所以肯定是AppClassLoader类加载器**

```apl
Classpath G init
sun.misc.Launcher$AppClassLoader@18b4aac2
```



**打个 jar 包**

将 jar 包拷贝到 JAVA_HOME/jre/lib/ext 重新执行

```apl
Classpath G init 
sun.misc.Launcher$ExtClassLoader@29453f44
```

**表明放在JAVA_HOME/jre/lib/ext的会被ExtClassLoade加载**



### 3:双亲委派模式

所谓的双亲委派：就是指`调用类加载器的loadClass方法时，查找类的规则`

**注意：这里应该说成上级更加合适，因为并没有继承关系，所以面试的时候可以提出这个观点**

![image-20221116160655536](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20221116160655536.png)

- 检查该类是否已经加载，没有加载的话看上级
- 有上级的话，委派上级 loadClass
- 如果没有上级了（ExtClassLoader）得到的结果为null，则委派 BootstrapClassLoader
- 每一层都找不到，调用 findClass 方法（每个类加载器自己扩展）来加载
- 记录耗时



```java
package com.zzx
class H {
	static {
		System.out.println("H init");
	}
}

public class Load5_3 {
	public static void main(String[] args) throws ClassNotFoundException {
		Class<?> aClass = 
		Load5_3.class.getClassLoader().loadClass("com.zzx.H");
		System.out.println(aClass.getClassLoader());
	}
}

```

- sun.misc.Launcher$AppClassLoader 开始查看已加载的类，结果没有
- sun.misc.Launcher$AppClassLoader 委派上级 sun.misc.Launcher$ExtClassLoader.loadClass()
- sun.misc.Launcher$ExtClassLoader 查看已加载的类，结果没有
- sun.misc.Launcher$ExtClassLoader 查看上级为null 委派上级Bootstrap加载类
- BootstrapClassLoader在 JAVA_HOME/jre/lib 下找 H 这个类，显然没有
- sun.misc.Launcher$ExtClassLoader调用自己的findClass，在JAVA_HOME/jre/lib/ext下找H这个类，显然没有，回到sun.misc.Launcher$AppClassLoader
- 继续执行到sun.misc.Launcher$AppClassLoader，调用它自己的 findClass方法，在 classpath下查找，找到了



### 4:线程上下文类加载器一

我们在使用 JDBC 时，都需要加载 Driver 驱动，不知道你注意到没有，不写

```java
Class.forName("com.mysql.jdbc.Driver")
```

也是可以让 com.mysql.jdbc.Driver 正确加载的，你知道是怎么做的吗？



我们追踪一下 DriverManager 源码：

```java
public class DriverManager {
	// 注册驱动的集合
	private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers
	= new CopyOnWriteArrayList<>();
    
	// 初始化驱动
	static {
		loadInitialDrivers();
		println("JDBC DriverManager initialized");
	}
}
```

**DriverManager是java.sql.DriverManager  所以类加载器是Bootstrap的类加载器**

```elixir
System.out.println(DriverManager.class.getClassLoader());

打印 null，表示它的类加载器是 Bootstrap ClassLoader
会到 JAVA_HOME/jre/lib 下搜索类，但JAVA_HOME/jre/lib 下
显然没有 mysql-connector-java-5.1.47.jar 包
这样问题来了，在DriverManager 的静态代码块中，怎么能正确加载 com.mysql.jdbc.Driver 呢？
```



**其实：它最后是使用 Class.forName 完成类的加载和初始化，关联的是应用程序类加载器，因此可以顺利**

**完成类加载**

**事实上：在某些时候，会打破双亲委派模式，会直接调用Application加载器来完成类加载，不然有些类是**

**找不到的**



**打破双亲委派机制：**即本来应该在上级的类加载器来加载类，但是有时候使用的是下级的类加载器，不然

有些类是找不到的



### 5:线程上下文类加载器二

Service Provider Interface【SPI】

约定如下，在实现类 jar 包的 META-INF/services 包下，以接口全限定名名为文件，文件内容是实现类名称

![image-20220831100914805](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831100914805.png)

这样就可以配合ServiceLoader的load方法根据接口找到实现类来进行实例化

```java
ServiceLoader<接口类型> allImpls = ServiceLoader.load(java.sql.Driver.class);
	Iterator<接口类型> iter = allImpls.iterator();
	while(iter.hasNext()) {
		iter.next();
	}
```

体现的是【面向接口编程+解耦】的思想，在下面一些框架中都运用了此思想：



- JDBC 
- Servlet 初始化器 
- Spring 容器 
- Dubbo（对 SPI 进行了扩展）



**怎么获取ServiceLoader呢**

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
	// 获取线程上下文类加载器
	ClassLoader cl = Thread.currentThread().getContextClassLoader();
	return ServiceLoader.load(service, cl);
}
```



线程上下文类加载器是当前线程使用的类加载器，默认就是应用程序类加载器，这样就可以加载那些例如java.mysql.jdbc.Driver类了



### 6:自定义类加载器

问问自己，什么时候需要自定义类加载器 

- 既不在启动类加载器的目录，也不再拓展类加载器的目录，也不在 classpath下面
- 想加载非 classpath 随意路径中的类文件 
- 都是通过接口来使用实现，希望解耦时，常用在框架设计
- 这些类希望予以隔离，不同应用的同名类都可以加载，不冲突，常见于 tomcat 容器



步骤： 

- 继承 ClassLoader 父类 
- 遵从双亲委派机制，重写findClass方法 不是重写loadClass方法，否则不会走双亲委派机制 
- 读取类文件的字节码 
- 调用父类的 defineClass 方法来加载类 
- 使用者调用该类加载器的 loadClass 方法



## 3.17:运行期优化

### 1:分层编译

```java
public class JIT1 {
	public static void main(String[] args) {
		for (int i = 0; i < 200; i++) {
			long start = System.nanoTime();
			for (int j = 0; j < 1000; j++) {
				new Object();
			}
			long end = System.nanoTime();
			System.out.printf("%d\t%d\n",i,(end - start)); //循环次数，消耗时间到纳秒
		}
	}
}
```

**计时统计：精确到纳秒**



**开始：耗费时间较长**

![image-20220831105626788](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831105626788.png)



**中期：耗费时间大大降低**

![image-20220831105723249](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831105723249.png)



**后期：耗费时间更低**

![image-20220831105755320](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831105755320.png)



原因是什么呢？

JVM 将执行状态分成了 5 个层次:



- 0 层，解释执行（Interpreter） 
- 1 层，使用 C1 即时编译器编译执行（不带 profiling） 
- 2 层，使用 C1 即时编译器编译执行（带基本的 profiling） 
- 3 层，使用 C1 即时编译器编译执行（带完全的 profiling） 
- 4 层，使用 C2 即时编译器编译执行



```elixir
profiling 是指在运行过程中收集一些程序执行状态的数据
例如【方法的调用次数】,【循环的回边次数】等

0层：字节码通过类加载器加载到JVM，由解释器将字节码转变为机器码，让CPU识别机器码，字节码被反复调用达到阈值之后，就会启用编译器来执行编译执行，上升到一层，解释器遇到相同的字节码，还是需要重复的转变为机器码

C1即时编译器：会把那些经常使用的字节码在达到阈值之后，使用C1即时编译器来提前编译存放，不用再每次来了之后使用解释器进行解释为机器码，然后CPU执行的操作，相当于把常用的达到阈值的先生成机器码作为缓存

C2即时编译器：相比于C1即时编译器，功能更加强大，优化程度更高
```



即时编译器（JIT）与解释器的区别：

- 解释器是将`字节码解释为机器码`，下次即使遇到相同的字节码，仍会执行重复的解释 
- JIT 是将一些`字节码编译为机器码`，并存入 Code Cache【缓存】下次遇到相同的代码，直接执行，无需 再编译 
- 解释器是将字节码解释为针对所有`平台都通用`的机器码【效率较低】 
- JIT 会根据平台类型，生成`平台特定`的机器码【效率高】



```apl
对于占据大部分的不常用的代码，我们无需耗费时间将其编译成机器码，而是采取解释执行的方式运
行；另一方面，对于仅占据小部分的热点代码，我们则可以将其编译成机器码，以达到理想的运行速
度。 执行效率上简单比较一下 Interpreter【解释器】 < C1 < C2，总的目标是发现热点代码
也就是hotspot名称的由来
```



刚才的一种优化手段称之为【逃逸分析】发现新建的对象是否逃逸

也就是看我创建的对象【上面是`new Object()`】是否会在外面被引用，是C2编译器做的优化

发现上面的`new Object()`根本就没用，C2即时编译器JIT进行优化之后，也许都不会创建这个对象

可以使用 -XX:- DoEscapeAnalysis 关闭逃逸分析，再运行刚才的示例观察结果



**发现不会得到刚刚800多纳秒的结果，全部集中在20000左右的阶段**



### 2:方法内联

```java
private static int square(final int i) {
	return i * i;
}	
System.out.println(square(9));
```

如果发现 square 是热点方法，并且长度不太长时，会进行内联，所谓的内联就是把方法内代码拷贝

也就是直接把方法的实现传入方法调用者这里

优化之后：

```java
System.out.println(9 * 9);
```

还能够进行常量折叠优化

```java
System.out.println(81);
```



```java
public static void main(String[] args) {
	int x = 0;
	for (int i = 0; i < 500; i++) {
		long start = System.nanoTime();
		for (int j = 0; j < 1000; j++) {
			x = square(9);
		}
		long end = System.nanoTime();
		System.out.printf("%d\t%d\t%d\n",i,x,(end - start));
	}
}

private static int square(final int i) {
	return i * i;
}
```

**结果：**

之前时间都是五位数纳秒

后面是4位数

后面是三位数

后面直接为0

发现后面根本不调用这个方法了，效率大大提升



**注意：如果方法很长的话，是不会进行内联的**



### 3:字段优化

方法是否内联会影响成员变量读取的优化

尽可能使用局部变量，少使用成员变量



### 4:反射优化

```java
package cn.itcast.jvm.t3.reflect;
import java.io.IOException;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
public class Reflect1 {
	public static void foo() {
		System.out.println("foo...");
	}
	public static void main(String[] args) throws Exception {
		Method foo = Reflect1.class.getMethod("foo");
		for (int i = 0; i <= 16; i++) {
			System.out.printf("%d\t", i);
			foo.invoke(null);
		}
		System.in.read();
	}
}
```



```apl
默认情况下：invoke前面 0 ~ 15 次调用使用的 MethodAccessor的 NativeMethodAccessorImpl实现

当调用到第 16 次（从0开始算）时,会采用运行时生成的类代替掉最初的实现,可以通过 debug 得到类名为 sun.reflect.GeneratedMethodAccessor1 然后会大幅度提升性能
```



```apl
sun.reflect.noInflation 可以用来禁用膨胀（直接生成 GeneratedMethodAccessor1，但首
次生成比较耗时，如果仅反射调用一次，不划算）
sun.reflect.inflationThreshold 可以修改膨胀阈值
```





# 4:内存模型



## 4.1:java内存模型

很多人将【java 内存结构】与【java 内存模型】傻傻分不清

【java 内存结构】是前面三章的内容

【java 内存模型】是 Java Memory Model（JMM）的意思

简单的说，JMM 定义了一套在`多线程读`写共享数据时（成员变量、数组）时

对数据的可见性、有序性、和原子性的规则和保障

跟java的内存结构没有什么关系





### 1:原子性和线程安全问题

原子性在学习线程时讲过，下面来个例子简单回顾一下：

问题提出:两个线程对初始值为0的静态变量一个做自增，一个做自减，各做5000 次，结果是0吗？

结果可能是正数、负数、零。为什么呢？因为 Java 中对静态变量的自增，自减并不是原子操作

```java
pulic class Demo4_1{
	static int i=0;
	public static void main(String[] args){
		Thread  t1=new Thread(()->{
            for(int j=0;j<5000;j++){
                i++;
            }
        )};
                              
        Thread  t2=new Thread(())->{
            for(int j=0;j<5000;j++){
                i++;
            }
        )};
                              
        t1.start();
        t2.start();
                              
        t1.join();
        t2.join();                      
	}
}
```

局部变量i++，是在局部变量表执行的iinc

静态变量的，要先把1和静态变量都加载到操作数栈，在执行iadd



![image-20220831161232676](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831161232676.png)



java内存模型分为了两个部分

- 主内存
- 工作内存

主内存存放共同操作的数据，工作内存是各自线程的区域，就是在工作内存和主内存来回切换

![image-20220831161315101](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831161315101.png)



如果是单线程以上代码是顺序执行（不会交错）就没有问题

![image-20220831161438036](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831161438036.png)



但多线程下代码可能交错运行（为什么会交错？思考一下）： 



出现负数的情况

![image-20220831162040526](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831162040526.png)

也就是在线程一加法还没结束的时候，线程二就读取(getstatic)的值为0了，这样就算加法线程结束了，我们读到的还是0，再执行减法的时候就用-1覆盖了1



出现正数的情况

![image-20220831162541833](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831162541833.png)

也就是在线程一减法还没结束的时候，线程二就读取(getstatic)的值为0了，这样就算减法线程结束了，我们读到的还是0，再执行加法的时候就用1覆盖了-1

```apl
所以我们要确保线程操作结束了之后采取读取数据
```



### 2:解决原子性

```java
synchronized( 对象 ) {
	要作为原子操作代码
}
```

用 synchronized 解决并发问题：

```java
static int i = 0;
static Object obj = new Object();
public static void main(String[] args) throws InterruptedException {
	Thread t1 = new Thread(() -> {
	for (int j = 0; j < 5000; j++) {
		synchronized (obj) {
			i++;
		}
	}
});
    
	Thread t2 = new Thread(() -> {
	for (int j = 0; j < 5000; j++) {
		synchronized (obj) {
		i--;
		}
	}
});
    
t1.start();
t2.start();
t1.join();
t2.join();
System.out.println(i);
}
```



```apl
对i++和i--加锁，就可以保证i++和i--的字节码是整体运行的
```

![image-20220831163034914](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831163034914.png)



![image-20220831163101660](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831163101660.png)

**就能保障上面的四行字节码能够确保原子运行**



```apl
如何理解呢：你可以把 obj 想象成一个房间，线程 t1，t2 想象成两个人。
当线程 t1 执行到 synchronized(obj) 时就好比 t1 进入了这个房间，并反手锁住了门，在门内执行
count++ 代码。
这时候如果 t2 也运行到了 synchronized(obj) 时，它发现门被锁住了，只能在门外等待
当 t1 执行完 synchronized{} 块内的代码，这时候才会解开门上的锁，从 obj 房间出来。t2 线程这时才可以进入 obj 房间，反锁住门，执行它的 count-- 代码。
```



专业解释：

```apl
使用了synchronized之后，存在monitor Owner EntryList Waitset

当线程T1进入时，发现owner为空【代表没人使用锁】，T1就会占用Owner
并执行monitorenter锁住整个monitor

当线程T2进入时，发现owner不为空，T2就不可以占用Owner，进入EntryList
当T1结束完毕，会离开Owner执行monitorexit释放整个monitor区域 并通知EntryList

T2再占用owner，并执行monitorenter锁住整个monitor区域

EntryList存在多个线程，就会争抢
```



上述代码是加在for循环的里面，会进行5000次反反复复的monitorenter和monitorexit

**可以加在for循环的外面，效率会高一点**



## 4.2:可见性



### 1:退不出的循环

先来看一个现象，main 线程对 run 变量的修改对于 t 线程不可见，导致了 t 线程无法停止：

```java
static boolean run = true;
public static void main(String[] args) throws InterruptedException {
	Thread t = new Thread(()->{
	while(run){
		// ....
	}
});
t.start();
Thread.sleep(1000);
run = false; // 线程t不会如预想的停下来
}
```



t 线程刚开始从主内存读取了 run 的值到工作内存。

![image-20220831164921277](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831164921277.png)



因为 t 线程要频繁从主内存中读取 run 的值，JIT 编译器会将 run 的值缓存至自己工作内存中的高速缓存中，减少对主存中 run 的访问，提高效率，以致于后面主线程修改了数据，但是t线程还是读到了true

![image-20220831165140944](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831165140944.png)



1 秒之后，main 线程修改了 run 的值，并同步至主存，而 t 是从自己工作内存中的高速缓存中读 取这个变量的值，结果永远是旧值

![image-20220831165157678](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831165157678.png)



### 2:退不出的循环解决办法



使用 volatile（易变关键字）

它可以用来修饰成员变量和静态成员变量，他可以避免线程从自己的工作缓存中查找变量的值，必须到主存中获取它的值，线程操作 volatile 变量都是直接操作主存



字节码如下

![image-20220831165609089](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831165609089.png)



- 体现了可见性【一个线程修改了数据，对另外的线程是可见的，避免缓存问题】

- 不保证原子性【跟没有加synchronized一样】

- 适用于一个线程写，多个线程读

  

synchronized 语句块既可以保证代码块的原子性，也同时保证代码块内变量的可见性

但是synchronized是属于重量级操作，性能相对更低

volatile 只能保证可见性，不能保证原子性



```apl
如果在前面示例的死循环中加入 System.out.println() 
会发现即使不加 volatile 修饰符，线程 t 也能正确看到对 run 变量的修改了，想一想为什么？
```

**其实是synchronized起的作用，println方法底层实际上加了synchronized，就会使当前线程读取主内存，而不是工作内存的缓存**



## 4.3:有序性



### 1:诡异的结果



I_Result 是一个对象，有一个属性 r1 用来保存结果，问，可能的结果有几种？

```java
int num = 0;
boolean ready = false;
// 线程1 执行此方法
public void actor1(I_Result r) {
	if(ready) {
		r.r1 = num + num;
	} else {
		r.r1 = 1;
	}
}
// 线程2 执行此方法
public void actor2(I_Result r) {
	num = 2;
	ready = true;
}
```



有同学这么分析 

- 情况1：线程1 先执行，这时 ready = false，所以进入 else 分支结果为 1 
- 情况2：线程2 先执行 num = 2，但没来得及执行 ready = true，线程1 执行，还是进入 else 分支，结果为1 
- 情况3：线程2 执行到 ready = true，线程1 执行，这回进入 if 分支，结果为 4（因为 num 已经执行过了）



但我告诉你，结果还有可能是 0

存在这样的情况，执行了ready=true，但是num还没有赋值... 就得到结果为0+0=0

**原因：产生了指令重排**





### 2:解决方法

volatile 修饰的变量，可以禁用指令重排



### 3:有序性理解



```java
static int i;
static int j;
// 在某个线程内执行如下赋值操作
i = ...; // 较为耗时的操作
j = ...;
```



可以看到，至于是先执行 i 还是 先执行 j ，对最终的结果不会产生影响。所以

上面代码真正执行时， 既可以是

![image-20220831173523362](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831173523362.png)

这种特性称之为『指令重排』，多线程下『指令重排』会影响正确性



### 4:happens-before

happens-before 规定了哪些写操作对其它线程的读操作可见

它是可见性与有序性的一套规则总结， 抛开以下 happens-before 规则

JMM 并不能保证一个线程对共享变量的写，对于其它线程对该共享变量的读可见





- 线程start前对变量的写，对线程start后对该变量的读可见

```java
static int x;
x = 10;
new Thread(()->{
	System.out.println(x);
},"t2").start();
```



- 线程对 volatile 变量的写，对接下来其它线程对该变量的读可见

```java
volatile static int x;
new Thread(()->{
	x = 10;
},"t1").start();

new Thread(()->{
	System.out.println(x);
},"t2").start();
```



- 线程解锁 m 之前对变量的写，对于接下来对 m 加锁的其它线程对该变量的读可见

```java
static int x;
static Object m = new Object();
new Thread(()->{
	synchronized(m) {
		x = 10;
	}
},"t1").start();

new Thread(()->{
synchronized(m) {
	System.out.println(x);
	}
},"t2").start();

```



- 线程结束前对变量的写，对其它线程得知它结束后的读可见

```java
static int x;
Thread t1 = new Thread(()->{
	x = 10;
},"t1");
t1.start();

t1.join();//等待线程t1执行完成
System.out.println(x);
```



- 线程t1打断 t2（interrupt）前对变量的写，对于其他线程得知 t2 被打断后对变量的读可见

```java
static int x;
public static void main(String[] args) {
	Thread t2 = new Thread(()->{
		while(true) {
			if(Thread.currentThread().isInterrupted()) {
			System.out.println(x);
			break;
		}
	}
},"t2");

t2.start();

new Thread(()->{
	try {
    	Thread.sleep(1000);
	} catch (InterruptedException e) {
		e.printStackTrace();
	}
	x = 10;
	t2.interrupt();
},"t1").start();

while(!t2.isInterrupted()) {
	Thread.yield();
}
System.out.println(x);
```



- 对变量默认值（0，false，null）的写，对其它线程对该变量的读可见 
- 具有传递性，如果 x hb-> y 并且 y hb-> z 那么有 x hb-> z
- 变量都是指成员变量或静态成员变量



## 4.4:CAS与原子类



### 1:CAS

CAS是配合volatile使用的重要技术，CAS即 Compare and Swap 

它体现的一种乐观锁的思想。比如多个线程要对一个共享的整型变量执行+1操作



![image-20220831180058315](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220831180058315.png)



获取共享变量时，为了保证该变量的可见性，需要使用 volatile 修饰。结合 CAS 和 volatile 可以实现无锁并发，适用于竞争不激烈、多核 CPU 的场景下

- 无锁并发：CAS 结合 volatile 

- 因为没有使用 synchronized，所以线程不会陷入阻塞，这是效率提升的因素之一 

- 但如果竞争激烈，可以想到重试必然频繁发生，会产生很多无效的修改，反而效率会受影响
- CAS就算一种不断尝试的机制



CAS 底层依赖于一个 Unsafe 类来直接调用操作系统底层的 CAS 指令

我们不能直接使用Unsafe类，需要使用反射才能调用到对应的API



### 2:乐观锁与悲观锁

java的乐观锁其实就是CAS

java的悲观锁其实就是synchronized

- CAS 是基于乐观锁的思想：最乐观的估计，不怕别的线程来修改共享变量，就算改了也没关系， 我吃亏点再重试呗。 
- synchronized 是基于悲观锁的思想：最悲观的估计，得防着其它线程来修改共享变量，我上了锁 你们都别想改，我改完了解开锁，你们才有机会。



### 3:原子操作类

JUC（java.util.concurrent）中提供了原子操作类，可以提供线程安全的操作

例如：AtomicInteger、 AtomicBoolean等

它们底层就是采用 CAS 技术 + volatile 来实现的。



例如AtomicInteger

就可以保证整数类型的原子操作，不需要自己设计，相当于封装了



### 4:synchronized优化介绍

Java HotSpot 虚拟机中，每个对象都有对象头（包括 class 指针和 Mark Word）。Mark Word 平时存储这个对象的 哈希码 、 分代年龄 ，当加锁时，这些信息就根据情况被替换为标记位 、 线程锁记录指针 、 重量级锁指针 、 线程ID 等内容



## 4.5:synchronized优化



### 1:轻量级锁

如果一个对象虽然有多线程访问，但多线程访问的时间是错开的（也就是没有竞争），那么可以使用轻量级锁来优化。这就好比：

学生（线程 A）用课本占座，上了半节课，出门了（CPU时间到），回来一看，发现课本没变，说明没有竞争，继续上他的课。 如果这期间有其它学生（线程 B）来了，会告知（线程A）有并发访问，线程 A 随即升级为重量级锁，进入重量级锁的流程

而重量级锁就不是那么用课本占座那么简单了，可以想象线程 A 走之前，把座位用一个铁栅栏围起来 假设有两个方法同步块，利用同一个对象加锁



![image-20220901092731423](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220901092731423.png)



### 2:轻量级锁-无竞争

每个线程都的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的 Mark Word



​            线程一                  对象Mark Word           线程二

![image-20220901093139286](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220901093139286.png)



### 3:轻量级锁-锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

```java
static Object obj=new Object();
public static void method1() {
	synchronized( obj ) {
		// 同步块
	}
}
```

![image-20220901094023746](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220901094023746.png)



### 4:重量级锁-自旋

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞。

在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能



自旋重试成功的情况

![image-20220901094836073](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220901094836073.png)



**自旋并不会因为别人未释放锁就马上阻塞线程，因为阻塞又重新竞争的话，消耗较大，时间较长**

- 自旋会占用CPU的时间，单核CPU自旋就是浪费，多核CPU自旋才是有优势的
- 汽车等红灯，熄火了就算阻塞，踩离合就算自旋【时间长了的话，不划算】
- Java 7 之后不能控制是否开启自旋功能



自旋重试失败的情况

![image-20220901095049463](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220901095049463.png)



### 5:偏向锁



轻量级锁存在锁重入这个情况

问题：每次锁重入的时候还是会用CAS来修改Mark Word的内容为当前线程的锁记录地址，没必要，因为之前就是自己的锁，修改了还是自己的锁，存在无效修改，可以使用偏向锁

![image-20220901092731423](https://zzx-note.oss-cn-beijing.aliyuncs.com/jvm/image-20220901092731423.png)



轻量级锁在没有竞争的时候，每次重入的时候需要进行CAS操作，偏向锁的优化:只有第一次来的时候才使用CAS将线程ID设置到对象的Mark Word头信息，之后发现这个线程ID是自己的话，就代表锁重入，不会进行重复的CAS

- 撤销偏向锁需要将持锁线程升级为轻量级锁，这个过程中所有线程需要暂停（STW） 
- 访问对象的 hashCode 也会撤销偏向锁 
- 如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程 T1 的对象仍有机会重新偏向 T2， 重偏向会重置对象的 Thread ID 
- 撤销偏向和重偏向都是批量进行的，以类为单位 
- 如果撤销偏向到达某个阈值，整个类的所有对象都会变为不可偏向的 
- 可以主动使用 -XX:-UseBiasedLocking 禁用偏向锁



### 6:其它优化

- 减少上锁时间 同步代码块中尽量短
- 锁的粒度尽量细，将一个锁拆分为多个锁提高并发度

```apl
例如CurrentHashMap  
只是锁住了数组+链表中的单个数组下标的链表
而HashTable是锁住了整个，效率就会低下很多
```

- 锁粗化

```apl
多次循环进入同步块不如同步块内多次循环 另外 JVM 可能会做如下优化，把多次 append 的加锁操作
粗化为一次（因为都是对同一个对象加锁，没必要重入多次）
new StringBuffer().append("a").append("b").append("c");
```

- 锁消除

```apl
JVM 会进行代码的逃逸分析，例如某个加锁对象是方法内局部变量，不会被其它线程所访问到，这时候
就会被即时编译器忽略掉所有同步操作。也就是JVM会检查是否存在线程安全问题，没有的话会消除无效的
```

- 读写分离

```apl
读的时候在原有数组读取，写的时候会复制一份到新数组来进行写，读操作不需要同步，写的时候才同步，就不需要对原始的加锁导致读写的时候都有锁
```





# 5:JVM常用参数

****

## 5.1:栈内存相关

- 栈内存大小：-Xss128k =====》设置每个线程栈的大小为128K







## 5.2:堆内存相关

- 堆内存初始大小：-Xms2048m ===》设置 JVM 初始堆 内存为2048M
- 堆内存最大大小：-Xmx2048m ===》设置 JVM 最大堆 内存为2048M





## 5.3:方法区相关

- -XX:MetaspaceSize / -XX:PermSize=256m 设置元空间/永久代初始值为256M
- -XX:MaxMetaspaceSize / -XX:MaxPermSize=256m 设置元空间/永久代最大值为256M





## 5.4:垃圾回收相关

- -XX:+PrintGCDetails ===》打印垃圾回收信息

- -XX:+DissableExplicitGC ===>标志自动将所有的system.gc()调用转换成一个空操作

  让System.gc()无效，因为jvm执行一个堆内存的“全部清扫”，比一个常规的GC操作要昂贵好几个数量级





## 5.5:字符串常量池相关

- -XX:StringTableSize=XXXX   ===》桶个数 StringTable底层是数组+链表，桶的个数就是数组长度

  适当增加桶个数，可以减少hash碰撞的概率





