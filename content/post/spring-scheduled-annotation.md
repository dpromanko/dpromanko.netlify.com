---
title: "My Java Spring @Scheduled method stopped running in production"
tags: ["Java", "Spring"]
date: 2019-05-20T18:32:19-04:00
draft: false
---

### TLDR
Hopefully you can say none of this is a surprise to you because you read the documentation on Spring's [EnableScheduling](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html). Or maybe like me, you just started using it. Either way I hope you can learn something from my experience and not run into similar issues yourself. If you want to skip the story about my @Scheduled methods not running in production, you can jump right to [what I learned after reading the documentation](#the-documentation) and [how you can prevent this from happening](#how-do-we-fix-it).

### The first time
The first time I encountered this issue was an @Scheduled annotation wrapping a method that performed a long(ish) running query. For example:

{{< highlight java >}}
@Scheduled(fixedRate = 600000)
public void scheduledMethod() {
  System.out.println("I am running");
  performSomeLongRunningQuery();
  System.out.println("I am done");
}
{{< / highlight >}}

Works great, right? For months in production my query had been running every 10 minutes as I so clearly told it to until one day I got a call that indicated to me that is was no longer running. Sure enough after checking the logs of this fake code above I saw a message stating "I am running" and it never finished. To my knowledge, once this happens to a snippet such as the one above your only option is to restart the application. As expected after a restart we were back up and running every 10 minutes. So what went wrong? I was able to confirm that the query did indeed finish executing on the database, so why didn't it return? I won't get into details, but someone was changing something somewhere at the same time my query was running and I believe it broke the connection. At this point I figured we just got lucky (or unlucky) and I would just forget about it.

### The second time
The second time I encountered this issue (in a different application) was an @Scheduled annotation wrapping a method that performed an api call. Another developer and I were looking into this and I immediately knew what was wrong (although this time I had no idea what caused it) and how to fix it. We restarted the application and everything was back to normal. I should also note, this application has more than just one @Scheduled method and we noticed those were no longer running either.

### Experiment with behavior
Clearly now it was time for me to better understand what is actually going on under the hood of the @Scheduled annotation. I wrote the following proof of concept:

{{< highlight java >}}
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
      // TODO Auto-generated catch block
      e.printStackTrace();
    }
    sleepy();
  }

}
{{< / highlight >}}

This example demonstrates both scenarios I ran into. I called something external that never returned and more importantly never threw an exception. Simply adding
{{< highlight java >}}
throw new Exception();
{{< / highlight >}}
anywhere inside the method `sleepy()` will cause the behavior I want/expect. When the exception is thrown the the scheduled method will complete and be able to run again.

Even more to my surprise if you add another @Scheduled method, it won't run while `scheduledMethod()` is running forever. I had never read the documentation on @EnableScheduling so I didn't know why this was the case.

### The documentation
{{< tweet 1060067235316809729 >}}

Taken from the Spring [EnableScheduling](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html) documentation:

>By default, will be searching for an associated scheduler definition: either a unique TaskScheduler bean in the context, or a TaskScheduler bean named "taskScheduler" otherwise; the same lookup will also be performed for a ScheduledExecutorService bean. If neither of the two is resolvable, a local single-threaded default scheduler will be created and used within the registrar.

Takeaways from the above documentation:

1. The default scheduler is only allotted one thread, which is why the method above will only run once.
2. All @Scheduled annotations share the same thread pool, which means all other @Scheduled annotations would be blocked from executing in the above example.

### How do we fix it
Based on my two scenarios above, we have two problems to fix. How do we run multiple @Scheduled methods simultaneously and how do we make sure a single @Scheduled method will complete so it can run again.

#### How many @Scheduled methods could be executing simultaneously?
If you only have one @Scheduled method in your application then technically you are fine with leaving it with the default single thread. Although, I'm willing to bet when you add another you won't remember to increase the thread pool so it is probably worth doing it now.

If more than one @Scheduled method in your application could be overlap, then you should look at implementing [SchedulingConfigurer](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/SchedulingConfigurer.html). As you can see in the example below you will implement this on a class that is annotated with @Configuration and @EnableScheduling. One thing to note about the example below is they are using 100 as the thread pool size. To my knowledge the number here doesn't really matter as long as it is at least the number of @scheduled methods that could overlap, that way no @Scheduled method is ever waiting on another to finish before running.

Taken from the Spring [EnableScheduling](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/scheduling/annotation/EnableScheduling.html) documentation:

{{< highlight java >}}
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
{{< / highlight >}}

#### Is there any chance your @Scheduled method will run forever?
This one is on you to figure out based on what you are doing in each of your @Scheduled methods. As I showed with the `sleepy()` example, if the method never finishes it will never run again. At this point I think you could come to a conclusion about why my @Scheduled methods stopped running in production in the two brief stories I tell. In both it seems that a timeout configuration was missing so the underlying logic never threw an exception and was just waiting for a response.


### The end
If you made it this far, thanks for reading my first technical blog post! I acknowledge this is nothing profound and nothing you couldn't learn from reading the documentation. I just wanted to share a little story about something that broke in production and what I learned from it.

If you think any of this information is wrong, have questions/comments, or have any feedback then please reach out to me on [Twitter](https://twitter.com/dpromanko)!