---
layout: post
title: Spring Framework AOP
categories: java
excerpt: Spring Framework AOP, V6.0.7
tags: Java Spring SpringFramework
date: 2023-04-05
isHidden: true
---
## 1.写在前面的内容
`AOP(Aspect Oriented Programming)` ，面向切面编程。思考一下，我们写类、写方法的目的是什么，是为了提高我们的开发效率，更好的复用、维护代码。我们将各种功能（方法）抽取，再“纵向”组合，快捷的开发出各式各样的系统。系统运行的时候是垂直纵向的，从第一行代码开始，到最后一行代码结束，从第一个方法开始，到最后一个方法结束。
向系统中添加新功能时，需要修改原有的代码，比如给一些方法添加日志输出，这需要在每个方法中添加 `log.info("xxxx")` ，这样不仅繁琐，而且不容易维护，可能向数千数万的方法中添加这行代码，随着版本不断的迭代，输出的日志格式还可能变化，很难维护。

`AOP` 就可以很方便的解决上述问题，向这个纵向的结构“横切上一刀”，在这个切面上进行编程。

有一个计算器，代码如下
```java
public interface Calculator {
    int add(int i, int j);
    int sub(int i, int j);
}

public class CalculatorImp implements Calculator {
    @Override
    public int add(int i, int j) {
        int res = i + j;
        return res;
    }
    @Override
    public int sub(int i, int j) {
        int res = i - j;
        return res;
    }
}
```

现在有需求要求输出每个方法执行的参数、结果等信息
```java
@Slf4j
public class CalculatorImp implements Calculator {
    @Override
    public int add(int i, int j) {
        log.info("add 方法开始，参数 i={}, j={}", i, j);
        int res = i + j;
        log.info("add 方法结束，参数 i={}, j={}，结果={}", i, j, res);
        return res;
    }
    @Override
    public int sub(int i, int j) {
        log.info("sub 方法开始，参数 i={}, j={}", i, j);
        int res = i - j;
        log.info("sub 方法结束，参数 i={}, j={}，结果={}", i, j, res);
        return res;
    }
}
```

这些日志输出代码，和业务功能耦合在一起，分散不易维护。我们最好将日志和业务代码分离出来，可以使用代理，实现分离。代理有动态和静态两种，

静态代理，这种方法虽然可以将日志功能分离出来，但是还是很死板，每个需要添加日志的类都要写一个代理类，想想都可怕
```java
@Slf4j
public class CalculatorImpProxy implements Calculator {

    private Calculator calculator;

    public CalculatorImpProxy(Calculator calculator) {
        this.calculator = calculator;
    }
    
    @Override
    public int add(int i, int j) {
        log.info("add 方法开始，参数 i={}, j={}", i, j);
        int res = calculator.add(i, j);
        log.info("add 方法结束，参数 i={}, j={}，结果={}", i, j, res);
        return res;
    }

    @Override
    public int sub(int i, int j) {
        log.info("sub 方法开始，参数 i={}, j={}", i, j);
        int res = calculator.sub(i, j);
        log.info("sub 方法结束，参数 i={}, j={}，结果={}", i, j, res);
        return res;
    }
}
```

动态代理，利用 `Proxy`
```java
@Slf4j
public class LoggerDynamicProxy {

    private Object target;

    public LoggerDynamicProxy(Object target) {
        this.target = target;
    }

    public Object getObject() {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    log.info("方法开始，method is {}, args are {}", method.getName(), Arrays.toString(args));
                    Object res = method.invoke(target, args);
                    log.info("方法结束，method is {}, args are {}, result is {}", method.getName(), Arrays.toString(args), res);
                    return res;
                });
    }
}
```

动态代理比静态代理灵活了一点，但也只是一点点，还是有问题，只能代理有接口的类，及侵入性。下面看看如何用 `Spring AOP` 如何优雅的解决这样问题，

AOP

开启动态代理
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.alamide.spring6.aop"/>
    <aop:aspectj-autoproxy/>
</beans>
```

配置切面
```java
@Slf4j
@Aspect
@Component
public class LogAspect {
    @Before("execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))")
    public void beforeMethod(JoinPoint joinPoint){
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("方法开始，method is {}, args are {}",methodName, args);
    }

    @AfterReturning(value = "execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))", returning = "res")
    public void afterMethod(JoinPoint joinPoint, Object res){
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("方法结束，method is {}, args are {}, result is {}", methodName, args, res);
    }
}
```

这就配置好了，完全没有侵入原有代码，透明、优雅、完美。

## 2.AOP 切面编程
添加依赖
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.7</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>6.0.7</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>6.0.7</version>
</dependency>
```

### 2.1 注解配置切面类
```java
@Slf4j
@Aspect
@Component
public class LogAspect {
    @Before("execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))")
    public void beforeMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("前置通知，method is {}, args are {}", methodName, args);
    }

    @AfterReturning(value = "execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))", returning = "res")
    public void afterMethodReturning(JoinPoint joinPoint, Object res) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("返回通知，method is {}, args are {}, result is {}", methodName, args, res);
    }

    @AfterThrowing(value = "execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))", throwing = "ex")
    public void afterThrowingMethod(JoinPoint joinPoint, Throwable ex) {
        String methodName = joinPoint.getSignature().getName();
        log.info("异常通知，method is {}, exception is {}", methodName, ex);
    }

    @After("execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))")
    public void afterMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("后置通知，method is {}, args are {}", methodName, args);
    }

    @Around("execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))")
    public Object aroundMethod(ProceedingJoinPoint point) {
        Object res = null;
        try {
            log.info("环绕通知，方法执行之前");
            res = point.proceed();
            log.info("环绕通知，方法执行之后");
        } catch (Throwable e) {
            log.info("环绕通知，方法出现异常");
        } finally {
            log.info("环绕通知，方法执行完毕");
        }

        return res;
    }
}
```

