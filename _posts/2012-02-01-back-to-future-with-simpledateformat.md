---
layout: post
title: Back To The Future With SimpleDateFormat
date: '2012-02-01T10:27:00.000+01:00'
tags:
modified_time: '2012-03-11T13:59:26.226+01:00'
thumbnail: /images/back-to-future-with-simpledateformat/thumbnail.jpg
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-239286130093191068
blogger_orig_url: http://www.code-thrill.com/2012/02/back-to-future-with-simpledateformat.html
---
<img src="/images/back-to-future-with-simpledateformat/back-to-the-future.jpg" title="Back to the future" class="float-left" />
<p>Some time ago I had to fix a really strange bug in a Java web application. A date displayed to the user was going crazy. Where it should be 20 November 2011 it was something like 12 January 510 or 4 October 3040. The app was obviously broken...</p>
<p>I checked the code and at first it looked all right. There was an utility class looking something like that:</p>
{% highlight java %}
import java.text.DateFormat;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class TimeUtil {

    private static final DateFormat FORMAT =
            new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static String formatDate(Date date) {
        return FORMAT.format(date);
    }

    public static Date parse(String dateString) throws ParseException {
        return FORMAT.parse(dateString);
    }
}
{% endhighlight %}

<p>I debugged the code and it run OK. I googled a few times and nobody seemed to had this problem. So I went back and forth between source files wondering how this was possible... and then I got an idea. I wrote a small program to prove me right and it all became clear. The program looked like that:</p>
{% highlight java %}
import java.text.DateFormat;
import java.text.SimpleDateFormat;
import java.util.Date;

class DateFormatBugThread extends Thread {

    private DateFormat df;

    public DateFormatBugThread(DateFormat df) {
        this.df = df;
    }

    public void run() {
        for(int i=0; i<10; i++) {
            try {
                Date d = new Date(df.parse("2011-08-25 22:05").getTime());
                System.out.println(df.format(d));
            } catch (Exception ex) {
                System.out.println(ex);
            }
        }
    }
}

public class Main {
    public static void main(String[] args) throws Exception {
        DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm");
        DateFormatBugThread t1 = new DateFormatBugThread(df);
        DateFormatBugThread t2 = new DateFormatBugThread(df);

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
{% endhighlight %}

<p>And the results were:</p>

{% highlight text %}
java.lang.NumberFormatException: For input string: ""
2011-08-25 22:05
2200-08-25 22:05
2200-08-25 22:05
1970-01-05 22:20
1970-01-05 22:20
1970-08-01 22:05
1970-08-01 22:05
java.lang.NumberFormatException: For input string: ""
java.lang.NumberFormatException: For input string: ""
2011-08-01 22:05
2011-08-01 22:05
java.lang.NumberFormatException: For input string: ""
1970-01-01 22:05
java.lang.NumberFormatException: For input string: ""
2011-08-25 22:05
0005-08-25 22:05
2011-08-25 22:05
2011-08-25 22:05
2011-08-25 22:05
{% endhighlight %}

<p>As you can see the SimpleDateFormat class is not thread-safe. There is even a caution in <a href="http://docs.oracle.com/javase/6/docs/api/java/text/SimpleDateFormat.html#synchronization">the Oracle JavaDoc</a>.</p>

<h2>Solution</h2>
<p>The simplest solution is to create a new SimpleDateFormat object every time it is needed. However, according to <a href="http://www.javacodegeeks.com/2010/07/java-best-practices-dateformat-in.html">this source</a> the fastest way to use <code>DateFormat</code> in a multi-threaded environment is to use <code>ThreadLocal</code>. So, our utility class could look like this:</p>
{% highlight java %}
import java.text.DateFormat;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class TimeUtil {

    private static final ThreadLocal<DateFormat> FORMAT =
        new ThreadLocal<DateFormat>() {
            @Override protected DateFormat initialValue() {
                return new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            }
        };

    public static String formatDate(Date date) {
        return FORMAT.get().format(date);
    }

    public static Date parse(String dateString) throws ParseException {
        return FORMAT.get().parse(dateString);
    }
}
{% endhighlight %}

<h2>Summary</h2>
<p>To sum up, if your application works in a concurrent environment, you should always check your classes for thread-safety, especially when using Java Formatters.</p>

<h2>Bonus Exercise</h2>
<p>Why not check your code right now? Here is a regular expression that will do it for you:</p>
{% highlight text %}
static\s+(final\s+)?\w*Formatter
{% endhighlight %}
