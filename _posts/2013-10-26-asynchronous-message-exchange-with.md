---
layout: post
title: Asynchronous message exchange with Apache Camel
date: '2013-10-26T16:37:00.001+02:00'
tags: 
modified_time: '2013-10-29T18:40:18.848+01:00'
thumbnail: http://3.bp.blogspot.com/-U5BAUMAbXEY/Uk7pYScBYRI/AAAAAAAAAJM/_2kpvpKLf3M/s72-c/apache-camel.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-2504159450226512161
blogger_orig_url: http://www.code-thrill.com/2013/10/asynchronous-message-exchange-with.html
---
<h2>Introduction</h2> 

<img title="Apache Camel logo" src="/images/asynchronous-message-exchange-with-apache-camel/apache-camel.png" class="float-left" />

<p>In this post I will show you how to get started with Apache Camel. Apache Camel is an integration technology, that enables you to connect different systems with different endpoint types. For example requests from system A exposed through RESTful WebService could be transformed and pushed to system B accessible through SOAP WebService.</p> 

<div class="my-info" style="display: inline-block;">You can read more about the types of endpoints <a href="http://camel.apache.org/components.html">here</a>. </div> <p>To learn the basics we are going to build the following system:</p> 

<img title="Architecture diagram big picture" src="/images/asynchronous-message-exchange-with-apache-camel/architecture_diagram_big_picture.png" class="img-center" />

<p>As you can see, it is a standard two-way message exchange (request-response). Here is the sequence of steps in our application: 
	<ol>  
		<li>User requests the servlet.</li>  
		<li>Servlet pushes the request to the messaging queue and hangs waiting for the response.</li>  
		<li>On the other side a message processor pops the message from the queue and processes the request.</li>  
		<li>When the answer is ready it is pushed back to the messaging queue.</li>  
		<li>Servlet receives the response from the messaging system and shows the result to the user.</li>
	</ol>
</p> 

<div class="my-info" style="float: left">TLDR; You can skip the blog entry and jump right away to the source code <a href="https://github.com/mbukowicz/Learning-Apache-Camel">posted on GitHub</a>. </div> 
		
<h2>More details</h2> 

<p>Here is a more detailed view of the flow:</p> 

<img title="Architecture diagram in detail" src="/images/asynchronous-message-exchange-with-apache-camel/architecture_diagram.png" class="img-center" />

<ol style="clear: both">  
	<li>ExampleServlet calls Camel route called "direct:inbox" and hangs waiting for a response.</li>  
	<li>The route moves the request to a queue named "outbox".</li>  
	<li>The other route listening on the "outbox" queue pops the message.</li>  
	<li>Then the message goes through the connection spawned from the pool.</li>  
	<li>Finally the process method on the ExampleProcessor class gets called.</li>  
	<li>ExampleProcessor using the input body prepares response and pushes it back.</li>  
	<li>The message goes further through a JMS connection...</li>  
	<li>...and is put on the reply queue.</li>  
	<li>Based on the id number of the message Apache Camel knows it received response for the request sent in the first step.</li>  
	<li>ExampleServlet receives the response.</li>
</ol> 

<p>Once we learned the flow lets plan our TO-DO list:</p>

<ul>  
	<li>Configure Maven dependencies</li>  
	<li>Load Spring Camel configuration from the web.xml</li>  
	<li>Configure Apache Camel routes</li>  
	<li>Implement ExampleServlet</li>  
	<li>Implement ExampleProcessor</li>  
	<li>Configure ActiveMQ embedded server (broker)</li>  
	<li>Configure ActiveMQ JMS Connection Pool</li>
</ul> 

<h2>Configure Maven dependencies</h2>

<p>Here are the dependencies that should be added to the <strong>pom.xml</strong>:</p>

