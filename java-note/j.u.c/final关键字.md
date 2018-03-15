# 原理介绍
## 内存语义
### 重新排序规则
+ 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
+ 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

#### 写final域的重排序规则
> 写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则的实现包含下面2个方面
>
> + JMM禁止编译器把final域的写重排序到构造函数之外。
>
> + 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。

#### 读final域的重排序规则
> 在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（注意，这个规则仅仅针对处理器）。编译器会在读final域操作的前面插入一个LoadLoad屏障。
> 读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。

#### 引用类型final域的重排序规则
> 在构造函数内对一个final引用的 **对象的成员域的写入**，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。


# 代码分析
```
public class FinalExample {
    int i;                            //普通变量
    final int j;                      //final变量
    static FinalExample obj;

    public void FinalExample () {     //构造函数
        i = 1;                        //写普通域
        j = 2;                        //写final域
    }

    public static void writer () {    //写线程A执行
        obj = new FinalExample ();
    }

    public static void reader () {       //读线程B执行
        FinalExample object = obj;       //读对象引用
        int a = object.i;                //读普通域
        int b = object.j;                //读final域
    }
}
```

## 写final域的重排序规则分析
writer ()方法只包含一行代码：finalExample = new FinalExample ()。

> 这行代码包含两个步骤：
> + 构造一个FinalExample类型的对象；
> + 把这个对象的引用赋值给引用变量obj。
假设线程B读对象引用与读对象的成员域之间没有重排序（马上会说明为什么需要这个假设）

### 可能的执行时序
![final-write](https://github.com/alanzhang211/learning-note/raw/master/img/final-write.png)

在上图中，写普通域的操作被编译器重排序到了构造函数之外，读线程B错误的读取了普通变量i初始化之前的值。而写final域的操作，被写final域的重排序规则“限定”在了构造函数之内，读线程B正确的读取了final变量初始化之后的值。

## 读final域的重排序规则分析
> reader()方法包含三个操作：
> + 初次读引用变量obj。
> + 初次读引用变量obj指向对象的普通域j。
> + 初次读引用变量obj指向对象的final域i。


### 可能的执行时序
> 假设写线程A没有发生任何重排序，同时程序在不遵守间接依赖的处理器上执行。

![final-read](https://github.com/alanzhang211/learning-note/raw/master/img/final-read.png)

在上图中，读对象的普通域的操作被处理器重排序到读对象引用之前。读普通域时，该域还没有被写线程A写入，这是一个错误的读取操作。而读final域的重排序规则会把读对象final域的操作“限定”在读对象引用之后，此时该final域已经被A线程初始化过了，这是一个正确的读取操作。

## final域是引用类型分析
```
public class FinalReferenceExample {
final int[] intArray;                     //final是引用类型
static FinalReferenceExample obj;

public FinalReferenceExample () {        //构造函数
    intArray = new int[1];              //1
    intArray[0] = 1;                   //2
}

public static void writerOne () {          //写线程A执行
    obj = new FinalReferenceExample ();  //3
}

public static void writerTwo () {          //写线程B执行
    obj.intArray[0] = 2;                 //4
}

public static void reader () {              //读线程C执行
    if (obj != null) {                    //5
        int temp1 = obj.intArray[0];       //6
    }
}
}
```
> 对上面的示例程序，我们假设首先线程A执行writerOne()方法，执行完后线程B执行writerTwo()方法，执行完后线程C执行reader ()方法。

### 可能的执行时序

![final-ref](https://github.com/alanzhang211/learning-note/raw/master/img/final-ref.png)

在上图中，1是对final域的写入，2是对这个final域引用的对象的成员域的写入，3是把被构造的对象的引用赋值给某个引用变量。这里除了前面提到的1不能和3重排序外，2和3也不能重排序。

JMM可以确保读线程C至少能看到写线程A在构造函数中对final引用对象的成员域的写入。即C至少能看到数组下标0的值为1。而写线程B对数组元素的写入，读线程C可能看的到，也可能看不到。JMM不保证线程B的写入对读线程C可见，因为写线程B和读线程C之间存在数据竞争，此时的执行结果不可预知。

如果想要确保读线程C看到写线程B对数组元素的写入，写线程B和读线程C之间需要使用同步原语（lock或volatile）来确保内存可见性。


# 应用场景
> final有三种使用场景，分别是修饰变量、方法和类，无论哪种修饰，一旦声明为final类型，你将不能改变这个引用了，编译器会检查代码，如果你试图再次初始化，编译器会报错。将类、方法、变量声明为final能够提高性能，这样JVM就有机会进行估计，然后优化。

## 修饰变量
当final修饰一个变量的时候一般把他作为常量，通常和static关键字配合使用。

```
public static final String LOAN = "loan";
LOAN = new String("loan") //invalid compilation error
```

## 修饰方法
当一个方法被final修饰后，表示该方法不能被子类重写，final方法有一个好处是比非final方法要快，因为在编译时已经静态绑定了，不需要在运行时在动态绑定。
```
class PersonalLoan{
    public final String getName(){
        return "personal loan";
    }
}

class CheapPersonalLoan extends PersonalLoan{
    @Override
    public final String getName(){
        return "cheap personal loan"; //compilation error: overridden method is final
    }
}
```

## 修饰类
当一个类被final修饰后，表示该类是完整的，不能被继承，例如Java中String、Integer类都是final类。
```
 final class PersonalLoan{

    }

    class CheapPersonalLoan extends PersonalLoan{  //compilation error: cannot inherit from final class

}
```

# 实战分析
1. 不允许被继承的类，如：String 类。
2. 不允许修改引用的域对象，如：POJO 类的域变量。
3. 不允许被重写的方法，如：POJO 类的 setter 方法。
4. 避免局部变量在使用过程中被重新复制。可以在定义的时候，使用final进行修饰。
5. 避免上下文重复使用一个变量，使用 final描述可以强制重新定义一个变量，方便更好地进行重构。

# FAQ
1. 为什么final引用不能从构造函数内“逸出”？

写final域的重排序规则可以确保：在引用变量为任意线程可见之前，该引用变量指向的对象的final域已经在构造函数中被正确初始化过了。其实要得到这个效果，还需要一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程可见，也就是对象引用不能在构造函数中“逸出”。

**如：**
```
public class FinalReferenceEscapeExample {
final int i;
static FinalReferenceEscapeExample obj;

public FinalReferenceEscapeExample () {
    i = 1;                              //1写final域
    obj = this;                          //2 this引用在此“逸出”
}

public static void writer() {
    new FinalReferenceEscapeExample ();
}

public static void reader {
    if (obj != null) {                     //3
        int temp = obj.i;                 //4
    }
}
}
```
假设一个线程A执行writer()方法，另一个线程B执行reader()方法。这里的操作2使得对象还未完成构造前就为线程B可见。即使这里的操作2是构造函数的最后一步，且即使在程序中操作2排在操作1后面，执行read()方法的线程仍然可能无法看到final域被初始化后的值，因为这里的操作1和操作2之间可能被重排序。实际的执行时序可能如下图所示：

![final-out](https://github.com/alanzhang211/learning-note/raw/master/img/final-out.png)

2. 说说final关键字的好处。

+ final关键字提高了性能。JVM和Java应用都会缓存final变量。
+ final变量可以安全的在多线程环境下进行共享，而不需要额外的同步开销。
+ 使用final关键字，JVM会对方法、变量及类进行优化。

---
*参考资料：*
+ http://ifeve.com/java-memory-model/
+ 《阿里巴巴 Java 开发手册》
+ 《java并发编程的艺术》
