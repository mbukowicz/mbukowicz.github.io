---
layout: post
title: PHP REST client with Pest
date: '2013-11-11T15:54:00.001+01:00'
tags: 
modified_time: '2013-11-12T09:19:50.321+01:00'
thumbnail: http://4.bp.blogspot.com/-CWF0OqyWRZI/Unvu-XwGKII/AAAAAAAAAJ4/KzKTfSlfh9w/s72-c/php-med-trans.png
blogger_id: tag:blogger.com,1999:blog-7932117927902690732.post-9061903315236332880
blogger_orig_url: http://www.code-thrill.com/2013/11/php-rest-client-with-pest.html
---
<h2>Introduction</h2> 

<img title="PHP logo" src="/images/php-rest-client-with-pest/php-med-trans.png" class="float-left" />

<p>In this post I'm going to show a very nice and simple way of creating PHP clients to RESTful web services. For this we are going to use a library called <a href="https://github.com/educoder/pest">Pest</a>, available on GitHub.</p>

<h2>Prerequisities</h2> 

<p><a href="https://github.com/educoder/pest">Pest</a> requires the <strong>mod_curl</strong> extension, so please make sure it is properly installed on your web server before proceeding further.</p> 

<h2>Project structure</h2> 

<p>The next step is to download the library files and put them in a separate folder called <strong>/Pest</strong>. We will also create an example <strong>index.php</strong> script, where we will put several code snippets calling a REST service. Here is the final project structure:</p> 

{% highlight text %}
|
+- Pest/
|  |
|  +- Pest.php
|  +- PestJSON.php
|  +- PestXML.php
|
+- index.php
{% endhighlight %}

<h2>Include <strong>Pest</strong></h2> 

<p>At the beginning of the <strong>index.php</strong> lets include the <strong>Pest</strong> library:</p> 

{% highlight php %}
<?php

require_once 'Pest/Pest.php';
require_once 'Pest/PestJSON.php';
require_once 'Pest/PestXML.php';
{% endhighlight %}

<h2>Plain HTTP</h2> 

<p>We will start with a simple example of calling our service using plain HTTP <strong>GET</strong> method:</p> 

{% highlight php startinline=true %}
$address = "http://127.0.0.1:8080";
$pest = new Pest($address);
$result = $pest->get("/my/rest/uri");
{% endhighlight %}

<p>In this example the <strong>$result</strong> variable contains a raw string, so data processing must be manually done by the programmer. It means that you will have to implement unmarshalling logic, for example from JSON, XML, YAML, etc., all by yourself.</p> 

<p>Now, lets see how we can <strong>POST</strong> some data and headers:</p> 

{% highlight php startinline=true %}
$data = array(
 "i" => 123,
 "str" => "Bob",
 "arr" => array("a", "b", "c"),
 "map" => array(
  "a" => 1,
  "b" => 2,
  "c" => 3
 )
);

$headers = array(
 "abc" => "123"
);

$result = $pest->post("/my/rest/uri", $data, $headers);
{% endhighlight %}

<p>In the service data is available through the parameters map:</p> 

{% highlight text %}
i -> 123
str -> Bob
arr[0] -> a
arr[1] -> b
arr[2] -> c
map[a] -> 1
map[b] -> 2
map[c] -> 3
{% endhighlight %}

<p>...and headers are available through the headers map:</p> 

{% highlight text %}
abc -> 123
{% endhighlight %}

<p>Again, we have to manually process the <strong>$result</strong>. This is a tedious task and a programmer shouldn't be involved in the most cases. In the following examples we will look into ways of automatically marshalling/unmarshalling data to/from JSON/XML.</p> 

<h2>JSON</h2> 

<p>Supposedly, we have a REST service accessible by calling <strong>GET</strong> method on the uri <strong>/my/rest/uri.json</strong> returning this data:</p> 

{% highlight javascript %}
{ 
  "i": 42, 
  "str": "Alice", 
  "arr": ["d", "e", "f"], 
  "map": { "x": 123, "y": 456 } 
}
{% endhighlight %}

<p>We can "turn on" the automatic unmarshalling from JSON to PHP objects tree by replacing the <strong>Pest</strong> object with <strong>PestJSON</strong>:</p> 

{% highlight php startinline=true %}
$address = "http://127.0.0.1:8080";
$pest = new PestJSON($address);
$result = $pest->get("/my/rest/uri.json");
print_r($result);
{% endhighlight %}

<p>The upper script prints out:</p> 

