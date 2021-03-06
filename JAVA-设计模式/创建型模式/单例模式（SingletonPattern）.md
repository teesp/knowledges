# 一.简介

单例模式可以说是最常使用的设计模式了，它的作用是确保某个类只有一个实例，自行实例化并向整个系统提供这个实例。在实际应用中，线程池、缓存、日志对象、对话框对象常被设计成单例，总之，选择单例模式就是为了避免不一致状态，下面我们将会简单说明单例模式的几种主要编写方式，从而对比出使用枚举实现单例模式的优点。

# 二.单例设计模式的关键点

1. 私有构造函数

2. 声明静态单例对象

3. 构造单例对象之前要加锁（lock一个静态的object对象，或直接lock类）

4. 需要两次检测单例实例是否已经被构造，分别在锁之前和锁之后


# 三.首先看看饿汉式的单例模式：
```java
/**
 * Created by wuzejian on 2017/5/9.
 * 饿汉式
 */
public class SingletonHungry {

    private static SingletonHungry instance = new SingletonHungry();

    private SingletonHungry() {
    }

    public static SingletonHungry getInstance() {
        return instance;
    }
}
```

显然这种写法比较简单，但问题是无法做到延迟创建对象，事实上如果该单例类涉及资源较多，创建比较耗时间时，我们更希望它可以尽可能地延迟加载，从而减小初始化的负载，

# 四.于是便有了如下的懒汉式单例：

```java
/**
 * Created by wuzejian on 2017/5/9..
 * 懒汉式单例模式（适合多线程安全）
 */
public class SingletonLazy {

    private static volatile SingletonLazy instance;

    private SingletonLazy() {
    }

    public static synchronized SingletonLazy getInstance() {
        if (instance == null) {
            instance = new SingletonLazy();
        }
        return instance;
    }
}
```

这种写法能够在多线程中很好的工作避免同步问题，同时也具备lazy loading机制，遗憾的是，由于synchronized的存在，效率很低，在单线程的情景下，完全可以去掉synchronized，
# 五.双重检查锁单例模式，线程安全
为了兼顾效率与性能问题，改进后代码如下：

```java
1 public class Singleton {
2    private static volatile Singleton singleton = null;

3    private Singleton(){}

4   public static Singleton getSingleton(){
    //当已经有了实例singleton后，其他线程直接使用，不用等待，如果已经实例化，则提高了并发度。而Synchronized懒汉式则每个线程在判断是否已经实例化时，都需要等待上一个线程使用完释放了才能去判断。
5        if(singleton == null){
6           synchronized (Singleton.class){
7                if(singleton == null){
8                    singleton = new Singleton();
9                }
10            }
11        }
12       return singleton;
    }    
}
```

这种编写方式被称为 **双重检查锁**，主要在getSingleton()方法中，进行两次null检查。这样可以极大提升并发度，进而提升性能。毕竟在单例中new的情况非常少，绝大多数都是可以并行的读操作，因此在加锁前多进行一次null检查就可以减少绝大多数的加锁操作，也就提高了执行效率。

但是必须注意的是**volatile关键字**，该关键字有两层语义。

* 第一层语义是可见性，可见性是指在一个线程中对该变量的修改会马上由工作内存（Work Memory）写回主内存（Main Memory），所以其它线程会马上读取到已修改的值，注意工作内存是线程独享的，主存是线程共享的。

* volatile的第二层语义是禁止指令重排序优化，我们写的代码（特别是多线程代码），由于编译器优化，在实际执行的时候可能与我们编写的顺序不同。编译器只保证程序执行结果与源代码相同，却不保证实际指令的顺序与源代码相同，这在单线程并没什么问题，然而一旦引入多线程环境，这种乱序就可能导致严重问题。volatile关键字就可以从语义上解决这个问题，值得关注的是volatile的禁止指令重排序优化功能在Java 1.5后才得以实现，因此1.5前的版本仍然是不安全的，
> 如果没有加volatile，则依旧存在问题，假设线程A执行到了第5行，它判断对象为空，于是线程A执行到第8行，去初始化这个对象，但初始化是需要耗费时间的，但是这个对象的地址其实已经存在了。此时线程B也执行到了第5行，它判断不为空，于是直接跳到12行得到了这个对象。但是，这个对象还没有被完整的初始化，得到一个没有初始化完全的对象有什么用。

