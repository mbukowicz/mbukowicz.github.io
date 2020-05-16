---
layout: post
title: Configuration That Rocks! With Apache Commons
date: '2012-05-05T09:54:00.001+02:00'
tags:
modified_time: '2012-05-05T10:23:49.290+02:00'
thumbnail: /images/configuration-that-rocks-with-apache-commons/thumbnail.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-3439647561559080159
blogger_orig_url: http://www.code-thrill.com/2012/05/configuration-that-rocks-with-apache.html
---
<img src="/images/configuration-that-rocks-with-apache-commons/commons-logo.png" title="Apache Commons logo" class="float-left" />

<p>Using a modern library in your project is great fun, but often it is far better to resist the tempation and fallback to the tried and tested. In terms of storing configuration data in Java a proven solution is the Apache Commons Configuration framework.</p>

<h2>What you will learn</h2>
<p>You will learn how to:
<ul>  
	<li>fetch data from a XML file</li>    
	<li>access environment variables</li>  
	<li>combine different types of configuration (XML-based, environment-based, etc.)</li>  
	<li>automatically reload configuration after a change</li>
</ul></p>

<p>In our example a database configuration will be stored in a XML file and the type of the environment (development, test, production, etc.) will be set as an environment variable. You will see more detail in a moment, but first lets configure Maven.</p>

<h2>Maven setup</h2>
<p>For our simple app we will need to add this dependencies to the <cite>pom.xml</cite>:</p>

{% highlight xml %}
<dependencies>
    <dependency>
        <groupId>commons-configuration</groupId>
        <artifactId>commons-configuration</artifactId>
        <version>1.8</version>
    </dependency>
    <dependency>
        <groupId>commons-beanutils</groupId>
        <artifactId>commons-beanutils</artifactId>
        <version>1.8.0</version>
    </dependency>
    <dependency>
        <groupId>commons-jxpath</groupId>
        <artifactId>commons-jxpath</artifactId>
        <version>1.3</version>
    </dependency>
</dependencies>
{% endhighlight %}

<p>In order to fetch the <cite>url</cite> and <cite>port</cite> we can use this piece of code:</p>

{% highlight java %}
XMLConfiguration config = new XMLConfiguration("const.xml");

// 127.0.0.1
config.getString("database.url");  

// 1521
config.getString("database.port");
{% endhighlight %}

<p><cite>XMLConfiguration</cite> is an Apache Commons class that loads contents from the specified configuration file and provides a nice <strong>dot notation for accessing the values</strong>. For example the expression <cite>database.port</cite> maps to the node <cite>config/database/port</cite> with text "1521". Ofcourse there are more methods than <cite>getString</cite> to obtain data. Here are some other basic methods:
<ul>  
	<li>getBoolean</li>  
	<li>getByte</li>  
	<li>getDouble</li>  
	<li>getFloat</li>  
	<li>getInt</li>  
	<li>getInteger</li>  
	<li>getList</li>  
	<li>getLong</li>  
	<li>getStringArray</li>
</ul>
You can check more in the <a href="http://commons.apache.org/configuration/apidocs/org/apache/commons/configuration/Configuration.html">Apache Commons JavaDoc</a>.</p>

<h2>Evolving configuration</h2>
<p>Suppose, after a while we decided that we would like to store configuration not only for single, but for <strong>multiple databases</strong>. We then change the <cite>const.xml</cite> file:</p>

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!-- const.xml -->
<config>
    <databases>
        <database>
            <name>dev</name>
            <url>127.0.0.1</url>
            <port>1521</port>
            <login>admin</login>
            <password>pass</password>
        </database>
        <database>
            <name>production</name>
            <url>192.23.44.100</url>
            <port>1521</port>
            <login>admin</login>
            <password>not-so-easy-pass</password>
        </database>
    </databases>
</config>
{% endhighlight %}

<p>Now to access the url values of the databases we can use this code:</p>

{% highlight java %}
XMLConfiguration config = new XMLConfiguration("const.xml");

// 127.0.0.1
config.getString("databases.database(0).url");

// 192.23.44.100
config.getString("databases.database(1).url");
{% endhighlight %}

<p>As you can see to fetch consecutive values we used <strong>zero-based index inside parantheses</strong>, i.e. <cite>databases.database(1).url</cite> maps to the second node inside the <cite>databases</cite> parent.</p>

<h2>Introducing XPath expressions</h2>
<p>Dot notation is nice, but works only in simple cases. For complex, real-life situations a better solution may be to use the XPath expression language. The main advantage here is that you can write <strong>advanced queries against XML, while still being concise</strong>. Here is how you could rewrite the previous snippet:</p>

{% highlight java %}
XMLConfiguration config = new XMLConfiguration("const.xml");
config.setExpressionEngine(new XPathExpressionEngine());

// 127.0.0.1
config.getString("databases/database[name = 'dev']/url");        

// 192.23.44.100
config.getString("databases/database[name = 'production']/url");
{% endhighlight %}

<p>...and here is an explanation of this two XPath queries:</p>

<img src="/images/configuration-that-rocks-with-apache-commons/databases_xpath.jpg" title="Database XPath explained" class="img-center" />

