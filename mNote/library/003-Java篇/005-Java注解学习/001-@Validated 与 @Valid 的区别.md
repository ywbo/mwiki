# @Validated 与 @Valid 的区别 注解实现

## 用法

```java
public interface UserService {
    public  UserModel get2( Integer uuid) ;
}
@Validated      //① 告诉MethodValidationPostProcessor此Bean需要开启方法级别验证支持
@Component
public class UserServiceImpl implement UserService  {
    //②声明前置条件/后置条件
    public @NotNull UserModel get2(@NotNull @Min(value = 1) Integer uuid) { 
        //获取 User Model
        UserModel user = new UserModel(); //此处应该从数据库获取
        if(uuid > 100) {//方便后置添加的判断（此处假设传入的uuid>100 则返回null）
            return null;
        }
        return user;
    }
}

<!--注册方法验证的后处理器-->
<bean class="org.springframework.validation.beanvalidation.MethodValidationPostProcessor"/>
```

## @validated和@valid不同点

在spring项目中，@validated和@valid功能很类似，都可以在controller层开启数据校验功能。
但是@validated和@valid又不尽相同。有以下不同点：

> 1. 分组
> 2. 注解地方,@Valid可以注解在成员属性(字段)上,但是@Validated不行
> 3. 由于第2点的不同,将导致@Validated不能做嵌套校验
> 4. @valid只能用在controller。@Validated可以用在其他被spring管理的类上。
>    对于第4点的不同，体现了@validated注解其实又更实用的功能。那就是@validated可以用在普通bean的方法校验上。

## @validated的使用注意点

> 1 @validated和@valid都可以用在controller层的参数前面，但这只能在controller层生效。
> 2 @validated如果要开启方法验证。注解应该打在类上，而不是方法参数上。
> 3 方法验证模式下，被jsr303标准的注解修饰的可以是方法参数也可以是返回值，类似如下
> public @NotNull Object myValidMethod(@NotNull String arg1, @Max(10) int arg2)
> 4 @validated不支持嵌套验证。所以jsr303标准的注解修饰的对象只能基本类型和包装类型。其他类型只能做到检测是否为空，
> 对于对象里面的jsr303标准的注解修饰的属性，不支持验证。

---

【转】[@Validated注解实现](https://www.jianshu.com/p/89a800eda155)
