---
layout: post
author: "Dennis Claassen"
title: "Slow pages that are blocking navigation"
---
When developing a website with PHP, from time to time pages might end up slow to load. Since we don't want our users to wait too long for these pages, we load the slow parts of the page with AJAX. This way, the perceived speed is much higher. But when we have multiple pieces of data that we try to load with simultaneous requests, the requests might be resolved sequentially. Or when the user wants to navigate to a different page, this page suddenly takes a long time to load. While it would normally be much faster!

How does this happen and how can we solve it?

## The issue

The slow data which is fetched with AJAX, still takes a long time to load. While the perceived speed is higher, the overall speed is not. That might even be slower, because of the overhead of the extra request(s).

To allow my AJAX request access to some user data, I use sessions. In PHP this is easily done by calling [`session_start()`](http://php.net/session_start).

But what does `session_start()` actually do? Explanation according to php.net:

> *session_start()* creates a session or resumes the current one based on a session identifier passed via a GET or POST request, or passed via a cookie.

This session that is resumed on each request, allows me to remember data over a multitude of pageviews. Which is really handy! But what does it do internally per request? When the function is called to start using the session, the server closes access to the session. Because of this, no other page can access the session during this time. When the first request is finished, it unlocks the session and the second request can go on where it was stuck - waiting at `session_start()`.

## The solution

Luckily there is a way to unlock the session earlier. If you do not have to write anything to your session anymore, this can be used to give other requests access to your session data. The magic PHP function [`session_write_close()`](https://www.php.net/session_write_close) solves the issue by closing the ability to write to the session. By closing the session in one request, it becomes available sooner in other requests that can now proceed sooner.

## A new issue

Because you close the possibility to write to the session, `$_SESSION` becomes a normal variable which you can write and read from. Modifying this variable will no longer actually write to the session data, it just writes to the variable. Nobody tells you this does not actually gets set in the session.

When a colleague, or your future self,  wants to write to the session, there are no hints that the session was closed. That data will not be saved in the session and hairs will be pulled out because of this issue.

## The final solution

I have build a custom function to register that the session was closed. This function calls `session_write_close()`. But besides that, it registers a shutdown handler. For this handler it remembers the state of the `$_SESSION` variable and compares it with the state of this variable at shutdown. If these states do not match, a PHP error is thrown to inform you about this.

{% highlight php %}
/**
 * Closes writing to the current session and register a check to throw a warning
 * when something has been written to the session.
 *
 * This will notify anyone writing to the session, while
 * the session was actually closed. Without this check, there
 * would be no warning from PHP's session_write_close function.
 */
function register_session_write_close()
{
  // Close writing to the current session.
  session_write_close();

  // Register a shutdown function to check if $_SESSION has changed.
  // We compare the session as it is now, to the session as it is at shutdown.
  // If the session seems changed, a PHP warning will be triggered.
  register_shutdown_function( function( $oldSession )
  {
    if( $_SESSION !== $oldSession )
    {
      trigger_error( 'Session was changed after calling register_session_Write_close.', E_USER_WARNING );
    }
  }, $_SESSION );
}
{% endhighlight %}
