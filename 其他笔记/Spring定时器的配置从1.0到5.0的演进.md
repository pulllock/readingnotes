这里主要是记录下从Spring1.0到现在的5.0中定时器的配置方式，关于源码，暂先不解释。主要用作自己记录用，如果有错误的还请指出一起改正学习，免得误导别人，谢谢。

# Spring1中定时器的配置
直接看Spring1.1.1的文档，里面都已经给出来了各种配置方式，更高版本的也都包含了这些，但是觉得看1.1.1的更纯粹一些。

Spring1中对定时器的支持有两种方式：

- jdk的Timer
- Quartz Scheduler

## Quartz Scheduler
Quartz Scheduler使用Triggers，Jobs，JobDetail来实现定时器功能。Spring提供了对Quartz的支持。

使用的大概步骤是：

1. 定义JobDetail，也就是定义具体的任务。
2. 定义trigger，就是定义触发器，指定什么任务，在什么时间执行或者隔多久执行。
3. 定义SchedulerFactoryBean，来执行任务。

下面我们以一个例子来说明，是一个定时的去获取信息和定时统计信息的示例。

### 定义JobDetail
定义JobDetail有两种方式，一种是使用JobDetailBean，一种是使用MethodInvokingJobDetailFactoryBean，后者可以指定要执行的具体方法。

#### 使用JobDetailBean
CountUserJob：

```
package me.cxis.spring.scheduling.quartz;

import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.scheduling.quartz.QuartzJobBean;

/**
 * Created by cheng.xi on 2017-04-19 11:00.
 * 定时的统计信息的JOb
 * 比如这里是定时的统计系统中总的用户数，总的用户数是我查询到的数和我在xml指定的数的总和
 */
public class CountUserJob extends QuartzJobBean{

    private int adminUser;

    public void setAdminUser(int adminUser) {
        this.adminUser = adminUser;
    }

    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        //执行真正的统计任务
        System.out.println("开始统计系统中人数");
        System.out.println("统计完成，共有101人");
        System.out.println("加上系统管理员之后共有" + (adminUser + 101) + "人");
    }
}
```

xml中声明一个job：

```
<!--定义一个JobDetailBean类型的Job，用来统计系统中总的人数，adminUser指的是系统中预先留的管理员数目-->
<bean name="countUserJob" class="org.springframework.scheduling.quartz.JobDetailBean">
    <property name="jobClass">
        <value>me.cxis.spring.scheduling.quartz.CountUserJob</value>
    </property>
    <property name="jobDataAsMap">
        <map>
            <entry key="adminUser">
                <value>10</value>
            </entry>
        </map>
    </property>
</bean>
```

#### 使用MethodInvokingJobDetailFactoryBean
MethodInvokingJobDetailFactoryBean可以指定方法。直接看例子。

GetJob：

```
package me.cxis.spring.scheduling.quartz;

/**
 * Created by cheng.xi on 2017-04-19 11:01.
 * 定时的获取信息的Job，定时从文件中获取数据
 */
public class GetJob {
    public void getSomethingFromFile(){
        System.out.println("从文件中获取数据。。。。");
    }
}

```

xml中配置：

```
<!--从文件中获取信息的bean-->
<bean id="getJob" class="me.cxis.spring.scheduling.quartz.GetJob"/>

<!--定义一个MethodInvokingJobDetailFactoryBean，从文件中获取数据的Job-->
<bean id="getJobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <property name="targetObject"><ref bean="getJob"/></property>
    <property name="targetMethod"><value>getSomethingFromFile</value></property>
</bean>
```

### 定义Triggers
上面我们把JobDetail都定义好了，也都配置好了，但是怎么去执行，多长时间执行一次都没有说明，这时候需要定义Triggers来描述任务什么时候执行等。只需要在xml中配置就可以了。

Triggers也有两种方式，一种是SimpleTriggerBean，一种是CronTriggerBean。我们的例子中，统计用户数使用SimpleTriggerBean，从文件中获取信息使用CronTriggerBean。

```
<!--定义Triggers，统计用户数-->
<bean id="countUserTrigger" class="org.springframework.scheduling.quartz.SimpleTriggerBean">
    <property name="jobDetail">
        <ref bean="countUserJob"/>
    </property>
    <!--第一次执行之前需要等待的时间-->
    <property name="startDelay">
        <!--10秒-->
        <value>10000</value>
    </property>
    <!--任务重复时间，每隔多少时间执行一次-->
    <property name="repeatInterval">
        <value>20000</value>
    </property>
</bean>

<!--定义Triggers，定时从文件中获取数据-->
<bean id="getJobTrigger" class="org.springframework.scheduling.quartz.CronTriggerBean">
    <property name="jobDetail">
        <ref bean="getJobDetail"/>
    </property>
    <property name="cronExpression">
        <!--cron表达式，这里是每隔两分钟执行一次-->
        <value>0 0/2 * * * ?</value>
    </property>
</bean>
```

### 定义SchedulerFactoryBean

```
<!--定义SchedulerFactoryBean-->
<bean class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref local="countUserTrigger"/>
            <ref local="getJobTrigger"/>
        </list>
    </property>
</bean>
```

