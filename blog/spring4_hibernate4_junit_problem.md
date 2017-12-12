## spring4+hibernate4+junit整合问题 Could not obtain transaction-synchronized Session for current thread的问题

当遇到org.hibernate.HibernateException: Could not obtain transaction-synchronized Session for current thread的问题时，要留意测试类是否加了@Transactional注解  
exception,问题原因，因为hibernate配置了事务处理，所以必须加上@Transactional注解，告诉spring执行程序时，需要进行事务处理。  
```
org.hibernate.HibernateException: Could not obtain transaction-synchronized Session for current thread
	at org.springframework.orm.hibernate4.SpringSessionContext.currentSession(SpringSessionContext.java:134)
	at org.hibernate.internal.SessionFactoryImpl.getCurrentSession(SessionFactoryImpl.java:1014)
	at com.etop.weiway.basic.dao.BaseDao.getSession(BaseDao.java:39)
	at com.etop.weiway.basic.dao.BaseDao.createQuery(BaseDao.java:110)
	at com.etop.weiway.basic.dao.BaseDao.createQuery(BaseDao.java:137)
	at com.etop.weiway.basic.dao.BaseDao.findUniqueResult(BaseDao.java:194)
	at com.etop.weiway.wxmanage.service.UserService.findByName(UserService.java:35)
	at com.etop.weiway.wxmanage.service.UserServiceTest.testFindByName(UserServiceTest.java:27)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
```

相关代码

```
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.transaction.annotation.Transactional;

/**
 * Created by Jeremie on 2014/10/23.
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = {"classpath:applicationContext.xml"})
@Transactional
public class UserServiceTest{

    @Autowired
    private UserService userService;

    @Before
    public void init(){

    }
    @Test
    public void testFindByName(){
        System.out.println(userService.findByName("jeremie").getPassword());
    }
}
```

[参考文档](http://www.cnblogs.com/javaleon/p/3978509.html)