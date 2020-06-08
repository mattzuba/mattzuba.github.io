---
title: Proper LIKE Comparison For MySQL and UUIDs
tags: mysql
redirect_from: /2015/01/proper-like-comparison-mysql-uuids/
---

I recently ran into this issue, and I wasn't able to find any meaningful information online until I posted on [StackOverflow](http://stackoverflow.com/q/27946895/2735804) about it, so I'm hoping that this will help someone else in the future.  

In an application I'm developing, UUIDv4's are being used to load up a form from the database to dynamically display to the end-user to fill out.  These are stored in the database as a BINARY(16), the hex representation of which is 32 characters long (excluding dashes), and this would simply be too cumbersome to ask an end-user to enter if they needed to manually navigate to a form.  Therefore, I decided to use a LIKE comparison which would allow the user to specify the UUID in the URL in 1 byte increments (2 hex characters).

This worked out well except when it came to demo day.  I generated a form, then tried to demonstrate navigating to it by its full UUID and a shortened one.  The shortened one worked, but the full UUID threw a 'Not Found' exception in my application, indicating the database search failed.

I started chopping off 1 byte at a time from the UUID to determine where the problem occurred and found that there was something wrong with the last 4 bytes, but had no idea what it could be.  MySQL query was pretty simple:

{% highlight sql %}
SELECT * FROM MyTable WHERE uuid LIKE CONCAT(UNHEX('0BFADD4EEFC14C05A7CA83245C37EEB7'), '%');
{% endhighlight %}
    
Do you see the problem with the last 4 bytes (8 hex characters)?  I didn't at first until [spencer7593](http://stackoverflow.com/a/27950723/2735804) opened my eyes to the problem.  The first of the last 4 bytes is 5C, or decoded, `\`.  And backslashes have special meaning in MySQL!  The CONCAT function was translating that backslash (and the literal 7 after it (hex 37)) into a normal 7, thus dropping the backslash.  This of course didn't return any results because the data in the table was an actual backslash byte.

The solution to this was painfully clear once spencer7593 pointed it out.  I needed to escape the backslash as well as the `%` and `_` characters that might appear as well AFTER UNHEX, but before CONCAT (since I needed to concat an actual `%`, not a literal).

Using the solution in the SO post linked above, my query now looks like this, and works flawlessly (so far anyway):

{% highlight sql %}
SELECT * FROM MyTable WHERE uuid LIKE CONCAT(REPLACE(REPLACE(REPLACE(UNHEX(?), '\\', '\\\\'), '_', '\\_'), '%', '\\%'), '%');
{% endhighlight %}

Wanna play around with it?  See it in action on this [SQLFiddle](http://sqlfiddle.com/#!9/53afa/1).