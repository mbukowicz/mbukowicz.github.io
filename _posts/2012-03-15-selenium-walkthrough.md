---
layout: post
title: Selenium Walkthrough
date: '2012-03-15T01:05:00.001+01:00'
tags: 
modified_time: '2012-03-15T21:24:27.027+01:00'
thumbnail: http://2.bp.blogspot.com/--B6cd_oPM1k/T10HFnNgfWI/AAAAAAAAAC4/Ik3rJgWvuuQ/s72-c/selenium2-browsers.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-1851511191782133696
blogger_orig_url: http://www.code-thrill.com/2012/03/selenium-walkthrough.html
---

<img src="/images/selenium-walkthrough/selenium2-browsers.png" title="Selenium browsers" class="float-left" />
<p>This tutorial will get you up and running with the Selenium WebDriver API. If you want to catch up on the Selenium basics try reading <a href="/2012/03/15/selenium-walkthrough.html">this introductory material</a>.</p> 

<h2 style="clear: both">What you will learn</h2>
<p>You will learn how to mimic usage of the following HTML elements:  
<ul>  
	<li>A link</li>  
	<li>BUTTON</li>  
	<li>checkbox</li>  
	<li>SELECT combo box</li>  
	<li>alert box</li>  
	<li>TABLE</li>
</ul>
If you want to get straight to the code, feel free to skip the article and download <a href="https://github.com/mbukowicz/Learning-Selenium">this github project</a> (look for the package called <code>selenium2</code>).</p>

<h2>Example Website</h2>
<p>Suppousedly, we have built an admin panel for managing users, which looks like this:</p> 
<img src="/images/selenium-walkthrough/SeleniumExampleSite.jpg" title="Selenium example site" class="img-center" />

<p>With the following HTML code behind:</p> 
{% highlight html %}
<h2>Users</h2>
<div id="filter-by-role">Show only users with role:<br />
    <input type="checkbox" checked="checked" value="Admin" />Admin<br />
    <input type="checkbox" checked="checked" value="Moderator" />Moderator<br />
    <input type="checkbox" checked="checked" value="User"/>User
</div>
<table>
    <tr>
        <th>Id</th>
        <th>Email</th>
        <th>Role</th>
        <th>Action</th>
    </tr>
    <tr>
        <td>1</td>
        <td>testuser1@example.com</td>
        <td>
            <select id="role_1">
                <option>Admin</option>
                <option>Moderator</option>
                <option>User</option>
            </select>
        </td>
        <td><a href="#">Edit</a> <a href="#">Remove</a></td>
    </tr>
    <tr>
        <td>2</td>
        <td>testuser2@example.com</td>
        <td>
            <select id="role_2">
                <option>Admin</option>
                <option>Moderator</option>
                <option>User</option>
            </select>
        </td>
        <td><a href="#">Edit</a> <a href="#">Remove</a></td>
    </tr>
 
    <!-- for clarity other users commented out -->
 
</table>
<div id="save-discard">
    <button>Save</button>
    <button>Discard</button>
</div>
{% endhighlight %}

<p>We would like to test a scenario, where: 
<ol>  
	<li>User opens the site, enters admin panel.</li>  
	<li>User wants to filter out all admin users, so he unchecks the "Admin" checkbox.</li>  
	<li>User promotes the third user (id=3) to being a "Moderator".</li>  
	<li>User deletes the fourth user (id=4) by clicking "Remove".</li>  
	<li>Finally, user clicks the "Save" button and confirms on the alert box.</li>
</ol></p> 

<h2>Expected Result</h2>
<p>After the scenario admin panel should look like this:</p> 
<img src="/images/selenium-walkthrough/SeleniumExampleSiteResult.jpg" title="Selenium expected result" class="img-center" />

<p>Here is a recording of the scenario:</p> 
<iframe width="480" height="360" src="http://www.youtube.com/embed/yMJ5ppqbOVA" frameborder="0" allowfullscreen></iframe> 

<h2>Unchecking the checkbox</h2>
<p>We would like to uncheck the "Admin" checkbox:</p> 
{% highlight html %}
<div id="filter-by-role">Show only users with role:<br />
    <input type="checkbox" checked="checked" value="Admin" />Admin<br />
    <input type="checkbox" checked="checked" value="Moderator" />Moderator<br />
    <input type="checkbox" checked="checked" value="User"/>User
