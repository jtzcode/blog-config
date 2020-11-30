---
title: Groovy有何不同 —— 类型篇
date: 2020-11-10 10:41:59
categories:
    - 技术
tags:
    - Tech
    - Groovy
    - Java
    - 编程语言
---
## 引言
Groovy是一门运行在Java虚拟机上的语言，语言特性与Java很类似，又有所区别，既可以利用JDK中丰富的API，由有自己独特之处。我们熟悉的应用场景之一就是在CI工具Jenkins中编写集成脚本。这篇文章就从**类型**的角度来分享一下Groovy与Java的一些不同之处，供您参考。
![groovy](groovy-logo.png)

## 动态类型
Groovy是动态强类型语言。静态类型语言的一个特点是编译时的类型检查，比如Java和C++语言。动态类型则可以把类型检查推迟到运行时。所谓类型的强弱，是指在运行时是否能及时发现类型错误。

比如C++，如果把一个变量强制转换为一个错误类型，编译器会并不会阻止，而在运行时是否出错、以及什么时候出现则不一定。根据内存如何配置，调用是否多态，虚函数表如何组织等不同条件，最终表现也很难预测。这就是“弱”类型语言的表现。从这个意义来讲，Java和Groovy是强类型语言，可以在运行时找出错误的类型转换。
<!--more-->
Groovy还是一种动态类型语言，可以不在编译时做类型检查，也可以动态修改程序结构，这给Groovy提供了足够的灵活性，以实现一些强大的功能，比如元编程能力。下图展示了类型的强弱与动态静态之间的关系。
![type](groovy-type.png)

### 能力式设计
由于Groovy语言具有动态性，在使用它进行程序设计时也与Java有所不同。

比如接口的设计，使用Java语言时我们通常采用**契约式**设计，即使用接口定义一组操作作为契约，凡是实现这个接口的类都要履行接口的契约，完成对指定方法的实现。这样做的好处是，我们不关心某一个类型是否是另一个类型（大象是否是一个动物，男人是否是人类，机器人是否是工人，等等），只关心它是否实现了某个接口定义的功能。

契约式设计的局限性是，我们在使用接口提供的功能时，要明确指定接口类型，即调用者依赖接口定义，比如：
```java
interface Worker {
    public void work();
}

class Robot implements Worker {
    public void work() {
    }
}

class User {
    public static void startWorking(Worker worker) {
        worker.work();
    }
}

Worker robot = new Robot();
User.startWorking(robot);
```
`User`要使用`Worker`完成工作，需要显式依赖并传入`Worker`接口类型。虽然我们保证了传入的对象一定具有`work`方法，但我们无法传入**具有work公开方法但没有实现Worker接口**的其他任何对象。Groovy等动态语言可以解决这个问题，这就是所谓的**能力式设计**，接受一切具备某种能力的对象，无论它是什么类型、实现了什么接口。回到刚才的例子：

```groovy
def startWorking(worker) {
    worker.work();
}
```
这里凡是具有`work`能力（即定义了`work`方法）的对象都可以传递给`startWorking`方法。

你可能会问，如果worker对象没有定义work方法怎么办？的确，这种设计需要使用者保证这一点，因此是有风险的。这是在安全性和灵活性之间的一种权衡，如果你希望编译器帮你保证调用的正确，那就要牺牲一定的灵活性，反之亦然。

使用动态类型语言需要谨慎和保守一些，并写好充分的单元测试，单元测试可以帮助我们发现一些潜在的、编译器无法直接发现的问题。

### 多方法
Groovy的动态性还改变了多态的实现方式。如果一个类中有重载的方法，Groovy会聪明地选择正确的实现。不仅基于调用方法的对象，还基于所提供的参数。因为方法分派基于多个实体：目标加参数，所以这被称作**多分派或多方法（Multimethods）**。回到刚才的例子：

```java
class Worker {
    public void work(Number taskID) {

    }
}

class Man extends Worker {
    public void work(Number taskID) {
    }
}
class Robot extends Worker {
    public void work(Number task) {
    }

    public void work(BigDecimal taskID) {
    }
}

class User {
    public static void startWorking(Worker worker) {
        worker.work(new BigDecimal(1230.0));
    }
}

Worker robot = new Robot();
User.startWorking(robot);

Worker man = new Man()
User.startWorking(man);
```
这次`Man`和`Robot`都继承于`Worker`类，实现了`work`方法。不同的是，`Robot`还另外重载了一个`work`方法，以`BigDecimal`为参数类型（重载`Number`类型）。那么在调用`startWorking`方法时，如果传入了`BigDecimal`类型的task ID，哪个方法会被执行呢？

在Java中多态是严格依据类型和方法签名来实现的，也就是说在基类`Worker`定义的`work`方法需要`Number`类型的参数，在调用多态方法时就会尽量寻找参数类型`Number`的`work`方法。在`Robot`类中，依然是第一个方法被调用，尽管它还声明了一个接受`BigDecimal`参数的方法，参数类型不一致，传入的`BigDecimal`会被隐式转换成`Number`。

而同样的代码放到Groovy环境中，结果会不同。如前所述，Groovy会同时匹配方法和参数，而不是严格依据基类的定义。这样在`Robot`类中，传入`BigDecimal`类型的重载方法就会被调用。

### 类型检查和编译
Groovy在默认情况下是不执行静态类型检查的，不过有时为了保证类型安全也可以开启该功能。比如：
```groovy
@groovy.transform.TypeChecked 
def log(String text) {
    println(text.touppercase())
}
```
在使用了`TypeChecked`注解后，对于类型上的错误（上面的touppercase方法拼写有误）Groovy在编译时就会报错。

Groovy元编程和动态类型的特性虽然灵活，但是需要以性能为代价。性能的下降与代码、所调用方法的个数等因素相关。当不需要元编程和动态能力时，与等价的Java代码相比，性能损失可能高达10%。

如果想避免大部分性能损失，可以选择关闭动态特性。可以使用`@CompileStatic`注解让Groovy执行静态编译。这样为目标代码生成的字节码会和Java生成的字节码很像。

下一篇文章将继续关注Groovy的其他特性。

## 参考资料

- Groovy程序设计 [美] Venkat Subramaniam