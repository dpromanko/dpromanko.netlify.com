---
title: "My Java Spring @Scheduled methods stopped running in production"
tags: ["Java", "Spring"]
date: 2019-05-20T18:32:19-04:00
draft: false
---

### The first time
The first time I encountered this issue was an @Scheduled annotation wrapping a method that performed a long(ish) running query. For example:

{{< code language="java" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
@Scheduled(fixedRate = 600000)
public void scheduledMethod() {
  System.out.println("I am running");
  performSomeLongRunningQuery();
  System.out.println("I am done");
}
{{< /code >}}

This works great, right? For months in production, my query had been running every 10 minutes as I so clearly told it to until one day I got a call that indicated to me that is was no longer running. Sure enough, after checking the logs of this code above I saw a message stating "I am running", but there was no "I am done" message. To my knowledge, once this happens to a snippet such as the one above your only option is to restart the application. As expected after a restart we were back up and running every 10 minutes. So what went wrong? I was able to confirm that the query did indeed finish executing on the database, so why didn't it return? I won't get into details, but I believe the connection to the database was severed. At this point, I figured we just got lucky (or unlucky) and I forgot about it.

### The second time
The second time I encountered an @Scheduled method that stopped running (in a different application) was an @Scheduled annotation wrapping a method that performed an API call. Another developer and I were looking into this and I immediately knew what was wrong and how to fix it, although this time I had no idea what caused it. We restarted the application and everything was back to normal. I should also note, this application has more than just one @Scheduled method and we noticed those were no longer running either.

### Time to figure out what happened
Clearly now it was time for me to better understand what is actually going on under the hood of the @Scheduled annotation. As soon as I got home from work I started playing around with the little demo below.

{{< code language="java" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
public class DemoSchedule {

  @Scheduled(fixedRate = 1000)
  public void scheduledMethod() {
    System.out.println("I am only going to run once");
    sleepy();
  }

  public void sleepy() {
    System.out.println("sleeping");
    try {
      Thread.sleep(60000);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    sleepy();
  }

}
{{< /code >}}

This example demonstrates what happens to an @Scheduled method that never returns and more importantly never throws an exception. Simply adding

{{< code language="java" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
throw new Exception();
{{< /code >}}

anywhere inside the method `sleepy()` will cause the behavior I want/expect. When the exception is thrown the scheduled method will complete and run again.

Even more to my surprise if you add another @Scheduled method, it won't run while `scheduledMethod()` is running forever. I didn't know why this was the case because I never looked at the documentation for @EnableScheduling.

### The documentation

Taken from the Spring [EnableScheduling](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html) documentation:

>By default, will be searching for an associated scheduler definition: either a unique TaskScheduler bean in the context, or a TaskScheduler bean named "taskScheduler" otherwise; the same lookup will also be performed for a ScheduledExecutorService bean. If neither of the two is resolvable, a local single-threaded default scheduler will be created and used within the registrar.

Takeaways from the above documentation:

1. The default scheduler is only allotted one thread, which is why the method above will only run once.
2. All @Scheduled annotations share the same thread pool, which means all other @Scheduled annotations would be blocked from executing in the above example.

{{< tweet 1060067235316809729 >}}

### How many @Scheduled methods could be executed simultaneously?
If you only have one @Scheduled method in your application then technically you are fine with leaving it with the default single thread. Although, I'm willing to bet when you add another you won't remember to increase the thread pool so it is probably worth doing it now.

If more than one @Scheduled method in your application could be overlap, then you should look at implementing [SchedulingConfigurer](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/SchedulingConfigurer.html). As you can see in the example below you will implement this on a class that is annotated with @Configuration and @EnableScheduling. One thing to note about the example below is they are using 100 as the thread pool size. To my knowledge, the number here doesn't really matter as long as it is at least the number of @Scheduled methods that could overlap, that way no @Scheduled method is ever waiting on another to finish before running.

Taken from the Spring [EnableScheduling](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html) documentation:

{{< code language="java" id="1" expand="Show" collapse="Hide" isCollapsed="false" >}}
@Configuration
@EnableScheduling
public class AppConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskExecutor());
    }

    @Bean(destroyMethod="shutdown")
    public Executor taskExecutor() {
        return Executors.newScheduledThreadPool(100);
    }
}
{{< /code >}}

### Is there any chance your @Scheduled method will never exit?
This one is on you to figure out based on what you are doing in each of your @Scheduled methods. As I showed with the `sleepy()` example, if the method never returns, or throws an exception, it will never run again. Both times my @Scheduled methods stopped running in production could have been prevented with the proper timeout configurations on the datasource/HTTP client.

### The end
If you made it this far, thanks for reading my first technical blog post! I acknowledge this is nothing profound and nothing you couldn't learn from reading the documentation. I just wanted to share a little story about something that broke in production and what I learned from it.

If you think any of this information is wrong, have questions/comments, or have any feedback then please reach out to me on [Twitter](https://twitter.com/dpromanko)!

