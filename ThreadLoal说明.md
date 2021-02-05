
spring模板类需要绑定数据连接或会话的资源。
但这些资源本身是非线程安全的，

某个对象是非线程安全的，在多线程环境下，对对象的访问必须采用synchronized进行线程同步。
但模板类并未采用线程同步机制，
线程同步会降低并发性，影响系统性能。

ThreadLocal在Spring中发挥着重要的作用，在
管理request作用域的Bean、事务管理、任务调度、AOP等模块都出现了它们的身影.

ThreadLocal，不是一个线程，而是线程的一个本地化对象。
多线程中的对象使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程分配一个独立的变量副本。
每一个线程都可以独立地改变自己的副本，而不会影响其他线程所对应的副本。
从线程的角度看，这个变量就像是线程的本地变量。

ThreadLocal类接口很简单，只有4个方法：

void set(Object value)
设置当前线程的线程局部变量的值；

public Object get()
该方法返回当前线程所对应的线程局部变量；

public void remove()
将当前线程局部变量的值删除，减少内存的占用，不调用会自动被垃圾回收，调用可以加快内存回收的速度。

protected Object initialValue()

返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。
这是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次。默认实现直接返回一个null。

JDK5.0中 ThreadLocal已经支持泛型，该类的类名已经变为ThreadLocal<T>，API方法分别是void set(T value)、T get()以及T initialValue()。 

ThreadLocal是如何做到为每一个线程维护变量的副本？

实现的思路很简单：在ThreadLocal类中有一个Map，用于存储每一个线程的变量副本，Map中元素的key为线程对象，而value则是线程的变量副本。

一个简单的实现版本： 
public class SimpleThreadLocal {

	private Map valueMap = Collections.synchronizedMap(new HashMap());
	
	
	public void set(Object newValue) {
                //①键为线程对象，值为本线程的变量副本
		valueMap.put(Thread.currentThread(), newValue);
	}
	
	public Object get() {
		Thread currentThread = Thread.currentThread();
 
                //②返回本线程对应的变量
		Object o = valueMap.get(currentThread); 
                
                //③如果在Map中不存在，放到Map中保存起来
               if (o == null && !valueMap.containsKey(currentThread)) {
			o = initialValue();
			valueMap.put(currentThread, o);
		}
		return o;
	}
	public void remove() {
		valueMap.remove(Thread.currentThread());
	}
	public Object initialValue() {
		return null;
	}
}


ThreadLocal和线程同步机制相比有什么优势？

都是为了解决多线程中相同变量的访问冲突问题。 

在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。

ThreadLocal为每一个线程提供一个独立的变量副本，从而隔离了多个线程对访问数据的冲突。

每一个线程都拥有自己的变量副本，没有必要对该变量进行同步了。

同步机制采用了“以时间换空间”的方式
访问串行化，对象共享化。


ThreadLocal采用了“以空间换时间”的方式
访问并行化，对象独享化。

前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，可以同时访问而互不影响。 

Spring使用ThreadLocal解决线程安全问题 

只有无状态的Bean才可以在多线程环境下共享，

在Spring中，绝大部分Bean都可以声明为singleton作用域。

因为Spring对一些Bean（如RequestContextHolder、TransactionSynchronizationManager、LocaleContextHolder等）中非线程安全的“状态性对象”采用ThreadLocal进行封装，让它们也成为线程安全的“状态性对象”，

将一些非线程安全的变量以ThreadLocal存放，在同一次请求响应的调用线程中，所有对象所访问的同一ThreadLocal变量都是当前线程所绑定的。

public class TopicDao {
   
   //①一个非线程安全的变量
   private Connection conn; 
   
   
   public void addTopic(){
   
        //②引用非线程安全变量
	   Statement stat = conn.createStatement();
	   
	   …
   }
}

由于①处的conn是成员变量，因为addTopic()方法是非线程安全的，

必须在使用时创建一个新TopicDao实例（非singleton）。

下面使用ThreadLocal对conn这个非线程安全的“状态”进行改造： 

import java.sql.Connection;
import java.sql.Statement;
public class TopicDao {
 
  //①使用ThreadLocal保存Connection变量
private static ThreadLocal<Connection> connThreadLocal = new ThreadLocal<Connection>();


public static Connection getConnection(){
         
	    //②如果connThreadLocal没有本线程对应的Connection创建一个新的Connection，
        //并将其保存到线程本地变量中。
if (connThreadLocal.get() == null) {


			Connection conn = ConnectionManager.getConnection();
			
			connThreadLocal.set(conn);
			
			
              return conn;
		}else{
		
              //③直接返回线程本地变量
			return connThreadLocal.get();
			
		}
	}
	
	public void addTopic() {
 
		//④从ThreadLocal中获取线程对应的
         Statement stat = getConnection().createStatement();
	}
}

保证了不同的线程使用线程相关的Connection，

而不会使用其他线程的Connection。

因此，这个TopicDao就可以做到singleton共享了。 

当然，这个例子本身很粗糙，

将Connection的ThreadLocal直接放在Dao

只能做到本Dao的多个方法共享Connection时不发生线程安全问题，

但无法和其他Dao共用同一个Connection，

要做到同一事务多Dao共享同一个Connection，

必须在一个共同的外部类使用ThreadLocal保存Connection。

但这个实例基本上说明了Spring对有状态类线程安全化的解决思路。