{% highlight text %}
Array ( 
  [i] => 42 
  [str] => Alice 
  [arr] => Array ( [0] => d [1] => e [2] => f ) 
  [map] => Array ( [x] => 123 [y] => 456 ) 
)
{% endhighlight %}

<p>With switching to the <strong>PestJSON</strong> we also get automatic marshalling to JSON. If we <strong>POST</strong> this data to our REST service:</p> 

{% highlight php startinline=true %}
$data = array(
 "i" => 123,
 "str" => "Bob",
 "arr" => array("a", "b", "c"),
 "map" => array(
  "a" => 1,
  "b" => 2,
  "c" => 3
 )
);
$result = $pest->post("/my/rest/uri.json", $data);
{% endhighlight %}

<p>...on the server in the body we receive:</p>

{% highlight javascript %}
{"i":123,"str":"Bob","arr":["a","b","c"],"map":{"a":1,"b":2,"c":3}}
{% endhighlight %}

<h2>XML</h2> 

<p>A similar case is with the XML format. Lets create a REST service accessible by calling <strong>GET</strong> on <strong>/my/rest/uri.xml</strong>. The service returns this XML:</p> 

{% highlight xml %}
<grandparent>
  <parent>
    <child>
      <i>123</i>
      <str>Frank</str>
      <list>
        <item>abc</item>
        <item>def</item>
        <item>ghi</item>
      </list>
      <map>
        <item key="a">1</item>
        <item key="b">2</item>
        <item key="c">3</item>
      </map>
    </child>
  </parent>
</grandparent>
{% endhighlight %} 

<p>Again, lets call it from Pest by replacing the <strong>Pest</strong> class with <strong>PestXML</strong> subclass:</p> 

{% highlight php startinline=true %}
$address = "http://127.0.0.1:8080";
$pest = new PestXML($address);
$result = $pest->get("/my/rest/uri.xml");
print_r($result);
{% endhighlight %}

<p>The script echoes the following PHP object tree:</p> 

{% highlight text %}
SimpleXMLElement Object ( 
  [parent] => SimpleXMLElement Object ( 
    [child] => SimpleXMLElement Object ( 
      [i] => 123 
      [str] => Frank 
      [list] => SimpleXMLElement Object ( 
        [item] => Array ( 
          [0] => abc 
          [1] => def 
          [2] => ghi 
        ) 
      ) 
      [map] => SimpleXMLElement Object ( 
        [item] => Array ( 
          [0] => 1 
          [1] => 2 
          [2] => 3 
        ) 
      ) 
    ) 
  ) 
)
{% endhighlight %}

<p>To access response we can either traverse the object tree in the standard PHP way:</p> 

{% highlight php startinline=true %}
$i = (int) $result->parent->child->i;
{% endhighlight %}

<p>...or by using XPath:</p> 

{% highlight php startinline=true %}
$queryResult = $result->xpath("/grandparent/parent/child/i");
$i = (int) $queryResult[0];
{% endhighlight %}

<p>The first example looks simpler, however XPath allows us to execute more advanced queries. What's more, we don't have to test the nullity of objects - the query will just return empty results array. In the first example, if parent or child was absent, an exception is thrown.</p> 

<p>Lets take a closer look at a case where XPath seems more suitable. The following code extracts the item with key equal to "a":</p> 

{% highlight php startinline=true %}
$queryResult = $result->xpath("/grandparent/parent/child/map/item[@key='a']");
$item = $queryResult[0];
$key = (string) $item['key'];
$value = (int) $item;
echo "$key -> $value";
{% endhighlight %}

<p>The result:</p> 

{% highlight text %}
a -> 1
{% endhighlight %}

<p>Lets finish with a little gotcha (which may be fixed in the future). If we <strong>POST</strong> this data:</p> 

{% highlight php startinline=true %}
$data = array(
 "i" => 123,
 "str" => "Bob",
 "arr" => array("a", "b", "c"),
 "map" => array(
  "a" => 1,
  "b" => 2,
  "c" => 3
 )
);

$result = $pest->post("/my/rest/uri.xml", $data);
{% endhighlight %}

<p>...on the server side the request will have empty body and data is passed like in plain HTTP (through parameters map):</p>

{% highlight text %}
i -> 123
str -> Bob
arr[0] -> a
arr[1] -> b
arr[2] -> c
map[a] -> 1
map[b] -> 2
map[c] -> 3
{% endhighlight %}

<h2>The End</h2> 

<p>I hope you enjoyed this post and as me you found the <a href="https://github.com/educoder/pest">Pest</a> library to be simple and robust.</p>