### 测试
Main：

```
package me.cxis.spring.scheduling.quartz;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-04-19 11:34.
 */
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:scheduling-quartz.xml");
    }
}
```

## JDK Timer
使用JDK Timer，也跟上面类似，大概的步骤是：

1. 创建一个TimerTask，里面是执行任务的逻辑。
2. 创建ScheduledTimerTask，就是什么时候或者隔多久执行任务。
3. 创建TimerFactoryBean，来执行任务。

### 创建TimerTask
同样，创建一个Task也有两种方式，一种是继承TimerTask，另外一种是使用MethodInvokingTimerTaskFactoryBean，后者可以指定具体方法。

#### 继承TimerTask
CountUserTask：

```
package me.cxis.spring.scheduling.timer;

import java.util.TimerTask;

/**
 * Created by cheng.xi on 2017-04-19 11:00.
 * 定时的统计信息的 task
 * 比如这里是定时的统计系统中总的用户数，总的用户数是我查询到的数和我在xml指定的数的总和
 */
public class CountUserTask extends TimerTask{

    private int adminUser;

    public void setAdminUser(int adminUser) {
        this.adminUser = adminUser;
    }

    public void run() {
        //执行真正的统计任务
        System.out.println("开始统计系统中人数");
        System.out.println("统计完成，共有101人");
        System.out.println("加上系统管理员之后共有" + (adminUser + 101) + "人");
    }
}
```
在xml中配置bean：

```
<!--统计用户数的bean-->
<bean id="countUserTask" class="me.cxis.spring.scheduling.timer.CountUserTask">
    <property name="adminUser">
        <value>10</value>
    </property>
</bean>
```

#### 使用MethodInvokingTimerTaskFactoryBean
GetTask:

```
package me.cxis.spring.scheduling.timer;

/**
 * Created by cheng.xi on 2017-04-19 11:01.
 * 定时的获取信息的task，定时从文件中获取数据
 */
public class GetTask {
    public void getSomethingFromFile(){
        System.out.println("从文件中获取数据。。。。");
    }
}

```
xml中配置：

```
<!--从文件中获取信息的bean-->
<bean id="getTaskBean" class="me.cxis.spring.scheduling.timer.GetTask"></bean>

<!--使用MethodInvokingTimerTaskFactoryBean-->
<bean id="getTask" class="org.springframework.scheduling.timer.MethodInvokingTimerTaskFactoryBean">
    <property name="targetObject"><ref bean="getTaskBean"/> </property>
    <property name="targetMethod"><value>getSomethingFromFile</value></property>
</bean>
```

### 创建ScheduledTimerTask
定义ScheduledTimerTask来描述任务什么时候执行等。只需要在xml中配置就可以了。

使用Timer的方式，就这么一种配置，没法使用cron的方式。

```
<!--创建统计用户的ScheduledTimerTask，描述任务怎么运行-->
<bean id="countUserScheduledTimerTask" class="org.springframework.scheduling.timer.ScheduledTimerTask">
    <property name="delay">
        <value>10000</value>
    </property>
    <property name="period">
        <value>20000</value>
    </property>
    <property name="timerTask">
        <ref local="countUserTask"/>
    </property>
</bean>

<!--创建从文件获取信息的ScheduledTimerTask，描述任务怎么运行-->
<bean id="getScheduledTimerTask" class="org.springframework.scheduling.timer.ScheduledTimerTask">
    <property name="period">
        <value>60000</value>
    </property>
    <property name="timerTask">
        <ref local="getTask"/>
    </property>
</bean>
```

### 创建TimerFactoryBean
任务执行的配置：

```
<!--创建TimerFactoryBean，执行任务-->
<bean id="timerFactory" class="org.springframework.scheduling.timer.TimerFactoryBean">
    <property name="scheduledTimerTasks">
        <list>
            <ref local="countUserScheduledTimerTask"/>
            <ref local="getScheduledTimerTask"/>
        </list>
    </property>
</bean>
```

### 测试

```
package me.cxis.spring.scheduling.timer;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import java.io.IOException;
import java.io.InputStream;

/**
 * Created by cheng.xi on 2017-04-19 11:34.
 */
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:scheduling-timer.xml");
        try {
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
上面就是Spring1.x中关于定时器的配置方式，配置清晰，易懂，但是任务多了之后，就会发现配置文件会迅速变得臃肿。

# Spring2中定时器的配置
What's new in Spring 2.0?

- 增加对Executors的支持

上面就是AOP在2.0版本新增的特性，1.0的所有AOP配置方式在2.0中都支持，下面主要看看2.0中新增的一些方法。

Spring2.0中新定义了一个TaskExecutor接口，增加了对线程池的支持，这个接口的功能跟JDK1.5中的Executor接口一样。那么2.0中线程池的增加，对定时器有什么影响呢？其实就是可以在定时任务执行的时候，使用线程池来执行任务，我们不用关心其他的实现。

## Spring2中配置示例

直接看示例，我们定时，每隔5分钟，每次都从20个文件中同时获取数据。

首先写实际执行业务的类，

```
package me.cxis.spring.scheduling.executor;

