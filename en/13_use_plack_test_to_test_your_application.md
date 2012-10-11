## Day 13: use Plack::Test to test your application

### Testing

Two common ways of testing web applications are to use a live server or to use a mock request technique. Some web application frameworks allow you to write unit tests using those methods, but the way you write tests differs by framework.

Plack::Test gives you *a unified interface* to test *any* web applications and frameworks that is compatible with PSGI using *both* mock requests and a live HTTP server.

### Using Plack::Test

Using Plack::Test is simple and compatible with Perl's standard testing protocol [TAP](http://testanything.org/wiki/) and the [Test::More](http://search.cpan.org/perloc?Test::More) module.

    use Plack::Test;
    use Test::More;
    use HTTP::Request;
    
    my $app = sub {
        return [ 200, [ 'Content-Type', 'text/plain' ], [ "Hello" ] ];
    };
    
    test_psgi $app, sub {
        my $cb = shift;
        
        my $req = HTTP::Request->new(GET => 'http://localhost/');
        my $res = $cb->($req);
        
        is $res->code, 200;
        is $res->content, "Hello";
    };
    
    done_testing;

Create and load the PSGI application as usual (you can use [Plack::Util](http://search.cpan.org/perldoc?Plack::Util)'s `load_psgi` function if you want to load an app from a `.psgi` file) and call the `test_psgi` function to test the application. The second argument is a callback that acts as a testing client.

You can use named parameters as well, like the following:

    test_psgi app => $app, client => sub { ... }

The client code takes a callback (`$cb`) which you can pass an HTTP::Request object that returns a HTTP::Response object like a normal LWP::UserAgent would do. You can make as many requests as you want and test various attributes and response details.

Save the code as a `.t` file and use a tool such as `prove` to run the tests.

### use HTTP::Request::Common

It's not required, but it's recommended that you use [HTTP::Request::Common](http://search.cpan.org/perldoc?HTTP::Request::Common) when making an HTTP request since it's more obvious and requires less code.

    use HTTP::Request::Common;
    
    test_psgi $app, sub {
        my $cb = shift;
        my $res = $cb->(GET "/");
        # ...
    };

Notice that you can even omit the scheme and hostname, which default to http://localhost/.

### Run in a server/mock mode

By default the `test_psgi` function's callback runs in a *Mock HTTP* request mode turning a HTTP::Request object into a PSGI env hash, running the PSGI application, and returning the response as a HTTP::Response object.

You can change this to live HTTP mode, by setting either the package variable `$Plack::Test::Impl` or the environment variable `PLACK_TEST_IMPL` to the string `Server`.

    use Plack::Test;
    $Plack::Test::Impl = "Server";
    
    test_psgi ... # the same code

By using the environment variable, you don't really need to change the .t code:

    env PLACK_TEST_IMPL=Server prove -l t/test.t

This will run the PSGI application using the Standalone server backend and will use LWP::UserAgent to send live HTTP requests. You don't need to modify your testing client code, and the callback will automatically adjust host names and port numbers depending on the test configuration.

### Test your web application framework with Plack::Test

Once again, the beauty of PSGI and Plack is that everything written to run on the PSGI interface can be used for *any* web application framework that speaks PSGI. By [running your web application framework in PSGI mode](http://advent.plackperl.org/2009/12/day-7-use-web-application-framework-in-psgi.html), you can also use Plack::Test:

    use Plack::Test;
    use MyCatalystApp;
    
    my $app = MyCatalystApp->psgi_app;
    
    test_psgi $app, sub {
        my $cb = shift;
        # ...
    };
    done_testing;

You can do the same thing for any framework that supports PSGI.
