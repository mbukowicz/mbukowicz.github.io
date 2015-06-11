---
layout: post
title: Integrating jQuery With Selenium
date: '2012-04-02T11:54:00.000+02:00'
tags: 
modified_time: '2012-04-03T18:46:58.870+02:00'
thumbnail: http://4.bp.blogspot.com/-Eu4tPFKu5uU/T2SN7paH2CI/AAAAAAAAAD4/6d0hhgTm4g8/s72-c/JQuery_logo_color_onwhite.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-1242398963094999505
blogger_orig_url: http://www.code-thrill.com/2012/04/integrating-jquery-with-selenium.html
---
<img src="/images/integrating-jquery-with-selenium/JQuery_logo_color_onwhite.png" title="JQuery logo" class="float-left" />

<p>When testing JavaScript-heavy websites, at some point you may reach Selenium limits. A simple use case will fail. You will get <code>ElementNotVisibleException</code>, even though you can clearly see that element. Other times the same selector will return different results depending on the type of the browser. You can lose a lot of time searching for workarounds and hacks. Fortunately, a simple solution exists...</p> <a name='more'></a> <p>...but first let me introduce you to a new concept.</p> 

<div class="my-info">Not familiar with Selenium? You can grasp some knowledge <a href="/2012/03/01/first-steps-with-selenium.html">here</a> and <a href="/2012/03/15/selenium-walkthrough.html">here</a>. </div> 

<h2>JavaScript in Selenium tests</h2>
<p>Did you know you can execute JavaScript code inside a Selenium test? Here is how you can accomplish this using the <code>WebDriver</code> object:</p> 

{% highlight java %}
WebDriver driver;
// create driver, i.e. driver = new FirefoxDriver();
JavascriptExecutor js = (JavascriptExecutor) driver;
js.executeScript("alert('Hello from Selenium!');");
{% endhighlight %}

<p>The result is this pop-up:</p> 
<img src="/images/integrating-jquery-with-selenium/alert-box.jpg" title="Alert box" class="img-center" />

<p>You can go further and return a JavaScript object. It will then be transformed into a Java <code>WebElement</code> object, on which you can execute standard Selenium methods. If on a page there was a <code>DIV</code> element with ID equals "example":</p> 

{% highlight html %}
<div id="example">Hello from Selenium!</div>
{% endhighlight %}

<p>Then you could fetch that element with this piece of code:</p> 

{% highlight java %}
JavascriptExecutor js = (JavascriptExecutor) driver;
String script = "return document.getElementById('example');";
WebElement exampleDiv = (WebElement) js.executeScript(script);
exampleDiv.getText(); // returns "Hello from Selenium!"
{% endhighlight %}

<h2>jQuery to reduce verbosity</h2>
<p>Using pure JavaScript alone may be very verbose. jQuery can help here with its concise and elegant syntax. With jQuery you can: 
<ul>  
	<li>use <strong>a great deal of selectors</strong> that work pretty much the same on every browser</li>  
	<li>use <strong>jQuery functions</strong> for interaction with a page (for example to do double click use the <code>.dblclick()</code> function)</li>  
	<li>easily <strong>bypass any Selenium bug</strong> by putting JavaScript code in your tests</li>
</ul></p> 

<h2>Example scenario</h2> 
<p>Assuming we have a page for editing a list of users which looks like this:</p> 

<img src="/images/integrating-jquery-with-selenium/list_of_users_html.jpg" title="HTML for list of users" class="img-center" />

<p>...with <code>HTML</code> code looking like this:</p> 

{% highlight html %}
<table id="users">
    <tr>
        <th>Name</th>
        <th>Remove</th>
    </tr>
    <tr>
        <td>John</td>
        <td>
            <button onclick="removeUser('John')">
                Remove
            </button>
        </td>
    </tr>
    <tr>
        <td>Bob</td>
        <td>
            <button onclick="removeUser('Bob')">
                Remove
            </button>
        </td>
    </tr>
    <tr>
        <td>Frank</td>
        <td>
            <button onclick="removeUser('Frank')">
                Remove
            </button>
        </td>
    </tr>
</table>
{% endhighlight %}

<p>Lets try to test the following scenario: 
<ol>  
	<li>Web browser is opened.</li>  
	<li>Page with a list of users is shown.</li>  
	<li>User 'Bob' is removed from the list by clicking the 'Remove' button in front of his name.</li>
</ol></p> 

<div class="my-info">Want to try <strong>the example code</strong>? You can download <a href="https://github.com/mbukowicz/Learning-Selenium">this github project</a> (check the <code>selenium3</code> package). </div> 

<p>This short video demonstrates what we are trying to accomplish:</p> 
<iframe width="480" height="360" src="http://www.youtube.com/embed/BNvDV8FLRVA?rel=0" frameborder="0" allowfullscreen></iframe> 

