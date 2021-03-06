[TOC]



# 单例

保证一个类只有一个实例，并提供一个访问它的全局访问点

```java
class Singleton{
  private static volatile Singleton sInstance;// 保证写会内存
  private Singleton(){}
	public Singleton getInstance(){
    if(sInstance == null){
      synchronized(Singleton.class){
        if(sInstance == null){
          sInstance = new Singleton();
        }
      }
    }
    return sInstance;
  }
}		
		
```

# 3.1 抽象工厂

提供一个创建一系列相关或相互依赖对象的接口，而不指定他们具体的类

```java
// 相关的产品，同一工厂生产
abstract class ProductA{}// 汽车
abstract class ProductB{}// 飞机
// 具体产品 工厂1
class ProductA1 extends ProductA{}// Toyota汽车
class ProductB1 extends ProductB{}// Toyota飞机
// 具体产品 工厂2
class ProductA2 extends ProductA{}// Benz汽车
class ProductB2 extends ProductB{}// Benz飞机

// 工厂接口
interface Factory{
  ProductA createProductA();// 创建汽车
  ProdictB createProductB();// 创建飞机
}
// Toyota工厂
class Factory1 implements Factory{
  ProductA createProductA(){
    return new ProductA1();
  }
  ProductB createProductB(){
    retrun new ProductB1()
  }
}
// Benz工厂
class Factory2 implements Factory{
  ProductA createProductA(){
    return new ProductA2();
  }
  ProductB createProductB(){
    return new ProductB2();
  }
}
class Client{
  public void sample(){
    Factory factory1 = new Factory1();// toyota工厂
    Product productA1 = factory1.createProductA();//Toyota汽车
    Product productB1 = factory1.createProductB();//Toyota飞机
    
    Factory factory2 = new Factory2();// Benz工厂
    Product productA2 = factory2.createProductA();// Benz汽车
    Product productB2 = factory2.createProductB();// Benz飞机
  }
}

```



# 3.3 工厂方法

定义一个创建对象的接口，让子类决定实例化哪个类，使一个类的实例化延迟到其子类。

```java
abstract class Product{
  abstract void use();
}
class ProductA{
  void use(){}
}
class ProductB{
  void use(){}
}
abstract interface Factory{
  Product createProduct();
}
class FactoryA implements Factory{
  public Product createProduct(){
    return new ProductA();
  }
}
class FactoryB implements Factory{
  public Product createProduct(){
    return new PaoductB();
  }
}

Class Client{
  Factory factory = new FactoryA();
  Product product = factory.createProduct();
  product.use();
}
```







# 观察者

定义一种一对多的依赖关系，当一个对象的状态发生改变，所有依赖于它的对象都会得到通知并且自动更新。

```java
class Subject{
  private Set<Observer> mObservers=new HashSet<>();
  void attach(Observer observer){
    mObserver.add(observer);
  }
  void detach(Observer observer){
    mObserver.remove(observer);
  }
  
  void notify(){
    for(Observer observer : mObservers){
      observer.update();
    }
  }
}

abstract class Observer{
 abstract void update();
}

class ConcreteObserver{
  void update(){
    // ...
  }
}
```





# 职责链

把多个对象连成一条链，并沿这条链传递请求，直到有对象处理他为止。使多个对象都有机会处理请求，解耦请求的发送者与接收者。

```java
class Reuqest{};
abstract class Handler{
  public Handler next;
  abstract void handleRequest(String request);
}

class ConcreteHandler extends Handler{
  void handleRequest(String request){
    // ...
    if(next != null)
      next.handleRequest(request);
    // ... 
  }
}

class Client{
  Handler handler = new ConcreteHandler();
  // handler.next = new ConcreteHandler2();
  handler.handleRequest(...);
}
```



# 代理

为其他对象提供一个代理，以控制这个对象访问

1、远程代理

​	为不在同一空间的对象提供，一个本地代理。binder

2、虚代理

​	延迟被代理对象的初始化，ContentProdiver

3、保护代理

4、智能指引

# 装饰DECORATOR

动态地给一个对象添加一些额外的指责，比生成子类更灵活；