<h2>Accessing the environment</h2>
<p>With Apache Commons Configuration you can also read environment variables. Here is the code for getting the value of the <cite>ENV_TYPE</cite> environment variable:</p>

{% highlight java %}
EnvironmentConfiguration config = new EnvironmentConfiguration();
config.getString("ENV_TYPE");
{% endhighlight %}

<p>It is assumed that the <cite>ENV_TYPE</cite> variable is set. To check whether you have done it correct, you can run the below script in the console:</p>

{% highlight java %}
echo %ENV_TYPE%        # for Windows
# or...
echo $ENV_TYPE         # for Linux/Mac OS
{% endhighlight %}

<p>You should see as an output the value you have chosen for the <cite>ENV_TYPE</cite> variable.</p>

<h2>Combining the configurations</h2>
<p>Lets summarize what we have learned so far. Underneath is a <cite>getDbUrl</cite> method that does the following:
<ol>  
	<li>checks the value of the environment variable called <cite>ENV_TYPE</cite></li>  
	<li>if the value is either <cite>dev</cite> or <cite>production</cite> it returns database url for the environment</li>  
	<li>if the variable is not set it throws exception</li>
</ol></p>

{% highlight java %}
public String getDbUrl() throws ConfigurationException {
    EnvironmentConfiguration envConfig = new EnvironmentConfiguration();
    String env = envConfig.getString("ENV_TYPE");
    if("dev".equals(env) || "production".equals(env)) {
        XMLConfiguration xmlConfig = new XMLConfiguration("const.xml");
        xmlConfig.setExpressionEngine(new XPathExpressionEngine());
        String xpath = "databases/database[name = '" + env + "']/url";
        return xmlConfig.getString(xpath);
    } else {
        String msg = "ENV_TYPE environment variable is " +
                     "not properly set";
        throw new IllegalStateException(msg);
    }
}
{% endhighlight %}

<h2>Centralize your configuration</h2>
<p>Creating multiple configuration objects for every configuration type seems a little bit awkward. What will happen, if we wanted to add another XML-based configuration? We should then create another XMLConfiguration object and we will end up with more and more distributed configuration code. A solution for the problem may be to <strong>centralize the configuration in a form of a single XML file</strong>. Here is an example of the XML file for our scenario:</p>

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!-- config.xml -->
<configuration>
  <env />
  <xml fileName="const.xml" />
</configuration>
{% endhighlight %}

<p>To load this file you should use the <cite>DefaultConfigurationBuilder</cite> which will combine the <cite>env</cite> and <cite>xml</cite> configurations. The final version of the <cite>getDbUrl</cite> method would look like this:</p>

{% highlight java %}
public String getDbUrl() throws ConfigurationException {
    DefaultConfigurationBuilder builder =
        new DefaultConfigurationBuilder("config.xml");
    boolean load = true;
    CombinedConfiguration config = builder.getConfiguration(load);
    config.setExpressionEngine(new XPathExpressionEngine());
    String env = config.getString("ENV_TYPE");
    if("dev".equals(env) || "production".equals(env)) {
        String xpath = "databases/database[name = '" + env + "']/url";
        return config.getString(xpath);
    } else {
        String msg = "ENV_TYPE environment variable is " +
                     "not properly set";
        throw new IllegalStateException(msg);
    }
}
{% endhighlight %}

<h2>Automatic reloading</h2>
<p>There are many cool features in the Apache Commons Configuration. One of them is <strong>reloading file-based configuration when it gets changed</strong>. The framework will poll for changes every specified amount of time and if the contents of the file changed the responsible object is refreshed. To get it working you just have to add a file reloading strategy to your configuration. You can do it either programmatically:</p>

{% highlight java %}
XMLConfiguration config = new XMLConfiguration("const.xml");
ReloadingStrategy strategy = new FileChangedReloadingStrategy();
strategy.setRefreshDelay(5000);
config.setReloadingStrategy(strategy);
{% endhighlight %}

<p>...or in the main configuration file:</p>

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!-- config.xml -->
<configuration>
  <env />
  <xml fileName="const.xml">
    <reloadingStrategy refreshDelay="5000"
      config-class="org.apache.commons.configuration.reloading.FileChangedReloadingStrategy"/>
  </xml>
</configuration>
{% endhighlight %}

<p>This configuration results in a <strong>check for a file change every 5 seconds</strong>.</p>

<div class="my-info">Want to try the examples yourself? Check out <a href="https://github.com/mbukowicz/Learning-Apache-Commons">this github project</a>. </div>

<h2>Final thoughts</h2>
<p>Apache Commons is my personal choice for the configuration-related code. I hope this article convinced you, that this framework may provide a very useful interface to your static data. What needs to be mentioned at the end is that not all features were covered in this article. Among many interesting are the abilities of the framework to:
<ul>  
	<li>load your configuration from different sources like properties files, ini files, a database, etc.</li>  
	<li>add new values to the configuration and save it back to the source it came from</li>  
	<li>listen for the configuration change event</li>  
	<li>automatically resolve the actual path to the configuration files (it does not matter whether you put them inside a jar or in the application folder)</li>
</ul>I hope you enjoyed reading my text. Please, feel free to leave a comment.</p>
