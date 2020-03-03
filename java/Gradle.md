## Gradle

### Gradle入门

- 强制依赖刷新命令（用于更新三方远程依赖的snapshot版本）

```groovy
./gradlew --refresh-dependencies assemble
```

### Groovy语法

#### 字符串

- 单引号和双引号都可以表示字符串 区别在于单引号内部的字符串中如果含有表达式是无法做运算的 例：

  ```groovy
  task printString << {
      def name = "zhangsan"
      
      println '单引号:${name}'
      println '双引号:${name}'
  }
  
  ./gradlew printString
  //结果
  //单引号:${name}
  //双引号:zhangsan
  ```

#### 集合

- List

  ```groovy
  task printList << {
      def numList = [1,2,3,4,5,6];
      println numList.getClass().name
      //输出ArrayList
      
      println numList[1]//第二个元素 2
      println numList[-1]//最后一个元素 6
      println numList[-2]//倒数第二个元素 5
      println numList[1..3]//2到4的元素
      
      numList.each{
          println it
      }
  }
  ```

- Map

  ```groovy
  task printMap << {
      def map = ['width':1024,'height':768];
      println map.getClass().name
      //hashMap
      //访问
      println map.width
      println map['height']
      map.each{
          println "key=${it.key},value=${it.value}"
      }
  }
  ```

#### 方法

```groovy
task invokeMethod << {
	method1(1,2)
    //可以省略括号
    method1 1,2
}

def method1(int a,int b) {
    println "$a : $b"
}

task printMethod1 << {
    def add1 = method2 1,2
    def add2 = method2 2,3
    println "add1:$add1,add2:$add2"
}
//return 语句可省略
def method2(int a,int b) {
    if(a >b) {
        a
    } else {
        b
    }
}

//代码块可以当成参数传递 （类似于kotlin的高阶函数和lamada表达式）
//1
numList.each( {
  println it  
})

//2
numList.each() {
  println it  
}

//3 如果方法的最后一个参数是我们的代码块则可以省略括号
numList.each{
  println it  
}
```

#### 闭包

- 闭包是实现DSL的基础 类似于java的匿名内部类 实际就是一段代码段 kotlin里的高阶函数

  ```groovy
  task Closure << {
      customEach {
          //这里实际就是对应参数closure
          println it
      }
      //类似于kotlin的lambda
      eachMap {k,v ->
          println "$k is $v"
      }
  }
  
  //定义闭包
  def customEach(closure) {
      for (int i in 1..10) {//1到10循环
          closure(i)
      }
  }
  
  //闭包参数
  def eachMap(closure) {
      def map1 = ["name":"zhangsan","age":18]
      map1.each {
          closure(it.key,it.value)
      }
  }
  ```

  