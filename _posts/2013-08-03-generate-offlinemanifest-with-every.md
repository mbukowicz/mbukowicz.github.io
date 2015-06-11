---
layout: post
title: Generate offline.manifest with every Java build
date: '2013-08-03T07:35:00.000+02:00'
tags: 
modified_time: '2013-08-03T07:46:49.149+02:00'
thumbnail: http://2.bp.blogspot.com/-oYfAvcuLg0g/UfyYk2Jz2RI/AAAAAAAAAIw/gCG8Kb5wpIU/s72-c/HTML5-logo.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-2251784418726999096
blogger_orig_url: http://www.code-thrill.com/2013/08/generate-offlinemanifest-with-every.html
---
<h2>Introduction</h2> 

<img title="HTML5 logo" src="/images/generate-offline-manifest-with-every-java-build/HTML5-logo.png" class="float-left" /> 

<p>In this post I will explain how to automatically generate a list of web resources for HTML5 offline caching. To keep the story short, you need to specify all CSS, JS, etc. in a manifest file. Then the browser will know that it needs to store them in cache. When you disconnect from the Internet, it will use those files instead of complaining about no connection available.</p> 

<div class="my-info" style="float: left;">You can read more about building offline websites <a href="http://diveintohtml5.info/offline.html">here</a>. </div>

<h2 style="clear: both;">Project structure</h2> 

<p>Lets assume you're using Maven and your project structure looks something like this:</p> 

{% highlight text %}
|
+- src/
|  |
|  +- main/
|     |
|     +- webapp/
|        |
|        +- js/
|        |  |
|        |  +- ...a bunch of JS files
|        |
|        +- css/
|        |  |
|        |  +-...a bunch of CSS files
|        |
|        +- WEB-INF/
|        |  |
|        |  +- web.xml
|        |
|        +- offline.manifest
|        +- index.html
|
+- build.xml
+- pom.xml
{% endhighlight %}

<h2><cite>index.html</cite></h2> 

<p>To enable offline mode every html file of your website needs to include offline manifest. For example this is how <cite>index.html</cite> would look:</p>

{% highlight html %}
<!DOCTYPE html>
<html manifest="/offline.manifest">
  ...
</html>
{% endhighlight %}

<h2><cite>offline.manifest</cite></h2> 

<p><cite>offline.manifest</cite> looks something like this:</p>

{% highlight text %}
CACHE MANIFEST
# 01/08/2013 16:00:00
/css/first.css
/css/second.css
...
/js/first.js
/js/second.js
...
{% endhighlight %}

<p>It is a good practice to add some form of a timestamp during every build (<cite># 01/08/2013 16:00:00</cite>). By doing this you let the browser know that it needs to reload the cached resources. Otherwise, if one of your files changes, but it is not a new file and the name haven't changed, there is a good chance it won't be reloaded.</p> 

<p>We could add files manually to the <cite>offline.manifest</cite>, but that's quite error-prone. You can easily forget to add a file and then the offline mode won't work in a particular case. It's usually better to automatically generate it. Here we will use Apache Ant. The script logic will be placed in the <cite>build.xml</cite>.</p> 

<h2><cite>build.xml</cite></h2> 

<p>If you execute ant from the project directory:</p> 

{% highlight text %}
/path/to/your/project$ ant 
{% endhighlight %}

<p>it will automatically execute the <cite>generate-offline-manifest</cite> target:</p> 

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<project name="my-project" default="generate-offline-manifest">
    <target name="generate-offline-manifest">
        <property name="webapp-dir" value="${basedir}/src/main/webapp"/>

        <!-- 
                * pathsep -> separate files with new lines 
                * targetos -> make sure our files contain slashes (/) 
                              instead of windows backslashes (\) 
        -->
        <pathconvert property="list-of-files" pathsep="&#xA;" targetos="unix">
            <fileset dir="${webapp-dir}" >
                <include name="css/**/*" />
                <include name="js/**/*" />
            </fileset>

            <!-- make the path relative to the webappp dir (remove the absolute part) -->
            <map from="${webapp-dir}" to=""/>
        </pathconvert>

        <tstamp>
            <!-- generate timestamp - value will be stored in generation-time property -->
            <format property="generation-time" pattern="dd/MM/yyyy hh:mm:ss" />
        </tstamp>

        <!-- generate the offline.manifest -->
        <echo file="${webapp-dir}/offline.manifest">CACHE MANIFEST
# ${generation-time}
${list-of-files}</echo>
    </target>
</project>
{% endhighlight %}

<h2><cite>pom.xml</cite></h2> 

<p>In order to generate the <cite>offline.manifest</cite> every time the <cite>.war</cite> file is created, you can invoke the <cite>build.xml</cite> in your Maven build, during the prepare-package phase. You can accomplish this by adding the following snippet to your pom.xml:</p> 

{% highlight xml %}
<project>
    ...
    <build>
        ...
        <plugins>
            ...
            <plugin>
                <artifactId>maven-antrun-plugin</artifactId>
                <version>1.7</version>
                <executions>
                    <execution>
                        <phase>prepare-package</phase>
                        <goals>
                            <goal>run</goal>
                        </goals>
                        <configuration>
                            <target>
                                <ant antfile="${basedir}/build.xml">
                                    <target name="generate-offline-manifest"/>
                                </ant>
                            </target>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
{% endhighlight %}

<h2>The End</h2> 

<p>I hope you found this micro-tutorial useful. Happy coding!</p>