<h2>Clicking a button with jQuery</h2> 
<p>First we should come up with a proper jQuery selector:</p>

{% highlight text %}
#users tr:has(td:contains('Bob')) button:contains('Remove')
{% endhighlight %}

<p>Here is an explanation how this selector works:</p> 

<img src="/images/integrating-jquery-with-selenium/remove_user_btn_selector.jpg" title="Explained selector for remove user button" class="img-center" /> 

<p>To click the button that is returned as a result of our selector we can use this piece of code:</p> 

{% highlight java %}
String jQuerySelector = "#users " +
                        "tr:has(td:contains('Bob')) " + 
                        "button:contains('Remove')";
js.executeScript("$(\"" + jQuerySelector + "\").click();");
{% endhighlight %}

<p>If we want to do the clicking part on the Selenium side we should write this instead:</p> 

{% highlight java %}
String jQuerySelector = "#users " +
                        "tr:has(td:contains('Bob')) " + 
                        "button:contains('Remove')";
String findButton = "return $(\"" + jQuerySelector + "\").get(0);";
WebElement removeButton = (WebElement) js.executeScript(findButton);
removeButton.click();
{% endhighlight %}

<p>The jQuery <code>get</code> method is used to unwrap the jQuery decorator object and return the native DOM object. Without parameters this method returns a list of unwrapped objects. Only one <code>BUTTON</code> is expected to be returned, so it is more appropriate to use the <code>get</code> method with an integer parameter. This parameter specifies which element from the list should be returned. In our case it is the first element, so we can use the method call: <code>.get(0)</code>.</p> 

<h2>jQuery selectors using the <code>Selenium</code> class</h2>
<p>If you want to <strong>use the jQuery selectors alone</strong>, a sensible solution is to use the <code>com.thoughtworks.selenium.Selenium</code> class. It provides lots of helper methods for interacting with a website. Creating the <code>Selenium</code> object is as simple as wrapping the <code>WebDriver</code> object:</p> 

{% highlight java %}
String baseUrl = ...; // i.e. "http://google.com"
Selenium selenium = new WebDriverBackedSelenium(driver, baseUrl);
{% endhighlight %}

<p>Again, if we want to remove the user named <strong>Bob</strong>, we should prepend the previous selector with the type "<cite>css=</cite>" and put the result as a parameter to the <code>click</code> method of the <code>Selenium</code> object:</p> 

{% highlight java %}
selenium.click("css=#users " + 
               "tr:has(td:contains('Bob')) " + 
               "button:contains('Remove')");
{% endhighlight %}

<h2>Things to consider</h2>
<p>We reached the outcome we wanted, but there are still some points we should consider: 
<ul>  
	<li>What to do if we are not using jQuery on our website and are unable to use it? (we are either constrained by requirements or have no access to the sources)</li>  
	<li>What are the drawbacks of this technique?</li>
</ul></p> 

<h2>Loading jQuery from within a test</h2>
<p>If your <strong>site does not use jQuery</strong> and you do not want or cannot put the appropriate script tag, you can still use it by: 
<ul>  
	<li>loading the contents of the <strong>jQuery code into a <code>String</code></strong> from a JavaScript file (<code>jquery.js</code>, <code>jquery.min.js</code> or similar)</li>  
	<li>...and executing this <code>String</code> as JavaScript code using the <code>WebDriver</code> object</li>
</ul>
To load the jQuery file into a <code>String</code> you can use the Guava library. In case you were not using it already, here is the Maven dependency:</p>

{% highlight xml %}
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>11.0.2</version>
</dependency>
{% endhighlight %}

<p>To fetch the contents of a file the <code>com.google.common.io.Resources</code> class proves to be useful. Here is how you can use it to load jQuery from within a test:</p>

{% highlight java %}
URL jqueryUrl = Resources.getResource("jquery.min.js");
String jqueryText = Resources.toString(jqueryUrl, Charsets.UTF_8);
JavascriptExecutor js = (JavascriptExecutor) driver;
js.executeScript(jqueryText);
{% endhighlight %}

<h2>Avoid the side effects</h2>
<p>Using jQuery with Selenium is not always a bed of roses. It can significantly <strong>simplify your tests</strong> or solve your problem, but you should take into consideration <strong>the impact of injecting JavaScript code</strong> into a tested website. When using this technique think carefully whether it: 
<ul>  
	<li>changes the website's flow</li>  
	<li>may hide a bug</li>  
	<li>enables access to an element that is otherwise not visible</li>  
	<li>tides your test with the implementation of the website</li>
</ul></p> 

<h2>Final thoughts</h2>
<p>You have aquired a new tool in your toolbox. It can <strong>improve the development</strong> of Selenium tests. On the other hand it <strong>should not be overused</strong>. Understand the potential side effects and try to avoid them.</p>