# 六.静态内部类单例模式，线程安全
或许我们可以利用静态内部类来实现更安全的机制，静态内部类单例模式如下：

```java
public class InnerClassSingleton implements Serializable {
	public static int times;

	private InnerClassSingleton() {
		System.out.println("单例构造器被调用"+(++times)+"次");
	}

	private static class InnerClassHelper {
		private static final InnerClassSingleton INSTANCE = new InnerClassSingleton();
	}

	public static final InnerClassSingleton getInstance() {
		return InnerClassHelper.INSTANCE;
	}
}
```


正如上述代码所展示的，我们把Singleton实例放到一个静态内部类中，这样可以避免了静态实例在Singleton类的加载阶段（类加载过程的其中一个阶段的，此时只创建了Class对象，关于Class对象可以看博主另外一篇博文， 深入理解Java类型信息(Class对象)与反射机制）就创建对象，毕竟静态变量初始化是在SingletonInner类初始化时触发的，并且由于静态内部类只会被加载一次，所以这种写法也是线程安全的。
>相应的基础知识
> 1. 什么是类级内部类？
> 简单点说，类级内部类指的是，有static修饰的成员内部类。如果没有static修饰的成员式内部类被称为对象级内部类。
> 2. 类级内部类相当于其外部类的static成分，它的对象与外部类对象间不存在依赖关系，因此可以直接创建。而对象级内部类的实例，是绑定在外部对象实例中的。
> 3. 类级内部类中，可以定义静态的方法。在静态方法中只能引用外部类中的静态成员方法或变量。
> 4. 类级内部类相当于其外部类的成员，只有在第一次被使用的时候才会被装载。

>  多线程缺省同步锁的知识：
>  大家都知道，在多线程开发中，为了解决并发问题，主要是通过使用synchronized来加互斥锁进行同步控
>  制， 但是在某些情况下，JVM已经隐含的为您执行了同步，这些情况下就不用自己再来进行同步控制了。
>  这些情况包括：
 1. 由静态初始化器（在静态字段上或static{}块中的初始化器）初始化数据时
 2. 访问final字段时
 3. 在创建线程之前创建对象时
 4. 线程可以看见它将要处理的对象时


>要想很简单的实现线程安全，可以采用静态初始化器的方式，它可以由JVM来保证线程的安全性。
>比如前面的饿汉式实现方式。但是这样一来，不是会浪费一定的空间吗？因为这种实现方式，会在类装载
>的时候就初始化对象，不管你需不需要。如果现在有一种方法能够让类装载的时候不去初始化对象，那不就
>解决问题了？
>一种可行的方式就是采用类级内部类，在这个类级内部类里面去创建对象实例。这样一来，只要不使用到这个类级内部类，那就不会创建对象实例，从而同步实现延迟加载和线程安全。


# 七.从上述4种单例模式的写法中，似乎也解决了效率与懒加载的问题，但是它们都有两个共同的缺点：

## 1. 序列化可能会破坏单例模式，比如每次反序列化一个序列化的对象实例时都会创建一个新的实例
```java
public  void destorySingletonByAntiSerializable() throws Exception {
		InnerClassSingleton o1 = null;
		InnerClassSingleton o2 = InnerClassSingleton.getInstance();

        //先将单例对象写入文件
		FileOutputStream fos = new FileOutputStream("InnerClassSingleton.obj");
		ObjectOutputStream oos = new ObjectOutputStream(fos);
		oos.writeObject(o2);
		oos.flush();
		oos.close();
        
        //从文件中读取单例对象
		FileInputStream fis = new FileInputStream("InnerClassSingleton.obj");
		ObjectInputStream ois = new ObjectInputStream(fis);
		o1 = (InnerClassSingleton) ois.readObject();
		ois.close();

		System.out.println(o1);
		System.out.println(o2);
		System.out.println(o1.equals(o2));

	}

```
结果：
>单例构造器被调用1次
com.zhangC.common.utils.InnerClassSingleton@156643d4
com.zhangC.common.utils.InnerClassSingleton@5e265ba4
false

