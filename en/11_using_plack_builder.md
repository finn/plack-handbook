## Day 11: Using Plack::Builder

[Yesterday](http://advent.plackperl.org/2009/12/day-10-using-plack-middleware.html) we saw how to enable Plack middleware components in a .psgi file using the `wrap` method. But the the technique of loading the middleware with `use` and then wrapping the `$app` with `wrap` is tedious and not very intuitive. So we have a DSL (domain specific language) called Plack::Builder to make it much easier.

### Using Plack::Builder

Using Plack::Builder is easy. Just use the keywords `builder` and `enable`:

```perl
my $app = sub {
    return [ 200, [], [ "Hello World" ] ];
};

use Plack::Builder;
builder {
    enable "JSONP";
    enable "Auth::Basic", authenticator => sub { ... };
    enable "Deflater";
    $app;
};
```

This takes the original application (`$app`) and wraps it with the Deflater, Auth::Basic, and JSONP middleware components (inner to outer). It's equivalent to:

```perl
$app = Plack::Middleware::Deflater->wrap($app);
$app = Plack::Middleware::Auth::Basic->wrap($app, authenticator => sub { });
$app = Plack::Middleware::JSONP->wrap($app);
```

but without lots of `use`ing the module which is anti DRY.

### Outer to Inner, Top to the bottom

Did you notice that the order of middleware wrapping is in reverse? The builder/enable DSL allows you to *wrap* an application so the middleware in the line closest to the original `$app` is *inner* while the first one at the top is *outer*. You can compare that with [the onion picture](http://pylonshq.com/docs/en/0.9.7/_images/pylons_as_onion.png) which should make it more obvious: something closer to the application is inner.

`enable` takes the middleware name without the Plack::Middleware:: prefix but if you want to enable some other namespace such as MyFramework::PSGI::MW::Foo you can write:

```perl
enable "+MyFramework::PSGI::MW::Foo";
```

The key here is to use the plus (`+`) sign to indicate that it is a fully qualified class name.

### What's happening behind the scenes

If you're curious what Plack::Builder is doing take a look at the code and see what's going on. The `builder` takes the code block and executes the code and takes the result (return value of the last statement) as an original application (`$app`). It then returns the wrapped application by applying middleware in the reverse order. This means it is important to have `$app` in the last line inside the `builder` block and have the `builder` statement as the final statement in the .psgi file as well.

### Thanks, Rack

The idea of Plack::Builder was totally inspired by [Rack::Builder](http://m.onkey.org/2008/11/18/ruby-on-rack-2-rack-builder). You can see that they use the `use` keyword, but obviously we can't *use* that in Perl :) so we chose `enable` instead. You can see that they also use `map` to map applications to a different path. We'll talk about its equivalent in Plack tomorrow. ;)
