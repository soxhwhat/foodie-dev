### Spring Transaction
#### spring事务的原理
首先，我们先明白spring事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring是无法提供事务功能的。那么，我们一般使用JDBC操作事务的时候，代码如下(1)获取连接 Connection con = DriverManager.getConnection()(2)开启事务con.setAutoCommit(true/false);(3)执行CRUD(4)提交事务/回滚事务 con.commit() / con.rollback();(5)关闭连接 conn.close();
使用spring事务管理后，我们可以省略步骤(2)和步骤(4)，就是让AOP帮你去做这些工作。

#### spring什么情况下会进行事务回滚
首先，我们要明白Spring事务回滚机制是这样的：当所拦截的方法有指定异常抛出，事务才会自动进行回滚！
因此，如果你默默的吞掉异常，像下面这样那切面捕捉不到异常，肯定是不会回滚的。

还有就是，默认配置下，事务只会对Error与RuntimeException及其子类这些异常，做出回滚。一般的Exception这些Checked异常不会发生回滚（如果一般Exception想回滚要做出配置），如下所示
@Transactional(rollbackFor = Exception.class)

但是在实际开发中，我们会遇到这么一种情况！就是并没有异常发生，但是由于事务结果未满足具体业务需求，所以我们需要手动回滚事务，可以在代码里抛出一个自定义异常(常用)

#### spring事务什么时候失效
1. 发生自调用

```java
import org.springframework.stereotype.Service;

@Service
public class UserService {
    public void testUser() {
        this.test();
    }
    
    @Transactional
    public void test() {
        // do something
    }
}
```
此时这个this对象不是代理类，而是UserService对象本身！
解决方法很简单，让那个this变成UserService的代理类即可，就不展开说明了！
2. 方法不是public
   @Transactional注解的方法都是被外部其他类调用才有效！
3. 发生了未指定的异常
4. 数据库不支持事务

#### spring事务的传播行为
     * 事务传播类型 - Propagation
     * 事务传播类型，指的是事务与事务之间的交互策略。例如：在事务方法 A 中调用事务方法 B，当事务方法 B 失败回滚时，事务方法 A 应该如何操作？这就是事务传播类型。
     *
     * 针对事务传播类型，我们要弄明白的是 4 个点：
     *
     * 子事务与父事务的关系，是否会启动一个新的事务？
     * 子事务异常时，父事务是否会回滚？
     * 父事务异常时，子事务是否会回滚？
     * 父事务捕捉异常后，父事务是否还会回滚？
     *
     *      REQUIRED: 使用当前的事务，如果当前没有事务，则自己新建一个事务，子方法是必须运行在一个事务中的；
     *                如果当前存在事务，则加入这个事务，成为一个整体。
     *                举例：领导没饭吃，我有钱，我会自己买了自己吃；领导有的吃，会分给你一起吃。
     *      REQUIRES_NEW: 如果当前有事务，则挂起该事务，并且自己创建一个新的事务给自己使用；
     *                    如果当前没有事务，则同 REQUIRED
     *                    举例：领导有饭吃，我偏不要，我自己买了自己吃
     *      NESTED: 如果当前有事务，则开启子事务（嵌套事务），嵌套事务是独立提交或者回滚；
     *              如果当前没有事务，则同 REQUIRED。
     *              但是如果主事务提交，则会携带子事务一起提交。
     *              如果主事务回滚，则子事务会一起回滚。相反，子事务异常，则父事务可以回滚或不回滚。
     *              举例：领导决策不对，老板怪罪，领导带着小弟一同受罪。小弟出了差错，领导可以推卸责任。
     *      SUPPORTS: 如果当前有事务，则使用事务；如果当前没有事务，则不使用事务。
     *                举例：领导没饭吃，我也没饭吃；领导有饭吃，我也有饭吃。
     *      MANDATORY: 该传播属性强制必须存在一个事务，如果不存在，则抛出异常
     *                 举例：领导必须管饭，不管饭没饭吃，我就不乐意了，就不干了（抛出异常）
     *      NOT_SUPPORTED: 如果当前有事务，则把事务挂起，自己不适用事务去运行数据库操作
     *                     举例：领导有饭吃，分一点给你，我太忙了，放一边，我不吃
     *      NEVER: 如果当前有事务存在，则抛出异常
     *             举例：领导有饭给你吃，我不想吃，我热爱工作，我抛出异常

#### spring事务的隔离和数据库的事务隔离是一回事吗
是一回事！
而spring只是在此基础上抽象出一种隔离级别为default，表示以数据库默认配置的为主。例如，mysql默认的事务隔离级别为repeatable-read。而Oracle 默认隔离级别为读已提交。

#### 怎么保证spring事务内的连接唯一性
因为那个Connection在事务开始时封装在了ThreadLocal里，后面事务执行过程中，都是从ThreadLocal中取的，肯定能保证唯一，因为都是在一个线程中执行的!