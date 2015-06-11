---
layout: post
title: Practical introduction to JRebel
date: '2013-04-23T21:29:00.002+02:00'
tags: 
modified_time: '2013-04-24T13:22:13.294+02:00'
thumbnail: http://2.bp.blogspot.com/-S0oUknjcY40/UXfAKPVbjfI/AAAAAAAAAH8/7iiRLquCL4U/s72-c/JRebel_logo_min.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-5321751956101648416
blogger_orig_url: http://www.code-thrill.com/2013/04/practical-introduction-to-jrebel.html
---
<img title="JRebel logo" src="/images/practical-introduction-to-jrebel/JRebel_logo_min.jpg" class="float-left" />

<p>In this post I would like to present my experiences with JRebel: from basics to more advanced stuff, where I had to dig through the Internet or ask ZeroTurnaround to find the answer.</p>

<div class="my-info" style="clear:left"><strong>Disclaimer:</strong> I wasn't paid for this post. All the information is based on my personal experience after using JRebel for the last 10 months. I tried to be honest and not to force any opinions. I think JRebel is cool, but as everything it is not perfect, so before buying it you should learn how it works. </div> 

<h2>What is JRebel?</h2> 

<p>JRebel instantly reloads changes to Java code. It works as a java agent on top of the JVM. It looks for changes in the folder with your compiled classes. If you compile a class, JRebel notices the change and compares the differences between class definitions. Then at the bytecode level it adds/removes/replaces fields/method bodies/etc. without restarting the JVM.<p>

<h2>Example configuration</h2>

<p>To configure JRebel you need two things: 
	<ul>  
		<li>place <cite>rebel.xml</cite> in the main folder of the package (jar/ear/war, etc.) that will be run as a standalone application or will be deployed to server</li>  
		<li>run your application or start your server with the -javaagent parameter specifying the path to the jar with JRebel</li>
	</ul>
</p> 

<h2><cite>rebel.xml</cite></h2>

<p>Here is an example of the <cite>rebel.xml</cite> file:</p>
<pre class="brush:xml; gutter:false;"><br />&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;<br />&lt;application xmlns:xsi=&quot;http://www.w3.org/2001/XMLSchema-instance&quot;<br />                xmlns=&quot;http://www.zeroturnaround.com&quot; <br />                xsi:schemaLocation=&quot;http://www.zeroturnaround.com http://www.zeroturnaround.com/alderaan/rebel-2_0.xsd&quot;&gt;<br />    &lt;classpath&gt;<br />        &lt;dir name=&quot;/absolute_path_to_your_project/target/classes&quot;&gt;&lt;/dir&gt;<br />    &lt;/classpath&gt;<br />    &lt;web&gt;<br />        &lt;link target=&quot;/&quot;&gt;<br />            &lt;dir name=&quot;/absolute_path_to_your_project/src/main/webapp&quot;&gt;&lt;/dir&gt;<br />        &lt;/link&gt;<br />    &lt;/web&gt;<br />&lt;/application&gt;<br /></pre> 

<div class="my-info">You should put absolute paths, not relative ones, as the working directory from which JRebel will run may be hard to specify.</div> 

<p>The <cite>classpath</cite> element is used for the path to your compiled classes. If you use Eclipse and have ticked the option <cite>Project/Build Automatically</cite>, after you save the source file Eclipse will automatically compile your code and overwrite the class file. Then JRebel will find out that the class is changed and reload it.</p> 

<p>The <cite>web/link</cite> element is for the web resources that you would like to have reloaded, i.e.: JSPs, image files, CSS files, JavaScript files, etc.</p> 

<h2><cite>-javaagent</cite></h2>

<p>Next thing is the JVM launch configuration. In the simplest case you have to add the <cite>-javaagent</cite> parameter either to your launch script or to the server start script:</p>

<pre class="brush:plain; gutter:false;"><br />java<br />-javaagent:"/absolute_path_to_jrebel_installation/jrebel/jrebel.jar"<br />-Drebel.workspace.path="/absolute_path_to_eclipse_workspace"<br />-Drebel.log.file="/absolute_path_to_your_home_folder/.jrebel/jrebel.log"<br />-Drebel.properties="/absolute_path_to_your_home_folder/.jrebel/jrebel.properties"<br />...specific server options...<br />$SERVER_MAIN_CLASS<br /></pre>

<p>Again, remember to specify absolute paths.</p> 

<h2>Plugins</h2>

<p>In addition to class reloading, JRebel provides a large variety of plugins that alter the way reloading works for different languages and libraries. The main idea behind plugins is that sometimes you may want to reload more files after compiling a class or changing a resource file. For example, if you change your Spring XML configuration file, you would also like to inform the Spring container that a class definition is outdated.</p>

<p>Personally, I used the Maven, Eclipse, IntelliJ, Spring, Struts and EJB plugins with minor issues. </p> 

<h2>What canNOT be reloaded?</h2>

<p>From the major problems I encountered, the first one was the lack of JiBX plugin. It troubled me quite a lot, because it is heavily used in my current company. JRebel support team suggested writing a JiBX plugin myself (more info about creating plugins can be found <a href="http://zeroturnaround.com/software/jrebel/resources/jrebel-plugins/">here</a>), but unfortunately JiBX does not provide incremental builds, so it would not add much value anyway.</p> 

