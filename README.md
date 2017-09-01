# 前言

这是一篇介绍Java编码风格的建议性文章，旨在提高Java代码的可读性，降低项目的维护成本。

> 代码的时间成本分为两个方面——编写和使用。
    
使用一个方法的时间成本体现在
1. 查看定义
2. 查看方法注释
3. 阅读方法实现
4. 实际测试该方法

&emsp;&emsp;优秀的方法只需要1，2步就可以使用，糟糕的方法则需要3，4步。如果一个项目中充斥着大量的糟糕方法，
那么开发效率就会低下。所以花费一些编写成本来换取使用成本是一种十分有用的编程技巧。

> 可读性不是简单的注释，简陋注释对调用者的帮助很小。

&emsp;&emsp;注释是降低方法使用成本的万能钥匙，但注释会花费大量的编写成本。
很多程序员被迫写注释，它们的注释通常比较简洁。明明写了注释，却还要阅读方法实现的情况时有发生。
如果一个方法的实现经常发生变动，维护注释的高昂成本很可能导致注释与方法实现不符，这样的错误注释反而误导了调用者。


# 使用异常

Java的声明式异常是十分优秀的语言特性，但Java开发者却很少自定义异常。

下面是一段常见的登录代码：
```Java
public class UserController {

    private UserService userService;

    /**
    * 1: 登录成功 2: 用户名密码错误 3: 用户被锁定 4: ...
    */
    public CommonResponse login(String username, String password) { 
        CommonResponse res = new CommonResponse();
        res.code = userService.login(username, password);
        if(res.code == 1){
            code.msg = "登录成功";
        }else if(res.code == 2){
            code.msg = "用户名密码错误";
        }else if(res.code == 3){
            code.msg = "用户被锁定";
        }else if(...){
            ...
        }
        return res;
    }
}

public class UserService {
    
    /**
    * 1: 登录成功 2: 用户名密码错误 3: 用户被锁定 4: ...
    */
    public int login(String username, String password) { ... }
}
```

上面的Java代码注释清楚明白，但是 UserService#login 的定义存在一些问题：

- **使用困难** 调用者需要写很多的代码才可以处理 UserService#login 的返回。<br>
有些程序员会让 UserService#login 直接返回CommonResponse。这样可以减少Controller层的一些代码，
但是如果Controller层需要调用多个Service，又或者遇到Service之间相互调用，那么依然会产生各种 if 判断。

- **注释重复** UserService#login 的注释很可能会在调用它的所有方法面上重复出现。

- **维护困难** 当 UserService#login 的返回发生了变动，比如增加了一种状态，那么就需要修改所有调用者的代码和注释来响应这一变动。一旦遗漏，就会产生一个不容易发现的Bug。


声明式异常就在这种时候大显神通：
```Java
/**
* 2: 用户名密码错误 3: 用户被锁定 4: ...
*/
public class LoginException extends Exception { ... }

public class UserService {
    
    public int login(String username, String password) throws LoginException { ... }
}

public class UserController {

    private UserService userService;

    public void login(String username, String password) throws LoginException {
        userService.login(username, password);
    }
}
```

- **消灭if** 绝大多数情况下，业务系统遇到自定义异常时都会停止后续逻辑，直接提示用户。

- **消灭重复** 所有的出错情况都只在异常类上面说明即可。

- **响应变动** 一旦某个方法增加了一种异常，所有的调用者如果不做出响应，那么在编译阶段就会出错，不会产生隐藏Bug。

## 异常使用技巧

1. 利用继承对异常进行分类，在需要的时候catch它。
2. 在框架中添加异常处理机制，或通知用户，或打印日志。
3. 不要 throw,catch Exception类，它会“淹没”所有异常。
4. 慎用RuntimeException，因为它不是**声明式**的。


# NULL

空指针是最常见的Java异常，为了避免它，Java代码中到处充斥着：
```Java
if(obj == null) { ... }
```

更糟糕的是：
```Java
//controller
void login(String username, String password){
    if(StringUtils.isEmpty(username)) { ... }
    if(StringUtils.isEmpty(password)) { ... }

    userService.service(username, password);
}

//service
void login(String username, String password){
    if(StringUtils.isEmpty(username)) { ... }
    if(StringUtils.isEmpty(password)) { ... }

    ...
    String hashPassword = hash(username + password + "salt");
    ...
}

String hash(String str){
    if(str == null) { ... }
    ... //do hash
    return str;
}
```

&emsp;&emsp;上面的代码对 username和password 做了重复的 null 判断。不仅浪费CPU而且增加编写的时间成本。
为了解决 null 问题，很多语言提供了语法糖或者从根本上不允许出现 null 的变量。

在Java中，我们需要一些约定来减少这种重复的劳动。

> * 所有的参数与返回都不允许为 null，如果可能返回 null，或者允许传入 null， 则必须声明。
> * 所有的数组，数据集合(如 java.util.Collection, java.util.stream.Stream... )都不可以为 null。

&emsp;&emsp;遵守这些约定，把 null 判断的职责交给调用者。最终仅在数据的产生位置保证数据的正确性。
对于业务系统来说，仅在JS端保证不会提交 null 数据即可。
如果程序被黑盒测试或者遇到第三方攻击绕过了JS，让JVM帮助我们抛出空指针异常即可。<br>
&emsp;&emsp;使用 java.util.Collections 中内置的各种空数据集传入，返回。

## 如何声明 NULL

* **注解** 使用 @Nullable 声明 允许为 null 的参数，可能为 null 的返回。
目前 Eclipse 和 IDEA 支持自有注解库的静态分析。如果一个返回值添加了 @Nullable 注解，而使用者不经判断直接使用，IDE就会给出警告。

* **java.util.Optional<T>** Java8的optional包装就隐含了可能是 null 的意思。

# 参数与返回