执行完这段代码我们又会发现o1<>o2，可见通过反序列化，成功破坏了单例，创建了2个对象。那么如何避免这种情况发生呢？很简单，只要在代码中添加：
```java

//测试例子(四种写解决方式雷同)
public class Singleton implements java.io.Serializable {     
   public static Singleton INSTANCE = new Singleton();     

   protected Singleton() {     
   }  

   //反序列时直接返回当前INSTANCE
   private Object readResolve() {     
            return INSTANCE;     
      }    
}   
```

这时候我们可以再执行一下上面反序列化的方法，会很神奇的发现o1==o2，那这是为什么呢？我们一起来看下ois.readObject()的源码：
```java
private Object readObject0(boolean unshared) throws IOException {
	...省略
	case TC_OBJECT:
	  return checkResolve(readOrdinaryObject(unshared));
}

private Object readOrdinaryObject(boolean unshared){
	if (bin.readByte() != TC_OBJECT) {
            throw new InternalError();
        }

        ObjectStreamClass desc = readClassDesc(false);
        desc.checkDeserialize();

        Class<?> cl = desc.forClass();
        if (cl == String.class || cl == Class.class
                || cl == ObjectStreamClass.class) {
            throw new InvalidClassException("invalid class descriptor");
        }

        Object obj;
        try {
	//重点！！！
	//首先isInstantiable()判断是否可以初始化
	//如果为true，则调用newInstance()方法创建对象，这时创建的对象是不走构造函数的，是一个新的对象
            obj = desc.isInstantiable() ? desc.newInstance() : null;
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
        }

        passHandle = handles.assign(unshared ? unsharedMarker : obj);
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(passHandle, resolveEx);
        }

        if (desc.isExternalizable()) {
            readExternalData((Externalizable) obj, desc);
        } else {
            readSerialData(obj, desc);
        }

        handles.finish(passHandle);
	
	//重点！！！
	//hasReadResolveMethod()会去判断，我们的InnerClassSingleton对象中是否有readResolve()方法
        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())
        {
	//如果为true，则执行readResolve()方法，而我们在自己的readResolve()方法中 直接retrun InnerClassHelper.INSTANCE，所以还是返回的同一个对象，保证了单例
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                // Filter the replacement object
                if (rep != null) {
                    if (rep.getClass().isArray()) {
                        filterCheck(rep.getClass(), Array.getLength(rep));
                    } else {
                        filterCheck(rep.getClass(), -1);
                    }
                }
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
}

```

## 2. 使用反射强行调用私有构造器会破坏单例模式

```java
public  void destorySingletonByReflect() throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
		//通过单例模式获取单例对象innerClassSingleton1，调用构造器一次
		InnerClassSingleton innerClassSingleton1 = InnerClassSingleton.getInstance();
		System.out.println(innerClassSingleton1);
		//通过单例模式获取单例对象innerClassSingleton2
		InnerClassSingleton innerClassSingleton2 = InnerClassSingleton.getInstance();
		//观察控制台，这次获得的innerClassSingleton2和innerClassSingleton1是同一个对象，没有调用构造器
		System.out.println(innerClassSingleton2);
		//首先拿到万能的Class对象
		Class<InnerClassSingleton> clazz = InnerClassSingleton.class;
		//然后拿到构造器，使用这个方法私有的构造器也可以拿到
		Constructor<InnerClassSingleton> constructor = clazz.getDeclaredConstructor();
		//设置在使用构造器的时候不执行权限检查
		constructor.setAccessible(true);
		//由于没有了权限检查，所以在InnerClassSingleton类外面也可以创建对象了，然后执行方法
		//观察控制台，私有构造器又被调用了一次，单例模式被攻陷了，执行方法成功。
		System.out.println(constructor.newInstance());
	}

```
执行结果：

>单例构造器被调用1次
com.zhangC.common.utils.InnerClassSingleton@5e91993f
com.zhangC.common.utils.InnerClassSingleton@5e91993f
单例构造器被调用2次
com.zhangC.common.utils.InnerClassSingleton@1c4af82c


