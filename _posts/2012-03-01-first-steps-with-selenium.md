---
layout: post
title: First Steps With Selenium
date: '2012-03-01T08:41:00.000+01:00'
tags:
modified_time: '2012-03-11T20:58:03.785+01:00'
thumbnail: /images/first-steps-with-selenium/thumbnail.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-2734477711125479034
blogger_orig_url: http://www.code-thrill.com/2012/03/first-steps-with-selenium.html
---

<img src="/images/first-steps-with-selenium/selenium-logo.png" title="Selenium logo" class="float-left" />
<p>This is a short guide to getting started with Selenium - an automated website testing library.</p>

<h2>What you will learn</h2>
<p>You will learn how to automatically test a simple web form using Selenium. Examples will be created using  Java and Maven. If you would like to follow along with this tutorial, you can <a href="https://github.com/mbukowicz/Learning-Selenium">download this project from github</a> (look for the package called <code>selenium1</code>).</p>

<h2>What is Selenium?</h2>
<p>Selenium is a library for interacting with a web browser. It enables you to simulate how users may use your site, like filling up forms for example. What is important about Selenium - it can test JavaScript code. You can open up a browser of your choice and enter any site, even with advanced scripting. In the next sections we will dive into details how to do it.</p>

<h2>Maven Dependency</h2>
<p>First you need to add a dependency to the <code>pom.xml</code>:</p>
{% highlight xml %}
<dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>2.15.0</version>
</dependency>
{% endhighlight %}

<h2>Creating a WebDriver</h2>
<p>Next you should create a <code>WebDriver</code> object. <code>WebDriver</code> is an interface representing a general browser. For specific browsers you have to use concrete implementations. Currently supported out-of-the-box are:
<ul>  
	<li>Chrome</li>  
	<li>Internet Explorer</li>  
	<li>Firefox</li>  
	<li>Opera</li>
</ul></p>

<p>For starting Firefox you would write:</p>
{% highlight java %}
WebDriver driver = new FirefoxDriver();
{% endhighlight %}

<p>To open a specific page use the <code>get</code> method:</p>
{% highlight java %}
driver.get("http://google.com");
{% endhighlight %}

<h2>Simple Scenario</h2>
<p>Supposedly, we have a form like:</p>
{% highlight html %}
<form>
    <label>User:</label>
    <input name="user" type="text" />

    <label>Password:</label>
    <input name="password" type="password" />

    <input id="login" type="submit" value="Log in" />
</form>
{% endhighlight %}

<p>To mimic a user logging into our application we could use this code:</p>
{% highlight java %}
WebElement user = driver.findElement(By.name("user"));
user.sendKeys("Admin");

WebElement password = driver.findElement(By.name("password"));
password.sendKeys("HackMe!");

WebElement login = driver.findElement(By.id("login"));
login.click();
{% endhighlight %}

<p>Three important things happen here:
<ul>  
	<li>entering text into fields using <code>sendKeys</code> method</li>  
	<li>clicking the "Log in" button with <code>click</code> method</li>  
	<li>looking up web elements in the page structure (in the DOM model to be specific)</li>
</ul></p>

<p>Entering text and clicking is quite straightforward, however finding web elements can be quite complex.  You can specify search criteria by:
<ul>  
	<li><code>By.id(String id)</code></li>  
	<li><code>By.name(String name)</code></li>  
	<li><code>By.tagName(String tagName)</code></li>  
	<li><code>By.className(String className)</code></li>  
	<li><code>By.linkText(String linkText)</code> - will search for the "<code>a</code>" HTML element with the given text</li>  
	<li><code>By.partialLinkText(String linkText)</code> - same as above, but even partial match will succeed</li>  
	<li><code>By.xpath(String xpathExpression)</code></li>  
	<li><code>By.cssSelector(String selector)</code></li>
</ul></p>

<p>For simple look ups, choice of a specific method will be quite obvious. Most of the time you will use <code>By.id</code>, <code>By.name</code> or <code>By.className</code>.</p><p>For the non-trivial cases you will have to choose between <code>xpath</code> and <code>css</code> selectors. It is best to decide based on your knowledge and complexity of the result query. Sometimes xpath will be simpler, sometimes a css selector will work out better.</p>

<h2>The End</h2>
<p>Here ends this quick tour on Selenium basics. In further articles I will try to expand the topic with more complex scenarios.</p>
