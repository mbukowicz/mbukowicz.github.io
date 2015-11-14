---
layout: post
title:  "Comparing BigDecimals"
date:   2015-11-14 12:00:15
categories: java
---
<h2>Surprising behaviour</h2>

<p>What result do you expect from this code?</p>

{% highlight java %}
BigDecimal a = new BigDecimal("2.00");
BigDecimal b = new BigDecimal("2.0");
System.out.println( a.equals(b) );
{% endhighlight %}

<p>It does not look suspicious and you would suspect it to 
    simply render <strong>true</strong>, right?</p>

<h2>Wrong!</h2>

<p>Unfortunately, this prints out a big red 
    <strong>false</strong>.</p>

<h2>JavaDoc to the rescue</h2>

<p>As you may read in the JavaDoc:</p>

<img src="/images/comparing-big-decimals/big-decimal-javadoc-equals.png"
     alt="BigDecimal JavaDoc for the 'equals' method" />

<h2>Mystery solved</h2>

<p>Oh, so <strong>scale</strong> needs to be the same as well.
    How intuitive!</p>

<h2>Solution</h2>

<p>If what you really wanted is to compare BigDecimals as
    values and not as objects you should have used
    <strong>compareTo</strong> method:</p>

{% highlight java %}
BigDecimal a = new BigDecimal("2.00");
BigDecimal b = new BigDecimal("2.0");
System.out.println( a.compareTo(b) == 0 ); // true
{% endhighlight %}

<h2>TLDR;</h2>

<p>When comparing BigDecimals use <strong>compareTo</strong> 
    instead of <strong>equals</strong>.</p>