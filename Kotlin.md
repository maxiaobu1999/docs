# 关键字

## companion

## object

## 类型

### var

### val

### constant

## null相关

###lateinit

```kotlin
private lateinit var dog: Dog
//判断初始化
if (::dog.isInitialized) {
    ....
}
```

# 操作符

:: 创建一个成员引用或者一个类引用

!!

When

# 集合

# lambda表达式

本质其实是`匿名函数`，因为在其底层实现中还是通过`匿名函数`来实现的。

- `Lambda`表达式总是被大括号括着
- 其参数(如果存在)在 `->` 之前声明(参数类型可以省略)
- 函数体(如果存在)在 `->` 后面。

*语法如下：*

```
 
```

*实例讲解：*

```
 view.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
               ...
            }
        });
//无参数        
view.setOnClickListener { ...}
//1参数
view.setOnClickListener { v -> 
		v.set()
    v.get()
}
//多参数
view.setOnClickListener { v1,v2,v3 -> 
		v1.set()
    v2.get()
}
//多回调rxjava
Obserable.subscribe ( Consumer<PublicResponseEntity<String>> {
                data->
					//。。。
        },Consumer<Throwable> {
					//。。。
        })
//或
Obserable.subscribe ({
                data->
					//。。。
        },{
					//。。。
        })
```



#静态

`companion object`：

```kotlin
	class Sample {
    ...
       👇
    companion object {
        val anotherString = "Another String"
    }
}
```

object:创建一个类，并且创建一个这个类的对象

`companion` 可以理解为伴随、伴生，表示修饰的对象和外部类绑定。

Java 中的 `Object` 在 Kotlin 中变成了 `Any`

单例类

```kotlin
 🏝️
// 👇 class 替换成了 object
object A {
    val number: Int = 1
    fun method() {
        println("A.method()")
    }
}    
```

但这里有一个小限制：一个类中最多只可以有一个伴生对象，但可以有多个嵌套对象。就像皇帝后宫佳丽三千，但皇后只有一个。

### 

