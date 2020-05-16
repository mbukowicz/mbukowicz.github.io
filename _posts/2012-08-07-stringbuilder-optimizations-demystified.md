---
layout: post
title: StringBuilder Optimizations Demystified
date: '2012-08-07T09:30:00.001+02:00'
tags:
modified_time: '2012-08-10T16:53:26.260+02:00'
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-3996668414975823424
blogger_orig_url: http://www.code-thrill.com/2012/08/stringbuilder-optimizations-demystified.html
thumbnail: /images/string-builder-optimizations-demystified/thumbnail.png
---
<h2>Introduction</h2>

<p>There are lots of myths around concatenating Strings in Java. Lets find out exactly which of them are true.</p>
<div class="my-info">The results are based on the Oracle JDK 1.7 (Update 3). Please, feel free to leave a comment, if the compiler you are using works differently. </div>

<h2>Myth 1: You should always use StringBuilder when concatenating Strings</h2>

<p>When arguing about whether to use StringBuffer or StringBuilder it is usually fair to say that StringBuilder is better, because the StringBuffer has all the methods synchronized which can harm the performance. However, it is not always true that you have to use StringBuilder:</p>
<ol>  
	<li><p>The most obvious case is concatenating final values. Joining the standard Java constants:</p>

{% highlight java %}
public static final String X = "123";
public static final String Y = X + "456";
public static final String Z = Y + 789;
{% endhighlight %}

		<p>...will be optimized by the compiler to:</p>

{% highlight java %}
public static final String X = "123";
public static final String Y = "123456";
public static final String Z = "123456789";
{% endhighlight %}

		<p>What may be surprising, the final on its own is sufficient, so this snippet:</p>

{% highlight java %}
public class SomeClass {
  public final String a = "123";
  public final String b = a + "456";
  public final String c = b + 789;
}
{% endhighlight %}

		<p>...will be optimized to:</p>

{% highlight java %}
public class SomeClass {
  public final String a;
  public final String b;
  public final String c;

  public SomeClass() {
    a = "123";
    b = "123456";
    c = "123456789";
  }
}
{% endhighlight %}

		<p>The bytecode behind looks like this:</p>

{% highlight java %}
public class SomeClass {
  public final java.lang.String a;
  public final java.lang.String b;
  public final java.lang.String c;

  public SomeClass();
    Code:
       0: aload_0       
       1: invokespecial #18                 // Method java/lang/Object."<init>":()V
       4: aload_0       
       5: ldc           #7                  // String 123
       7: putfield      #20                 // Field a:Ljava/lang/String;
      10: aload_0       
      11: ldc           #10                 // String 123456
      13: putfield      #22                 // Field b:Ljava/lang/String;
      16: aload_0       
      17: ldc           #13                 // String 123456789
      19: putfield      #24                 // Field c:Ljava/lang/String;
      22: return        
}
{% endhighlight %}

	</li>   
	<li><p>What about the non-final fields? You should be a little bit concerned why do you want this, but leaving the rest to the compiler is also OK. Although the non-final fields are not optimized straight into Strings, StringBuilder is used. The following code for example:</p>

{% highlight java %}
public class SomeClass {
  public String x = "123";
  public String y = x + "456";
  public String z = y + 789;
}
{% endhighlight %}

		<p>...will be transformed by the compiler into this:</p>

{% highlight java %}
public class SomeClass {
  public String x;
  public String y;
  public String z;

  public SomeClass() {
    x = "123";
    y = (new StringBuilder(String.valueOf(x))).append("456").toString();
    z = (new StringBuilder(String.valueOf(y))).append(789).toString();
  }
}
{% endhighlight %}

		<p>The bytecode for the intrigued:</p>

{% highlight java %}
public class SomeClass {
  public java.lang.String x;
  public java.lang.String y;
  public java.lang.String z;

  public SomeClass();
    Code:
       0: aload_0       
       1: invokespecial #12                 // Method java/lang/Object."<init>":()V
       4: aload_0       
       5: ldc           #14                 // String 123
       7: putfield      #16                 // Field x:Ljava/lang/String;
      10: aload_0       
      11: new           #18                 // class java/lang/StringBuilder
      14: dup           
      15: aload_0       
      16: getfield      #16                 // Field x:Ljava/lang/String;
      19: invokestatic  #20                 // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
      22: invokespecial #26                 // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
      25: ldc           #29                 // String 456
      27: invokevirtual #31                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      30: invokevirtual #35                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      33: putfield      #39                 // Field y:Ljava/lang/String;
      36: aload_0       
      37: new           #18                 // class java/lang/StringBuilder
      40: dup           
      41: aload_0       
      42: getfield      #39                 // Field y:Ljava/lang/String;
      45: invokestatic  #20                 // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
      48: invokespecial #26                 // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
      51: sipush        789
      54: invokevirtual #41                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      57: invokevirtual #35                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      60: putfield      #44                 // Field z:Ljava/lang/String;
      63: return        
}
{% endhighlight %}

	</li>   
	<li><p>How about concatenating arguments of different types inside a method? Leave it to the compiler. This piece of code:</p>

{% highlight java %}
long a = 123;
String b = "456";
int c = 789;
String d = "000";
String result = a + b + c + d;
{% endhighlight %}

		<p>...will be turned into this:</p>

{% highlight java %}
long a = 123L;
String b = "456";
int c = 789;
String d = "000";
String result = (new StringBuilder(String.valueOf(a))).append(b).append(c).append(d).toString();
{% endhighlight %}

		<p>The bytecode:</p>