这就破坏了单例。为什么呢？罪魁祸首就是如下代码，它是反射的newInstance()的底层实现。

```java
UnsafeFieldAccessorImpl.unsafe.allocateInstance(class)
```

我们知道new创建对象时会被编译成3条指令：
1. 根据类型分配一块内存区域
2. 把第一条指令返回的内存地址压入操作数栈顶
3. 调用类的构造函数

而Unsafe.allocateInstance()方法值做了第一步和第二步，即分配内存空间，返回内存地址，没有做第三步调用构造函数。所以Unsafe.allocateInstance()方法创建的对象都是只有初始值，没有默认值也没有构造函数设置的值，**因为它完全没有使用new机制，绕过了构造函数直接操作内存创建了对象，而单例是通过私有化构造函数来保证的，这就使得单例失败。**

解决方案：在创建第二个实例时抛异常
```java
public class InnerClassSingleton implements Serializable {
	public static int times;

	private InnerClassSingleton() {
		System.out.println("单例构造器被调用"+(++times)+"次");
		if(InnerClassHelper.INSTANCE!=null){
			throw new IllegalArgumentException("单例构造器不能重复使用");
		}
	}

	private static class InnerClassHelper {
		private static final InnerClassSingleton INSTANCE = new InnerClassSingleton();
	}

	public static final InnerClassSingleton getInstance() {
		return InnerClassHelper.INSTANCE;
	}
}
```



最后总结一下静态内部类写法：

* 优点：不用synchronized，性能好；简单

* 缺点：无法避免被反射、反序列化破坏

## 使用枚举实现单例

```java
public enum EnumSingleton {
    
    INSTANCE;

    private Object data;

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }
    
}

```

反编译这段代码，得到：

```java
	static
        {
            INSTANCE = new EnumSingleton("INSTANCE",0);
            $VALUE = (new EnumSingleton[] {
                    INSTANCE
            });
        }

```


显然这是一种饿汉式的写法，用static代码块来保证单例（在类加载的时候就初始化了）。

代码相当简洁，我们也可以像常规类一样编写enum类，为其添加变量和方法，访问方式也更简单，EnumSingleton.INSTANCE进行访问，这样也就避免调用getInstance方法，更重要的是使用枚举单例的写法，我们完全不用考虑序列化和反射的问题。


### 可以避免被反射破坏

```java
//反射
Class clazz = EnumSingleton.class;
//拿到构造函数
Constructor c = clazz.getDeclaredConstructor(String.class, int.class);
c.setAccessible(true);
EnumSingleton instance1 = (EnumSingleton)c.newInstance("smart", 111);
-----------------------------------------------------------------------------------------
public T newInstance(Object ... initargs){
	if ((clazz.getModifiers() & Modifier.ENUM) != 0)
   	   throw new IllegalArgumentException("Cannot reflectively create enum objects");
} 

```
可以看到，在newInstance()方法中，做了类型判断，如果是枚举类型，直接抛出异常。也就是说从jdk层面保证了枚举不能被反射。


### 可以避免被反序列化破坏


Java规范中规定，每一个枚举类型极其定义的枚举变量在JVM中都是唯一的，在序列化的时候Java仅仅是将枚举对象的name属性输出到结果中，反序列化的时候则是通过 java.lang.Enum 的 valueOf() 方法来根据名字查找枚举对象。
```java
//以下为反序列化时的代码
...省略
EnumSingleton o1 = (EnumSingleton) ois.readObject();
-----------------------------------------------------------------------------------
private Object readObject0(boolean unshared) throws IOException {
	...省略
	case TC_ENUM:
	  return checkResolve(readEnum(unshared));
}
-------------------------------------------------------------------
private Object readEnum(boolean unshared){
	...省略
	String name = readString(false);
        Enum<?> result = null;
        Class<?> cl = desc.forClass();
        if (cl != null) {
            try {
                @SuppressWarnings("unchecked")
		//重点！！！
		//通过valueOf方法获取Enum，参数为class和name
                Enum<?> en = Enum.valueOf((Class)cl, name);
                result = en;
            } catch (IllegalArgumentException ex) {
                throw (IOException) new InvalidObjectException(
                    "enum constant " + name + " does not exist in " +
                    cl).initCause(ex);
            }
            if (!unshared) {
                handles.setObject(enumHandle, result);
            }
        }
}

```