<p>The next problem I had with JRebel was <cite>web.xml</cite>. Reloading <cite>web.xml</cite> is not supported, which was confirmed by the JRebel support team. This is quite reasonable, as such reloading feature should be placed in the server logic, which is unreachable from java agents.</p> 

<p>Besides this problems there are two well documented cases where your classes cannot be reloaded (for details check <a href="http://zeroturnaround.com/software/jrebel/features/">here</a>): 
	<ul>  
		<li>Replacing superclass</li>  
		<li>Adding/removing implemented interfaces</li>
	</ul>
</p> 

<h5>Replacing superclass</h5>
<p>Lets start with this set of classes:</p>

<pre class="brush:java; gutter:false;"><br />class A {}<br />class B {}<br />class C extends A {}<br /></pre>

<p>If you replace the parent of C with B:</p>

<pre class="brush:java; gutter:false;"><br />class C extends B {}<br /></pre>

<p>...then you will get the following error:</p>

<pre class="brush: plain; gutter:false;"><br />com.zeroturnaround.javarebel.SuperClassChangedError: <br />JRebel: Class 'C' superclass was changed from 'A' to 'B' and could not be reloaded!<br /></pre> 

<h5>Adding/removing implemented interfaces</h5>

<p>As for the new/removed interfaces lets start with this 2 interfaces and a class:</p>

<pre class="brush:java; gutter:false;"><br />interface A {}<br />interface B {}<br />class C implements A {}<br /></pre>

<p>...and this piece of code using them:</p>

<pre class="brush:java; gutter:false;"><br />A c = new C();<br /></pre>

<p>When we add a new interface implementation:</p>

<pre class="brush:java; gutter:false;"><br />class C implements A, B {}<br /></pre>

<p>...and tweak our code a little bit:</p>

<pre class="brush:java; gutter:false;"><br />B c = new C();<br /></pre>

<p>...we get this error:</p>

<pre class="brush: plain; gutter:false;"><br />java.lang.IncompatibleClassChangeError: <br />Class C does not implement the requested interface B<br /></pre> 

<p>As you can see JRebel is not a silver bullet and before buying it you should test it on your production projects first and learn about its limitations.</p> 

<h2>How to cope with reloading problems?</h2>

<p>From time to time JRebel will spit out an error that it could not reload the classes. Another time you will change the source code and the change would not be reloaded, because of plugin limitation, some special case, server cached some objects, etc.</p>

<p>My approach here to this problem is very simple. I try not to restart the server as long as possible. When I encounter a situation where my class/resource was not reloaded (it happens from time to time, but in practice not very often), I just rebuild the application.</p> 

<h2>Licenses</h2>

<p>JRebel is a commercial product and is not cheap, however take notice that you can use JRebel for free, if you are programming in Scala, working on a non-profit project or as a open-source developer.</p> 

<p>As for the commercial licenses there are currently two options: 
	<ul>  
		<li>Enterprise     
			<ul>      
				<li>obviously more expensive as the name suggests</li>        
				<li>comes with a reporting server (reporting the usage/number of reloads/etc.)</li>        
				<li>licences are floating, which means that every license is blocked whenever a developer uses JRebel</li>
				<li>because licenses are floating, you can buy less licenses (about 80% of the number of developers as suggested by the support team)</li>    
			</ul>  
		</li>  
		<li>Base     
			<ul>      
				<li>cheaper than enterprise</li>      
				<li>you have to do reporting on your own (you can compute statistics based on <cite>~/.jrebel/javarebel.stats</cite>, where timestamps of reloads are being stored)</li>      
				<li>named, which means attached to one developer at a time (the support team reassured us, that it does not mean that when a developer leaves the company the license is discarded, another programmer can still use it)</li>      
				<li>this is the one we have chosen</li>    
			</ul>  
		</li>
	</ul>
</p> 

<h2>How ZeroTurnaround protects JRebel against abuse?</h2>

<p>In order to prevent from cheating, i.e. buying 10 licences and giving them to 20 developers, ZeroTurnaround computes statistics of the usage of JRebel based on the IP numbers. If they notice any significant abnormalities, they will contact you to check whether it is an issue.</p> 

<h2>Trial period</h2>

<p>A member of JRebel marketing team contacted me and offered my company about 5 licenses for free for a 30-day trial period. During this time my teammates and myself were using JRebel for our daily tasks. After the license expired we decided that the tool helps us being more productive and the company purchased a license for every developer in my development team.</p> 

<h2>Support</h2>

<p>During the trial period I often asked questions by e-mail and the JRebel support team got back to me usually within one working day and sometimes even in a matter of minutes. I also know they have a forum, where you can get some help. Since buying JRebel we haven't encountered any issues, so fortunately there wasn't any need to contact the JRebel guys.</p> 

<h2>Final thoughts</h2>

<p>I hoped you learned something new from my post and it helped you in making a decision, whether JRebel is a good choice for you.</p>