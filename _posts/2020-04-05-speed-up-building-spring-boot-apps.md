---
layout: post
title:  "Speed up building Spring Boot apps"
date:   2020-04-05 16:11:19
categories: java spring-boot
---

<img src="/images/speed-up-building-spring-boot-apps/rocket-launch-67643_1280.jpg" title="Rocket launch your Spring Boot development process" class="float-left" />

The goal
-----------------

It is the ultimate goal of most programmers to __become more efficient
and build software quicker__.
One way to improve is to learn to type
faster, which obviously helps but will not get you far. A probably
better approach would be to expose yourself to harder problems so that you
gain experience and next time you stumble upon the same problem you already
have an idea how to solve it.

However, in the world of compiled languages there is one common hurdle we all
need to overcome, and that is the __long feedback loop__ (fix, save, build,
redeploy, wait, check result, repeat).

A low-hanging fruit
-------------------

If you are building your Java backend applications using Spring Boot then there is
a quick and easy step you can take to immediately make this feedback loop faster.

Instead of manually restarting your application every time you do a change:

<img src="/images/speed-up-building-spring-boot-apps/restart-icon.png"
title="Restart in IntelliJ" style="clear: both;" />

...you can just add this dependency and get automatic app reload for free:

{% highlight xml %}
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-devtools</artifactId>
  <optional>true</optional>
  <!--
    make sure it is marked as optional
    (otherwise it will be bundled up with your app)
  -->
</dependency>
{% endhighlight %}

{% highlight groovy %}
dependencies {
  developmentOnly("org.springframework.boot:spring-boot-devtools")
}
{% endhighlight %}

Now, when you change your code and rebuild the project the application will
automatically restart.

<img src="/images/speed-up-building-spring-boot-apps/devtools-app-reload.gif"
title="Restart in IntelliJ" style="clear: both;" />

What's cool is that this __automatic restart should be much faster__ because it
reloads only your own classes while keeping other non-project classes intact.

It is achieved by maintaining 2 separate classloaders:

* base classloader which loads classes outside your project (like all those classes
  from the dependency jars) - this classloader is preserved between restarts
* second classloader holds your project classes and is thrown away whenever
  restart occurs


<div class="my-info">
Spring Boot is highly customizable and so are Developer Tools.
You can learn more from <a href="https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-devtools"> the official documentation</a>.
</div>

Live-reload in the browser
--------------------------

Another time saver is the live-reload feature which refreshes the page
after the server restarts. In layman's terms, you __no longer need to hit the
refresh button__ in your browser.

All you need is install a live-reload plugin, like this one for Chrome:

<img src="/images/speed-up-building-spring-boot-apps/live-reload-plugin.png"
title="LiveReload Chrome plugin" style="clear: both;" />

After you enable the live-reload feature on your site:

<img src="/images/speed-up-building-spring-boot-apps/enable-live-reload.gif"
title="Restart in IntelliJ" style="clear: both;" />

...you can enjoy having the page refreshed for you:

<img src="/images/speed-up-building-spring-boot-apps/live-reload-in-action.gif"
title="Live reload in action" style="clear: both;" />


Squeeze out even more time
--------------------------

If you are a performance-fixated maniac constantly obsessed with improving
your daily workflow then sooner or later you will start getting annoyed by
having to wait several seconds for the restart to complete. This wait time
increases as the project grows larger and could get out of hand.

Fortunately, there are more options you can consider. For example, you can look
into the realm of JVM class reloading. Basically, this means using tools
that are replacing bytecode in the JVM while it is still running.

Reloading classes for free
--------------------------

There is __a cost free solution__ to reload your application in almost an instant.
It works surprisingly well and is called [DCEVM](https://dcevm.github.io), which
stands for Dynamic Code Evolution Virtual machine.

In order to use it you just replace your current JDK with an alternative
JDK. It all boils down to downloading a certain version of DCEVM JDK (as of time
of writing this blog post Java 8 and 11 are supported), unpacking it to a
folder and then configuring it in your IDE:

<img src="/images/speed-up-building-spring-boot-apps/dcevm-intellij-config.png"
title="DCEVM in IntelliJ" style="clear: both;" />

<img src="/images/speed-up-building-spring-boot-apps/dcevm-intellij-project-config.png"
title="DCEVM in IntelliJ" style="clear: both;" />

Once you have that you just need to create <a href="http://hotswapagent.org/mydoc_configuration.html">hotswap-agent.properties</a> within _src/main/resources_,
then fill it with the following content:

{% highlight properties %}
# hotswap-agent.properties
autoHotswap=true
{% endhighlight %}

...and Bob's your uncle. Now you have live class reload:

<img src="/images/speed-up-building-spring-boot-apps/dcevm-reload-in-action.gif"
title="DCEVM reload in action" style="clear: both;" />

If you are using Spring Boot in version 2.1.0 and above
then you can easily confirm that the configuration was successful.

You should observe the following statement in your logs
(here is <a href="https://github.com/spring-projects/spring-boot/pull/14807">the corresponding pull request</a> for reference) about the restart being
suspended in favour of DCEVM reloads:

{% highlight plain %}
[main] INFO org.springframework.boot.devtools.restart.RestartApplicationListener - Restart disabled due to an agent-based reloader being active
{% endhighlight %}

Is it any good?
---------------

Although DCEVM does not cost a dime it works pretty well and ships with a lot
of plugins that help with reloading changes related to a particular Java
library.

<div class="my-info">
Here you can check <a href="http://hotswapagent.org/index.html#plugins">
the list of available plugins</a> - it basically covers most of the major Java
frameworks. There is even <a href="http://hotswapagent.org/mydoc_plugin_spring.html">a DCEVM
plugin dedicated for Spring framework</a> which automatically reloads Spring's
application context.
</div>

Not so free alternatives
------------------------

Even though DCEVM is great, configuring it for a legacy project can be quite
challenging. If you have to make it work with a brown-field application then
you should accept the likelihood that you are going to get stuck.

There is, however, an alternative and it is called JRebel. It is hassle-free
and usually works out-of-the-box. The "only" downside is that it is quite
pricey so you have to weigh up the pros and cons. If you can afford it (or
stretch your project's budget) then it is definitely worth considering.

<div class="my-info">
Please check <a href="/2013/04/23/practical-introduction-to-jrebel.html">
this post about JRebel</a> if you would like to learn more.
</div>