/**
 * Created by cheng.xi on 2017-04-19 14:51.
 * 从文件中获取数据的Task
 */
public class GetDataFromFileTask implements Runnable {

    private int fileId;

    public GetDataFromFileTask(int fileId){
        this.fileId = fileId;
    }

    public void run() {
        //真正执行从文件中获取数据的逻辑
        System.out.println("从文件" + fileId + "中获取数据");
    }
}
```

然后是执行任务的定时器：

```
package me.cxis.spring.scheduling.executor;

import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.TimerTask;

/**
 * Created by cheng.xi on 2017-04-19 14:54.
 * 批量从文件中获取数据的定时器
 */
public class GetDataFromFileScheduler extends TimerTask {

    private ThreadPoolTaskExecutor executor;

    public void setExecutor(ThreadPoolTaskExecutor executor) {
        this.executor = executor;
    }

    public void run() {
    	System.out.println("CorePoolSize:" + taskExecutor.getCorePoolSize() + ";MaxPoolSize:" + taskExecutor.getMaxPoolSize());
        //每次都会同时执行从20个文件中获取数据
        for(int i = 0; i < 20;i++){
            executor.execute(new GetDataFromFileTask(i));
        }
    }
}

```

接着是xml的配置：

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
    <!--线程池taskExecutor-->
    <bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
        <!--核心线程数-->
        <property name="corePoolSize">
            <value>5</value>
        </property>
        <!--最大线程数-->
        <property name="maxPoolSize">
            <value>10</value>
        </property>
        <!--队列最大长度-->
        <property name="queueCapacity">
            <value>40</value>
        </property>
    </bean>

    <!--GetDataFromFileScheduler，获取数据的定时器-->
    <bean id="getDataFromFileScheduler" class="me.cxis.spring.scheduling.executor.GetDataFromFileScheduler">
        <property name="taskExecutor">
            <ref local="taskExecutor"/>
        </property>
    </bean>

    <!--创建统计用户的ScheduledTimerTask，描述任务怎么运行-->
    <bean id="getDataFromFileTimerTask" class="org.springframework.scheduling.timer.ScheduledTimerTask">
        <property name="delay">
            <value>10000</value>
        </property>
        <property name="period">
            <value>20000</value>
        </property>
        <property name="timerTask">
            <ref local="getDataFromFileScheduler"/>
        </property>
    </bean>

    <!--创建TimerFactoryBean，执行任务-->
    <bean id="timerFactory" class="org.springframework.scheduling.timer.TimerFactoryBean">
        <property name="scheduledTimerTasks">
            <list>
                <ref local="getDataFromFileTimerTask"/>
            </list>
        </property>
    </bean>

</beans>
```

测试类：

```
package me.cxis.spring.scheduling.executor;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

/**
 * Created by cheng.xi on 2017-04-19 15:11.
 */
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:scheduling-executor.xml");
    }
}

```
上面就是结合线程池的示例。也就是在执行任务的时候多了线程池，基本的配置方式使用方法基本没变。

# Spring3中定时器的配置

- Spring3中TaskExecutor继承了JDK的Executor。使用方面还是跟原来2.0一样。
- Spring3中还引入了新的接口TaskScheduler，Trigger，TriggerContext等。
- Spring3中还引入了task的命名空间`<task:scheduler/>`，`<task:executor/>`，`<task:scheduled-tasks/>`等。
- Spring3中还支持注解的方式`@Scheduled`，`@Async`，使配置更加简化。使用注解的方式，需要在配置文件中先开启注解支持`<task:annotation-driven/>`

注解方式的示例如下。

CountUserTask：

```
package me.cxis.spring.scheduling.annotation;


import org.springframework.scheduling.annotation.Scheduled;

/**
 * Created by cheng.xi on 2017-04-19 11:00.
 * 定时的统计信息的Task
 *
 */
public class CountUserTask{

    //@Scheduled(cron="*/5 * * * * MON-FRI") cron的方式
    //下面是固定时间，每隔5秒执行一次
    @Scheduled(fixedRate = 5000)
    public void countUser(){
        //执行真正的统计任务
        System.out.println("开始统计系统中人数");
        System.out.println("统计完成，共有101人");
    }
}

```

配置文件：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:task="http://www.springframework.org/schema/task"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
	http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
	http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task.xsd">

    <!--开启task的注解支持-->
    <task:annotation-driven />

    <!--执行任务的bean-->
    <bean id="countUserTask" class="me.cxis.spring.scheduling.annotation.CountUserTask"/>
</beans>
```

测试：

```
package me.cxis.spring.scheduling.annotation;

import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

import java.io.IOException;

/**
 * Created by cheng.xi on 2017-04-19 16:08.
 */
public class Main {
    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:scheduling-annotation.xml");
        try {
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

可以看到，使用注解的方式简化了很多很多。

# Spring4和Spring5中定时器的配置
Spring4中增加了`@EnableScheduling`注解来启用对`@Scheduled`注解的支持。其他的基本没有什么变化，使用方式还是跟以前一样，现在使用注解更多。