</div>
{% endhighlight %}

<p>So we lookup the INPUT with value "Admin" inside the <code>filter-by-role</code> DIV:</p> 
{% highlight java %}
WebElement adminCheckbox;
// CSS Selector
adminCheckbox = driver.findElement(By.cssSelector("#filter-by-role input[value=Admin]"));
// ...or equivalent XPath
adminCheckbox = driver.findElement(By.xpath("//div[@id='filter-by-role']/input[@value='Admin']"));
adminCheckbox.click();
{% endhighlight %}

<p>Here is a small explanation of the used selectors:</p> 
<img src="/images/selenium-walkthrough/unchecking_checkbox_selectors.jpg" title="Unchecking the checkbox selector explanation" class="img-center" />

<p>It is up to you, whether you choose XPath or CSS selectors, but sometimes one works better than the other. We will learn about it in a moment.</p> 

<h2>Selecting option in the combo box</h2> 
<p>We want to select "Moderator" for the user with id = 3:</p> 
{% highlight html %}
<select id="role_3">
    <option>Admin</option>
    <option>Moderator</option>
    <option selected="selected">User</option>
</select>
{% endhighlight %}

<p>So we find a <code>WebElement</code> with id <code>role_3</code> and wrap it inside a <code>Select</code> object:</p>
{% highlight java %}
Select roleSelect = new Select(driver.findElement(By.id("role_3")));
roleSelect.selectByVisibleText("Moderator");
{% endhighlight %}

<h2>Clicking the "Remove" link</h2>
<p>We would like to click the remove link in the fourth row (user with id = 4).</p> 
{% highlight html %}
<tr>
    <td>4</td>
    <td>testuser4@example.com</td>
    <td>
        <select id="role_4">
            <option>Admin</option>
            <option selected="selected">Moderator</option>
            <option>User</option>
        </select>
    </td>
    <td><a href="#">Edit</a> <a href="#">Remove</a></td>
</tr>
{% endhighlight %}

<p>Unfortunately, we cannot use any <code>By.id</code> selector, so we have to choose: 
<ol>  
	<li>We come up with a clever selector or...</li>  
	<li>We end up with foreach loops searching for the row with id = 4 and link with text "Remove".</li>
</ol>
Again, it depends on our preferences. However, I think here it is best to use some XPath goodness:</p> 
{% highlight java %}
String removeXpath =
        "//table" +                    // any table                   
        "//tr[contains(td[1], '4')]" + // any tr that contains '4' in first td
        "/td" +                        // a single td
        "/a[text() = 'Remove']";       // a link containing 'Remove'
WebElement removeLink = driver.findElement(By.xpath(removeXpath));
removeLink.click();
{% endhighlight %}

<p>To better understand this XPath selector take a look at this explanation [click to enlarge]:</p> 
<img src="/images/selenium-walkthrough/remove_link_selector.jpg" title="Remove link selector explanation" class="img-center" /> 

<h2>Clicking the "Save" button and confirming changes</h2> 
<p>Finally, we click the "Save" button and confirm our changes:</p> 
{% highlight html %}
<div id="save-discard">
    <button>Save</button>
    <button>Discard</button>
</div>
{% endhighlight %}

<p>Clicking the button is very straightforward:</p> 
{% highlight java %}
WebElement saveButton = driver.findElement(
        By.cssSelector("#save-discard button:first-child"));
saveButton.click();
{% endhighlight %}

<p>Fortunately, alert boxes are also simple to use. When we encounter the invocation of a JavaScript <code>confirm("Save changes?");</code> function, we have to <strong>switch</strong> to the alert window and then <strong>accept</strong> it:</p> 
{% highlight java %}
Alert alert = driver.switchTo().alert();
alert.accept();
{% endhighlight %}

<p>It will act like user clicked the OK button on the pop-up window. If you would like to cancel instead, use the <code>Alert.dismiss()</code> method.</p> 

<h2>The End</h2>
<p>Here ends our quick tour. I hope you enjoyed learning about the Selenium WebDriver API.</p>