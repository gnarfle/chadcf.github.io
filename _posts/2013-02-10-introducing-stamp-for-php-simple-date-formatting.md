---
layout: post
title: Introducing stamp for PHP - simple date formatting
created: 1360530524
---
A while back I discovered the excellent <a href="https://github.com/jeremyw/stamp">stamp</a> gem for ruby. It gives you a fantastic way to format dates by simply supplying an example string, rather than having to remember the various formatting parameters to traditional date formatting functions. I've found this so useful that I had to create a PHP version, which I also lamely called stamp. 

Stamp provides a simple method to format dates. For example:

<?php

$stamp = new Stamp\Stamp();
$stamp->stamp("January 1, 2012 at 4:53PM", time());

?>

By supplying an example date string, you never again have to remember the difference between d, D, j, etc in php's date formatting. Even better, the stamp method can be passed a timestamp, a DateTime object or a string (which will be passed through strtotime). 

<a href="https://github.com/chadcf/stamp">Check out stamp at github</a>

There is still plenty of work to do, but it should be usable now. In the near future I'll be buffing up the test suite and trying to make it as performant as possible. 
