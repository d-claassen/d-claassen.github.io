---
layout: post
author: "Dennis Claassen"
categories: "programming"
---
Sometimes we notice we have slow pages. We don't want the user to wait, so we load the slow page with Ajax. This way, the perceived speed is much higher. But when the user wants to navigate to a different page, this page suddenly takes a long time to load. While it would normally be much faster!

How does this happen and how can we solve it?

## The issue

The slow data which is fetched with AJAX, still takes a long time to load! While the perceived speed is higher, the overall speed is not. That might even be slower, because of an extra request!

To allow my AJAX request access to some user data, I use sessions. In PHP this is easily done by calling [`session_start()`](http://php.net/session_start).

But what does `session_start()` actually do?

_"It allows me to remember data over a multitude of pageviews."_ Which is definitely true! But what does it do per request? When the function is called to start using the session, the server closes access to the session. This i s the reason no other page can access the session during this time. When the first request is finished, it unlocks the session and the second request can go on where it was - at `session_start()`.

## The solution

Luckily there is a way to unlock the session earlier. If you do not have to write anything to your session anymore, this can be used to give other requests access to your session data. The magic PHP function [`session_write_close()`](https://www.php.net/session_write_close) solves the issue by closing the ability to write to the session.

## A new issue

Because you close the possibility to write to the session, `$_SESSION` becomes a normal variable which you can write and read from. Modifying this variable will no longer actually write to the session dtaa, it just writes to the variable. Nobody tells you this does not actually gets set in the session.

When another team member wants to write to the session, he gets no hints that the session was closed. His data will not be saved in the session and he will be pulling his hair out on this issue.

## The final solution

I have build a custom function to register that the session was closed. This function calls `session_write_close()`. But besides that, it registers a shutdown handler. For this handler it rembers the state of the `$_SESSION` variable and compares it with the state of this variable at shutdown. If these states do not match, a PHP error is thrown to inform you about this.

{% highlight php %}
/**
 * Closes writing to the current session and register a check to throw a warning
 * when something has been written to the session
 *
 * This will notify a developer if something is written to the session, while
 * the session was actually closed. Perhaps a developer did not know this, and
 * would not get warned by PHP's session_write_close function
 */
function register_session_write_close()
{
  // Close writing to the current session
  session_write_close();

  // Register a shutdown function to check if the SESSION was changed,
  // We compare by sending the session as it is now to the session as it is at shutdown
  // if the session seems changed, trigger a PHP warning
  register_shutdown_function( function( $oldSession )
  {
    if( $_SESSION !== $oldSession )
    {
      // Place the error message at the top of the page
      print '<div style="position:absolute;top:0;">';
      trigger_error( 'Session was changed after calling register_session_Write_close', E_USER_WARNING );
    }
  }, $_SESSION );
}
{% endhighlight %}