1. @Before 前置通知，目标方法被执行之前执行

2. @AfterReturning 返回通知，目标方法成功结束之后执行

3. @AfterThrowing 异常通知，目标方法异常结束之后通知 catch

4. @After 后置通知，目标方法最终结束之后通知 finally

5. @Around 环绕通知，上面所有的通知

`execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))` 是切入点表达式，语法如下

1. 修饰符和返回值（ `public int` ），可以用 `*` 代替，表示任意修饰符和任意返回类型

2. 包名部分（ `com.alamide.spring6.aop` ），可以用 `*` 表示一层包，如 `com.alamide.spring6.*` 表示 `spring6` 下的任意包；`*..` 表示任意包，任意层次，如 `com.alamide.*..` 表示 `com.almide` 下任意包，任意层次

3. 类名部分（ `CalculatorImp` ），`*` 可以表示部分匹配，也可以表示任意。`*Imp` 表示任意以 `Imp` 结尾的类，`*` 表示任意类

4. 方法名部分（ `.*` ），`*` 可以表示部分匹配，也可以表示任意。`*User` 表示任意以 `User` 结尾的方法，`*` 表示任意方法

5. 参数列表部分 ( `(..)` )，`(..)` 表示任意参数；`(int, ..)` 表示以 `int` 参数开头的参数类表。注意 `(int, ..)` 和 `(Integer, ..)` 是不一样的

6. 如果明确指定返回值类型，则必须同时写明修饰符

还可以重用切点表达式，上面的方法中切点表达式都是一样的，可以抽取出来，便于统一维护

```java
@Pointcut("execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))")
public void pointCut(){

}

//同一切面
@AfterReturning(value = "pointCut()", returning = "res")
public void afterMethodReturning(JoinPoint joinPoint, Object res) {
    String methodName = joinPoint.getSignature().getName();
    String args = Arrays.toString(joinPoint.getArgs());
    log.info("返回通知，method is {}, args are {}, result is {}", methodName, args, res);
}

//不同切面
@Before("com.alamide.spring6.aop.LogAspect.pointCut()")
public void beforeMethod(JoinPoint joinPoint) {
    String methodName = joinPoint.getSignature().getName();
    String args = Arrays.toString(joinPoint.getArgs());
    log.info("前置通知，method is {}, args are {}", methodName, args);
}
```

目标方法生有多个切面时，如何保证顺序？

可以给切面类加上 `@Order` 注解，`Order` 数值较小的数优先级高。这种时嵌套模式，优先级高的切面类包裹在低的外面。即前置先，后置后
```java
@Slf4j
@Aspect
@Component
@Order(99)
public class LogAspect {

    @Pointcut("execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))")
    public void pointCut(){

    }

    @Before("com.alamide.spring6.aop.LogAspect.pointCut()")
    public void beforeMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("前置通知，method is {}, args are {}", methodName, args);
    }

    @After("pointCut()")
    public void afterMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("后置通知，method is {}, args are {}", methodName, args);
    }
}

@Slf4j
@Component
@Aspect
@Order(100)
public class LogAspectLater {
    
    @Before("com.alamide.spring6.aop.LogAspect.pointCut()")
    public void beforeMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("LogAspectLater 前置通知，method is {}, args are {}", methodName, args);
    }

    @After("com.alamide.spring6.aop.LogAspect.pointCut()")
    public void afterMethod(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        String args = Arrays.toString(joinPoint.getArgs());
        log.info("LogAspectLater 后置通知，method is {}, args are {}", methodName, args);
    }
}
```

out:
```
09:20:53.500 [main] INFO com.alamide.spring6.aop.LogAspect -- 前置通知，method is div, args are [1, 1]
09:20:53.502 [main] INFO com.alamide.spring6.aop.LogAspectLater -- LogAspectLater 前置通知，method is div, args are [1, 1]
09:20:53.502 [main] INFO com.alamide.spring6.aop.CalculatorImp -- CalculatorImp.div method body execute()
09:20:53.502 [main] INFO com.alamide.spring6.aop.LogAspectLater -- LogAspectLater 后置通知，method is div, args are [1, 1]
09:20:53.502 [main] INFO com.alamide.spring6.aop.LogAspect -- 后置通知，method is div, args are [1, 1]
```

### 2.2 xml 配置切面类
```xml
<context:component-scan base-package="com.alamide.spring6.aop"/>
<aop:aspectj-autoproxy/>
<aop:config>
    <aop:aspect ref="logAspectLater">
        <aop:pointcut id="pointCut" expression="execution(public int com.alamide.spring6.aop.CalculatorImp.*(..))"/>
        <aop:before method="beforeMethod" pointcut-ref="pointCut"/>
        <aop:after-returning method="afterMethodReturning" pointcut-ref="pointCut" returning="res"/>
        <aop:after-throwing method="afterThrowingMethod" pointcut-ref="pointCut" throwing="ex"/>
        <aop:after method="afterMethod" pointcut-ref="pointCut"/>
    </aop:aspect>
</aop:config>
```