所以序列化的时候只将 INSTANCE 这个名称输出，反序列化的时候再通过这个名称，查找对应的枚举类型，因此反序列化后的实例也会和之前被序列化的对象实例相同。


对于android平台这个可能未必是最好的选择，在android开发中，内存优化是个大块头，而**使用枚举时占用的内存常常是静态变量的两倍还多**，因此android官方在内存优化方面给出的建议是尽量避免在android中使用enum。但是不管如何，关于单例，我们总是应该记住：线程安全，延迟加载，序列化与反序列化安全，反射安全是很重重要的。


### ThreadLocal单例模式
```java
public class Singleton {
    
    private Singleton(){}
    
    private static final ThreadLocal<Singleton> threadLocal = 
            new ThreadLocal<Singleton>(){
                @Override
                protected Singleton initialValue(){
                    return new Singleton();
                }
            };
    
    public static Singleton getInstance(){
        return threadLocal.get();
    }
    
}
```

这种写法利用了ThreadLocal的特性，可以保证局部单例，即在各自的线程中是单例的，但是线程与线程之间不保证单例。

### 应用场景

在Spring的第三方包baomidou的多数据源中，有用到这种写法:
```java
package com.baomidou.dynamic.datasource.toolkit;
import java.util.concurrent.LinkedBlockingDeque;

public final class DynamicDataSourceContextHolder {
	//重点！！！
	private static final ThreadLocal<LinkedBlockingDeque<String>> LOOKUP_KEY_HOLDER = new ThreadLocal() {
        protected Object initialValue() {
            return new LinkedBlockingDeque();
        }
	private DynamicDataSourceContextHolder() {
    }

    public static String getDataSourceLookupKey() {
        LinkedBlockingDeque<String> deque = (LinkedBlockingDeque)LOOKUP_KEY_HOLDER.get();
        return deque.isEmpty() ? null : (String)deque.getFirst();
    }

    public static void setDataSourceLookupKey(String dataSourceLookupKey) {
        ((LinkedBlockingDeque)LOOKUP_KEY_HOLDER.get()).addFirst(dataSourceLookupKey);
    }

    public static void clearDataSourceLookupKey() {
        LinkedBlockingDeque<String> deque = (LinkedBlockingDeque)LOOKUP_KEY_HOLDER.get();
        if (deque.isEmpty()) {
            LOOKUP_KEY_HOLDER.remove();
        } else {
            deque.pollFirst();
        }
    }
    };
}

```

PS：initialValue()一般是用来在使用时进行重写的，如果在没有set的时候就调用get，会调用initialValue方法初始化内容。

# 八. Mybatis中的单例模式
在Mybatis中有两个地方用到单例模式，ErrorContext和LogFactory，

* 其中ErrorContext是用在每个线程范围内的单例，用于记录该线程的执行环境错误信息，

* 而LogFactory则是提供给整个Mybatis使用的日志工厂，用于获得针对项目配置好的日志对象。

## ErrorContext的单例实现代码：
```java
public class ErrorContext {
 
	private static final ThreadLocal<ErrorContext> LOCAL = new ThreadLocal<ErrorContext>();
 
	private ErrorContext() {
	}
 
	public static ErrorContext instance() {
		ErrorContext context = LOCAL.get();
		if (context == null) {
			context = new ErrorContext();
			LOCAL.set(context);
		}
		return context;
	}

```

构造函数是private修饰，具有一个static的局部instance变量和一个获取instance变量的方法，在获取实例的方法中，先判断是否为空如果是的话就先创建，然后返回构造好的对象。

只是这里有个有趣的地方是，LOCAL的静态实例变量使用了ThreadLocal修饰，也就是说它属于每个线程各自的数据，而在instance()方法中，先获取本线程的该实例，如果没有就创建该线程独有的ErrorContext。