{% highlight xml %}
<project>
    ...
    <properties>
        <camel.version>2.11.0</camel.version>
        <activemq.version>5.8.0</activemq.version>
        <xbean.version>3.13</xbean.version>
        <spring.version>3.1.4.RELEASE</spring.version>
        <servlet.version>2.5</servlet.version>
    </properties>
    ...
    <dependencies>
        <!-- Apache Camel -->
        <dependency>
            <groupId>org.apache.camel</groupId>
            <artifactId>camel-core</artifactId>
            <version>${camel.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.camel</groupId>
            <artifactId>camel-jms</artifactId>
            <version>${camel.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.camel</groupId>
            <artifactId>camel-spring</artifactId>
            <version>${camel.version}</version>
        </dependency>

        <!-- Servlets -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>servlet-api</artifactId>
            <version>${servlet.version}</version>
            <scope>provided</scope>
        </dependency>

        <!-- the ActiveMQ client with connection pooling -->
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-client</artifactId>
            <version>${activemq.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-camel</artifactId>
            <version>${activemq.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-pool</artifactId>
            <version>${activemq.version}</version>
        </dependency>

        <!-- the ActiveMQ broker (embedded ActiveMQ) -->
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-broker</artifactId>
            <version>${activemq.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-spring</artifactId>
            <version>${activemq.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-pool</artifactId>
            <version>${activemq.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.activemq</groupId>
            <artifactId>activemq-kahadb-store</artifactId>
            <version>${activemq.version}</version>
        </dependency>

        <!-- Spring -->
        <dependency>
            <groupId>org.apache.xbean</groupId>
            <artifactId>xbean-spring</artifactId>
            <version>${xbean.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-web</artifactId>
            <version>${spring.version}</version>
        </dependency>
    </dependencies>
    ...
</project>
{% endhighlight %}

<h2>Loading Spring Camel configuration</h2>

<p>To load Camel Spring configuration lets put a standard Spring Context loader inside <strong>web.xml</strong>:</p>

{% highlight xml %}
<web-app>
    ...
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:camel-config.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
    ...
</web-app>
{% endhighlight %}

<h2>Apache Camel routes</h2>

<p>Below are the routes that represent the flow in our system. The entry point is the <strong>direct:inbox</strong> route, which will be called from the servlet. When a route starts with <strong>direct:</strong> it means you can call it directly from your Java code. On the other side, if you want to have Java code as your target endpoint (like with ExampleProcessor) then your class needs to implement the <strong>org.apache.camel.Processor</strong>.</p>

{% highlight xml %}
<beans>
    <import resource="classpath:activemq-connection-factory.xml" />
    <import resource="classpath:activemq-embedded.xml" />

    <camelContext id="exampleCamelContext" xmlns="http://camel.apache.org/schema/spring">
        <template id="exampleProducerTemplate" />

        <route>
            <from uri="direct:inbox"/>
            <to uri="activemq:queue:outbox" />
        </route>

        <route>
            <from uri="activemq:queue:outbox" />
            <to uri="exampleProcessor" />
        </route>
    </camelContext>

    <bean id="exampleProcessor" class="example.ExampleProcessor" />
</beans>
{% endhighlight %}

<h2>ExampleServlet</h2>

<p>Here is the servlet that calls <strong>direct:inbox</strong> route using <strong>ProducerTemplate</strong> provided by Camel:</p>

{% highlight java %}
public class ExampleServlet extends HttpServlet {

    private static final String DEFAULT_NAME = "World";

    private ProducerTemplate producer;

    @Override
    public void init() throws ServletException {
        producer = getSpringContext().getBean(
                       "exampleProducerTemplate", ProducerTemplate.class);
    }

    @Override
    public void service(ServletRequest req, ServletResponse res) 
        throws ServletException, IOException {
        
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String endpoint = "direct:inbox";
        String input = getName(request);
        String output = producer.requestBody(endpoint, input, String.class);

        response.getOutputStream().println(output);
    }

    private String getName(HttpServletRequest request) {
        String name = request.getParameter("name");
        if(name != null) {
            return name;
        }
        return DEFAULT_NAME;
    }

    private WebApplicationContext getSpringContext() {
        ServletContext servletContext = getServletContext();
        WebApplicationContext context = WebApplicationContextUtils.
            getRequiredWebApplicationContext(servletContext);
        return context;
    }
}
{% endhighlight %}

<h2>ExampleProcessor</h2>

<p>ExampleProcessor is responsible for receiving a request and creating a response:</p>

{% highlight java %}
public class ExampleProcessor implements org.apache.camel.Processor {

    @Override
    public void process(Exchange exchange) throws Exception {
        String name = exchange.getIn().getBody(String.class);

        String response = "Hello, " + name + "!";

        exchange.getOut().setBody(response);
    }
}
{% endhighlight %}

<h2>ActiveMQ embedded broker</h2>

<p>Below is the configuration of the embedded ActiveMQ (<strong>activemq-embedded.xml</strong>):</p>

{% highlight xml %}
<beans>
    <broker id="broker" brokerName="myBroker" useShutdownHook="false" useJmx="true"
            persistent="false" dataDirectory="activemq-data"
            xmlns="http://activemq.apache.org/schema/core">
        <transportConnectors>
            <transportConnector name="vm" uri="vm://myBroker"/>
            <transportConnector name="tcp" uri="tcp://0.0.0.0:61616"/>
        </transportConnectors>
    </broker>
</beans>
{% endhighlight %}

<p>To connect to the JMS broker we can use either the "vm://myBroker" uri when connecting from the same JVM, which performs better, or stick to the TCP on the 61616 port for external use (outside of the JVM). The most worthwhile attribute is certainly the "persistent" attribute, because it can significantly lower the performance of our system. If message durability is not required I strongly suggest to set it to false.</p> 

<h2>ActiveMQ JMS Connection Pool</h2>

<p>Finally the JMS configuration that will enable us to connect to the embedded broker:</p>

{% highlight xml %}
<beans>
    <bean id="jmsConnectionFactory" 
          class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="vm://myBroker?create=false&waitForStart=5000" />
    </bean>

    <bean id="pooledConnectionFactory" 
          class="org.apache.activemq.pool.PooledConnectionFactory" 
          init-method="start" destroy-method="stop">
        <property name="maxConnections" value="200" />
        <property name="connectionFactory" ref="jmsConnectionFactory" />
    </bean>

    <bean id="jmsConfig"
          class="org.apache.camel.component.jms.JmsConfiguration">
        <property name="connectionFactory" ref="pooledConnectionFactory"/>
        <property name="concurrentConsumers" value="250"/>
        <property name="autoStartup" value="true" />
    </bean>

    <bean id="activemq"
          class="org.apache.activemq.camel.component.ActiveMQComponent">
        <property name="configuration" ref="jmsConfig"/>
    </bean>
</beans>
{% endhighlight %}

<p>Here we are connecting using the intra-JVM protocol through the "vm://myBroker" uri. If connecting outside of the JVM, the connection factory would look like this:</p> 

{% highlight xml %}
<beans>
    <bean id="jmsConnectionFactory" 
          class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL" value="tcp://localhost:61616" />
    </bean>
    ...
</beans>
{% endhighlight %}

<p>Important thing that should be taken into consideration is the "id" attribute (<strong>id="activemq"</strong>). Using this identifier we configured a Camel route. The route was called <strong>activemq:</strong>queue:outbox. If we had to configure another ActiveMQ connection, we would have to come up with a different id.</p>  

<h2>The End</h2>

<p>I hope this post helped you understand the basics of Apache Camel. You can learn more from the great documentation on the <a href="http://camel.apache.org/examples.html">official Apache Camel site</a>.</p> 

<h2>Project on GitHub</h2> 

<p>If you want to try the example yourself, please feel free to check out the <a href="https://github.com/mbukowicz/Learning-Apache-Camel">project from GitHub</a>.</p>