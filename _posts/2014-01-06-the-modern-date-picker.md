---
layout: post
title: The Modern Date Picker
date: '2014-01-06T17:44:00.002+01:00'
tags: 
modified_time: '2014-01-06T17:55:38.190+01:00'
thumbnail: http://4.bp.blogspot.com/-fpbQiy6ucz4/Usrc8Ewd9MI/AAAAAAAAALY/SmU-5TTQ724/s72-c/calendar.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-309088979003843858
blogger_orig_url: http://www.code-thrill.com/2014/01/the-modern-date-picker.html
---
<h2>Introduction</h2> 

<img title="Calendar icon" src="/images/modern-date-picker/calendar.png" class="float-left" /> 

<p>In this post I'll show a simple way of handling date inputs that will work across modern and not so modern browsers. You'll also learn how to implement a mobile-friendly date picker in jQuery Mobile.</p> 

<div class="my-info" style="float: left">You can download the source codes from <a href="https://github.com/mbukowicz/modern-date-picker">this GitHub repo</a>. </div>

<h2 style="clear: both;">Good ol' date picker</h2> 

<p>The standard approach for giving the users a way to choose a date used to be: 
	<ol>  
		<li>Add an ordinary text input to your HTML 
{% highlight html %}
<input type="text" id="date-picker-here" />
{% endhighlight %}
		</li>  
		<li>Enhance it with JavaScript (my personal favourite is jQuery UI Date Picker) 
{% highlight javascript %}
$(function() {
  $("#date-picker-here").datepicker();
});
{% endhighlight %}
		</li>
	</ol>
</p> 

<p>As a result you would get a date picker that looks like this:</p> 

<img title="jQuery UI date picker" src="/images/modern-date-picker/jquery-ui-datepicker.jpg" class="img-center" /> 

<h2>HTML5 date input</h2> 

<p>With the introduction of HTML5, we got a new input type, the <a href="http://dev.w3.org/html5/markup/input.date.html#input.date">date input</a>:</p> 

{% highlight html %}
<input type="date" />
{% endhighlight %}

<p>It's as simple as that. The way the native date picker looks depends on the browser implementation. In Chrome it looks something like that:</p> 

<img title="HTML5 date picker" src="/images/modern-date-picker/native-date-picker.jpg" class="img-center" /> 

<p>The only problem with the native date input is that it's <a href="http://caniuse.com/input-datetime">not supported in all browsers and platforms</a>. Currently, you are forced to provide a fallback of some sort.</p> 

<h2>Modernizr to the rescue</h2> 

<p>To detect, if user's browser supports date inputs <a href="http://modernizr.com/">Modernizr</a> seems like the best choice. The availability of date inputs can be checked by testing the flag:</p> 

{% highlight javascript %}
Modernizr.inputtypes.date
{% endhighlight %}

<p>The whole code could look like this:</p> 

{% highlight javascript %}
var nativeDateInputIsSupported = Modernizr.inputtypes.date;
if (!nativeDateInputIsSupported) {
  $("input[type='date']").datepicker();            
}
{% endhighlight %}

<h2>What about mobile?</h2> 

<p>How about mobile devices? Unfortunately, the standard jQuery Date Picker is not very mobile-friendly. If you're using <a href="http://jquerymobile.com/">jQuery Mobile</a> you may find <a href="http://mobipick.sustainablepace.net/">Mobi Pick</a> the simplest and easiest to configure.</p> 

<p>The code behind is similar to that of jQuery UI:</p> 

{% highlight javascript %}
$(document).on("pagecreate", function() {
  $("#date-picker-here").mobipick();
});
{% endhighlight %}

<p>And here is the result:</p> 

<img title="Mobi Pick date picker" src="/images/modern-date-picker/mobi-pick.jpg" class="img-center" /> 

<h2>Mobile with fallback</h2> 

<p>The Mobi Pick date picker looks nice, however users probably expect a behavior similar to that of the platform they're using. This can be accomplished with the native date inputs. Again, we should provide a fallback. Let's use Mobi Pick for this:</p>

{% highlight javascript %}
$(document).on("pagecreate", function() {
  var nativeDateInputIsSupported = Modernizr.inputtypes.date;
  if (!nativeDateInputIsSupported) {
    $("input[type='date']").mobipick();            
  }
});
{% endhighlight %}

<h2>The End</h2> 

<p>I hope you enjoyed this post. Please, check <a href="https://github.com/mbukowicz/modern-date-picker">this GitHub repo</a>, if you would like to download the examples.</p>