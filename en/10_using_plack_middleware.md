## Day 10: Using Plack middleware

### Middleware

Middleware is a concept in PSGI (as always, stolen from Python's WSGI and Ruby's Rack) where a component defines both sides of the server and application interface.

![WSGI middleware onion](../images/pylons_as_onion.png)

(Image courtesy of Pylons project for Python WSGI)

This picture illustrates the middleware concept very well. The PSGI application is in the core of the onion layers and middleware components wrap the original application. They preprocess as a request comes in (outer to inner) and then postprocess as a response goes out (inner to outer).

Lots of functionality can be added to a PSGI application by wrapping it in a middleware component: HTTP authentication, capturing errors to log output, wrapping JSON output with JSONP, etc.

### Plack::Middleware

[Plack::Middleware](http://search.cpan.org/perldoc?Plack::Middleware) is a base class for middleware components that allows you to write middleware in a simple and reusable fashion.

Using middleware components written with Plack::Middleware is easy, just wrap the original application with the `wrap` method:

    my $app = sub { [ 200, ... ] };

    use Plack::Middleware::StackTrace;
    $app = Plack::Middleware::StackTrace->wrap($app);

This example wraps the original application with the StackTrace middleware (which is actually enabled [by default using plackup](http://advent.plackperl.org/2009/12/day-3-using-plackup.html)) using the `wrap` method. When the wrapped application throws an error the middleware component catches the error and [displays a beautiful HTML page](http://bulknews.typepad.com/blog/2009/10/develstacktraceashtml.html) using Devel::StackTrace::AsHTML.

Some middleware components take parameters, passed as a hash after `$app`:

    my $app = sub { ... };

    use Plack::Middleware::MethodOverride;
    $app = Plack::Middleware::MethodOverride->wrap($app, header => 'X-Method');

Installing multiple middleware components is tedious especially since you need to `use` those modules first, but we have a quick solution for that using a DSL style syntax.

    use Plack::Builder;
    my $app = sub { ... };

    builder {
        enable "StackTrace";
        enable "MethodOverride", header => 'X-Method';
        enable "Deflater";
        $app;
    };

We'll see more about Plack::Builder tomorrow.

### Middleware and Frameworks

The beauty of middleware is that it can wrap *any* PSGI application. It might not be obvious from the code examples, but the wrapped application can be anything, which means you can [run your existing web application in the PSGI mode](http://advent.plackperl.org/2009/12/day-7-use-web-application-framework-in-psgi.html) and apply middleware components to it. For instance, with CGI::Application:

    use CGI::Application::PSGI;
    use WebApp;

    my $app = sub {
        my $env = shift;
        my $app = WebApp->new({ QUERY => CGI::PSGI->new($env) });
        CGI::Application::PSGI->run($app);
    };

    use Plack::Builder;
    builder {
        enable "Auth::Basic", authenticator => sub { $_[1] eq 'foobar' };
        $app;
    };

This will enable the Basic authentication middleware for a CGI::Application based application. You can do the same with [any other frameworks that supports PSGI](http://plackperl.org/#frameworks).
