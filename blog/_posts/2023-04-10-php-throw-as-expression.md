---
layout: post
author: "Dennis Claassen"
title: "PHP8+ change: throw as an expression"
---

With PHP8, the throw statement has changed into an expression. This might not sound like much, but allows you to use throw expressions in new situations. All the details can be found [in the RFC](https://wiki.php.net/rfc/throw_expression).

## What’s the difference between a statement and an expression?

The main difference is where you can use a statement and where you can use an expression.

## What are the benefits?

It allows you to improve the type safety of your code, by throwing exceptions immediately. Let’s imagine a callable function with the following API:

{% highlight php %}
callable(string):\DateTimeInterface
{% endhighlight %}

Read more about this syntax at https://psalm.dev/docs/annotating_code/type_syntax/callable_types/.

We can create two function that match this API:

{% highlight php %}
$immutableFactory = fn(string $date): DateTimeInterface => DateTimeImmutable::createFromFormat('Y-m-d H:i:s', $date);

$mutableFactory = fn(string $date): DateTimeInterface => DateTime::createFromFormat('Y-m-d H:i:s', $date);
{% endhighlight %}

Do these functions match the specified API?
Reading https://www.php.net/manual/en/datetime.createfromformat.php shows that the `createFromFormat` methods can also return a `false` value. So, lets write them out to full functions with a type check to throw exceptions.

{% highlight php %}
$immutableFactory = function(string $input): \DateTimeInterface {
    $date = \DateTimeImmutable::createFromFormat('Y-m-d H:i:s', $input);
  if ( !( $date instanceof \DateTimeInterface ) ) {
    throw new \DomainException( ‘Invalid date provided.’ );
  }

  return $date;
}
{% endhighlight %}

That’s a perfectly fine solution. It works well, matches the callable type, and deals with any possibly invalid input. You might notice I only transformed the `$immutableFactory` and went from 1 to 8 lines. Not a big deal, clarity is often preferred over brevity. In the end, code is more often read than written. So a bit longer code that’s easier to understand is a good thing. But originally I showed 2 factories. That would mean going from 2 to 16 lines.

### `throw`as expression with a ternary operator

I was defining these methods in a data provider for my unit tests, where I was using them to quickly create `DateTimeInterface` objects for my tests. But a data provider should be about providing the data. I didn’t want to clutter it with 16 lines of boilerplate code, distracting from the actual tests data. So I decided to use the throw expression to simplify the code and hide its complexity on a single line. Because `createFromFormat` returns `false`, it allows using a short ternary (https://www.php.net/manual/en/language.operators.comparison.php) to either return the successfully created object, or to run a different expression. With `throw` now being an expression, it can be the right hand side of this short ternary.

{% highlight php %}
$immutableFactory = fn(string $date): DateTimeInterface => DateTimeImmutable::createFromFormat('Y-m-d H:i:s', $date) ?: throw new \DomainException('invalid date');

$mutableFactory = fn(string $date): DateTimeInterface => DateTime::createFromFormat('Y-m-d H:i:s', $date) ?: throw new \DomainException('invalid date');
{% endhighlight %}


### `throw` as expression in a logical operator

Besides using a ternary to trigger the `throw` expression, you can also trigger it with an `or` operator or the `||` operator (src:https://www.php.net/manual/en/language.operators.logical.php). This doesn’t go together with the short function syntax, because the `or` operator and the `||` operator both result in the function returning a `bool` instead of a `DateTimeInterface`. For variable assignments the `or` operator can however be used. I personally get a sentimental feeling from using the `or` operator for this, because it reminds me of the good old `mysql_connect() or die(‘…’);` statement. However, type-wise the `||` statement matches better because it doesn’t assign the invalid result of the `createFromFormat` method to the variable, sadly the result with this operator is always a boolean instead of the `DateTimeInterface` in this example. There is however a catch with the `or` operator, because the precedence of a variable assignment with the `or`operator results in the following:

{% highlight php %}
( $date = false ) or throw new \DomainException(‘’);
{% endhighlight %}

With the `||` operator, the precedence is slightly different and the assignment isn’t performed. Too bad the result is always casted to a Boolean.

{% highlight php %}
 $date = ( false || throw new \DomainException('') );
{% endhighlight %}

You might wonder what’s the catch. Usually a `throw` expression breaks the flow, so you can’t access the variable anymore with that unexpected type. But, with a `try .. finally` block you’re still able to access the variable. In such a situation, you might be surprised by the value of the variable after the assignment.

Lets have one last look at the long-form function we wrote earlier, but this time we'll use the short syntax with the `or` operator and wrap it in a `try .. catch .. finally` block.

{% highlight php %}
$immutableFactory = function(string $input): \DateTimeInterface {
  try {
    $date = \DateTimeImmutable::createFromFormat('Y-m-d H:i:s', $input) or throw new \DomainException( ‘Invalid date provided.’ );
    // When we get here, $date is a DateTimeImmutable.
    return $date;
  }
  catch( \Throwable $t ) {
    // When we get htere, $date is actually false.
    return null;
  }
  finally {
    // Here, $date can be either false or a DateTimeImmutable.
  }
}
{% endhighlight %}

So, there's the catch. A really small one, but always good to be aware of these little peculiarities. If you'll ever run into this, you might just realize what's happening a second faster.