{% highlight java %}
Code:
       0: ldc2_w        #51                 // long 123l
       3: lstore_1      
       4: ldc           #29                 // String 456
       6: astore_3      
       7: sipush        789
      10: istore        4
      12: ldc           #53                 // String 000
      14: astore        5
      16: new           #18                 // class java/lang/StringBuilder
      19: dup           
      20: lload_1       
      21: invokestatic  #55                 // Method java/lang/String.valueOf:(J)Ljava/lang/String;
      24: invokespecial #26                 // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
      27: aload_3       
      28: invokevirtual #31                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      31: iload         4
      33: invokevirtual #41                 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      36: aload         5
      38: invokevirtual #31                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      41: invokevirtual #35                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      44: astore        6
      46: return        
{% endhighlight %}
	</li>
</ol>

<h2>Myth 2: You can rely on the StringBuilder optimization when you are concatenating inside an <cite>if</cite> statement</h2>
<p>After reading some Java compiler specs you can fall for this. Unfortunately this is not true. Whenever you are concatenating Strings in more than one expression, you should watch out. For example this:</p>

{% highlight java %}
String x = "123";
x += "456";
x += "789";
{% endhighlight %}  

<p>...will be unfortunately "optimized" to this code:</p>

{% highlight java %}
String x = "123";
x = (new StringBuilder(String.valueOf(x))).append("456").toString();
x = (new StringBuilder(String.valueOf(x))).append("789").toString();
{% endhighlight %}

<p>...when we expected this code:</p>

{% highlight java %}
String x = (new StringBuilder("123")).append("456").append("789").toString();
{% endhighlight %}

<p>Again the bytecode for the comparison with the decompiled Java code:</p>

{% highlight java %}
Code:
       0: ldc           #14                 // String 123
       2: astore_1      
       3: new           #18                 // class java/lang/StringBuilder
       6: dup           
       7: aload_1       
       8: invokestatic  #20                 // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
      11: invokespecial #26                 // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
      14: ldc           #29                 // String 456
      16: invokevirtual #31                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: invokevirtual #35                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      22: astore_1      
      23: new           #18                 // class java/lang/StringBuilder
      26: dup           
      27: aload_1       
      28: invokestatic  #20                 // Method java/lang/String.valueOf:(Ljava/lang/Object;)Ljava/lang/String;
      31: invokespecial #26                 // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
      34: ldc           #66                 // String 789
      36: invokevirtual #31                 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      39: invokevirtual #35                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      42: astore_1      
      43: return    
{% endhighlight %}

<p>Going further, the "advanced" example with an <cite>if</cite> statement will work out no better. The following:</p>

{% highlight java %}
String y = "123";
long z = 789;
boolean b = false;

y += "456";
if(!b) {
  y += "789";
  y += z;
}
{% endhighlight %}  

<p>...will <strong>NOT</strong> be magically turned into this:</p>

{% highlight java %}
StringBuilder sb = new StringBuilder("123");
long z = 789;
boolean b = false;

sb.append("456");
if(!b) {
  sb.append("789");
  sb.append(z);
}
String y = sb.toString();
{% endhighlight %}

<p>Instead it will come down to this:</p>

{% highlight java %}
String y = "123";
long z = 789L;
boolean b = false;

y = (new StringBuilder(String.valueOf(y))).append("456").toString();
if(!b) {
  y = (new StringBuilder(String.valueOf(y))).append("789").toString();
  y = (new StringBuilder(String.valueOf(y))).append(z).toString();
}
{% endhighlight %}

<p>As an exercise you can check the bytecode yourself.</p>

<div class="my-info">To generate the bytecode you have to first compile your java class:

{% highlight text %}
javac YourJavaClass.java
{% endhighlight %}

	Then disassemble the class file:

{% highlight text %}
javap -cp . -c YourJavaClass > bytecode.txt
{% endhighlight %}

	As a result the bytecode will be saved to the <cite>bytecode.txt</cite> file.
</div>

<h2>Myth 3: StringBuilder should be used only if you are concatenating inside a loop</h2>

<p>This myth is based on the assumption that the compiler is so clever that it can guess correctly where and how to put StringBuilder for the most of the time. After reading up to this point you should already know it is not true. StringBuilder should be used much more than expected. However, joining Strings inside a loop is a special case, where you can be 100% sure you have to use StringBuilder (or StringBuffer on rare occasions). For instance this:</p>

{% highlight java %}
String x = "123";
for(int i = 1; i < 100; i++) {
  x += i;
}
{% endhighlight %}

<p>...equals this:</p>

{% highlight java %}
String x = "123";
for(int i = 1; i < 100; i++) {
  x = (new StringBuilder(String.valueOf(x))).append(i).toString();
}
{% endhighlight %}

<p>...and we really wanted this:</p>

{% highlight java %}
StringBuilder sb = new StringBuilder("123");
for(int i = 1; i < 100; i++) {
  sb.append(i);
}
String x = sb.toString();
{% endhighlight %}

<h2>Summary</h2>

<p>I hope you enjoyed the article. Please, leave a comment, if you think there is something that should be added to the topic.</p>

<div class="my-info"><cite>TL;DR: For the most cases use the StringBuilder class. Better be safe than sorry.</cite></div>

<br /><br /> <strong>UPDATE:</strong>

<div class="my-info"><cite>TL;DR (made by <a href="http://standardout.wordpress.com/">standardout</a> in the comments): As a general rule, use '+' operator String concatenation when everything is defined in a single line. In some cases it will be more efficient, but even when not it will at least be easier to read. But never use += with Strings.</cite></div>

<div class="my-info">If you want to learn more, you can follow the <a href="http://www.reddit.com/r/programming/comments/xtan6/stringbuilder_optimizations_demystified/">Reddit discussion</a> about this blog post (as <a href="http://www.blogger.com/profile/15028902221295732276">Javin Paul</a> suggested). </div>
