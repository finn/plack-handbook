## Day 12: Maps multiple apps with mount and URLMap

### Hello World! but anyone else?

Throughout the advent calendar we've mostly used a simple "Hello World" web application as an example:

```perl
my $app = sub {
    return [ 200, [], [ "Hello World" ] ];
};
```

What about more complex code? For instance you might have multiple applications each of which inherit from a different web application framework and use Apache magic like mod_alias.

### Plack::App::URLMap

Plack::App::URLMap allows you to *composite* multiple PSGI applications into one application and to dispatch requests to multiple applications using the URL path or even using virtual host based dispatch.

```perl
my $app1 = sub {
    return [ 200, [], [ "Hello John" ] ];
};

my $app2 = sub {
    return [ 200, [], [ "Hello Bob" ] ];
};
```

We have two apps, one to say hi to John and another to Bob, and we want to run these two applications on the same server. With Plack::App::URLMap you can do this:

```perl
use Plack::App::URLMap;
my $urlmap = Plack::App::URLMap->new;
$urlmap->mount("/john" => $app1);
$urlmap->mount("/bob"  => $app2);
my $app = $urlmap->to_app;
```

There you go. The app now dispatches all requests beginning with `/john` to `$app1`, which says "Hello John", and `/bob` to `$app2`, which says "Hello Bob". As a result, all requests to unmapped paths such as root ("/") give a 404.

Environment variables such as `PATH_INFO` and `SCRIPT_NAME` are automatically adjusted so it works the same as when your application is mounted using Apache's mod_alias or CGI scripts. Your application framework should always use `PATH_INFO` to dispatch requests and concatenate with `SCRIPT_NAME` to build links.

### mount in DSL

The `mount` interface of Plack::App::URLMap is quite useful so we decided to add it to the Plack::Builder DSL, again inspired by Rack::Builder:

```perl
use Plack::Builder;
builder {
    mount "/john" => $app1;
    mount "/bob"  => builder {
        enable "Auth::Basic", authenticator => ...;
        $app2;
    };
};
```

Requests to '/john' are handled exactly the same way as the normal URLMap. But this example uses `builder` for "/bob" so it enables basic authentication to display the "Hello Bob" page. This should be syntactically equivalent to:

```perl
$app = Plack::App::URLMap->new;
$app->mount("/john", $app1);

$app2 = Plack::Middleware::Auth::Basic->wrap($app2, authenticator => ...);
$app->mount("/bob",  $app2);
```

but, obviously, with less code to write and more easily understood syntax.

### Multi tenant frameworks

Of course you can use the URLMap mount API to run multiple framework applications on one server. Imagine you have three applications: "Foo" which is based on Catalyst, "Bar" which is based on CGI::Application, and "Baz" which is based on Squatting. Do this:

```perl
# Catalyst
use Foo;
my $app1 = Foo->psgi_app;

# CGI::Application
use Bar;
use CGI::Application::PSGI;
my $app2 = sub {
    my $app = Bar->new({ QUERY => CGI::PSGI->new(shift) });
    CGI::Application::PSGI->run($app);
};

# Squatting
use Baz 'On::PSGI';
Baz->init;
my $app3 = sub { Baz->psgi(shift) };

builder {
    mount "/foo" => $app1;
    mount "/bar" => $app2;
    mount "/baz" => $app3;
};
```

And now you have three applications, each of which inherit from different web framework, running on the same server (via plackup or other Plack::Handler::* implementations), mapped on different paths.
