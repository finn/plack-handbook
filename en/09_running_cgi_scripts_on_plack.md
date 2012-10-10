## Day 9: Running CGI scripts on Plack

For a couple of days we've been talking about how to convert existing CGI based applications to PSGI and then run them as a PSGI application. Today we'll show you the ultimate way to run *any* CGI script as a PSGI application, usually without modification.

[CGI::PSGI](http://search.cpan.org/perldoc?CGI::PSGI) is a subclass of CGI.pm to allow an easy migration from CGI.pm to PSGI with only *a few lines of code changed*. But what about a messy or legacy CGI script that just prints to STDOUT?

[CGI::Emulate::PSGI](http://search.cpan.org/perldoc?CGI::Emulate::PSGI) is a module to run any CGI based Perl program in a PSGI environment. Any messy/old script that prints directly to STDOUT or reads HTTP headers from `%ENV` should just work because that's what CGI::Emulate::PSGI emulates. The original POD of CGI::Emulate::PSGI illustrated it's use like so:

    use CGI::Emulate::PSGI;
    CGI::Emulate::PSGI->handler(sub {
        do "/path/to/foo.cgi";
        CGI::initialize_globals() if &CGI::initialize_globals;
    });

This runs existing CGI application that may or may not use CGI.pm. (CGI.pm caches lots of environment variables so it needs the `initialize_globals()` call to clear out the previous request variables).

On a flight from San Francisco to London to attend London Perl Workshop I hacked on something more intelligent that would take any CGI script and compile it into a subroutine. The module is named [CGI::Compile](http://search.cpan.org/perldoc?CGI::Comple) and is best used combined with CGI::Emulate::PSGI.

    my $sub = CGI::Compile->compile("/path/to/script.cgi");
    my $app = CGI::Emulate::PSGI->handler($sub);

There's also the [Plack::App::CGIBin](http://search.cpan.org/perldoc?Plack::App::CGIBin) Plack application to run existing CGI scripts written in Perl as PSGI applications. Suppose you have bunch of CGI scripts in `/path/to/cgi-bin`, you can run the server with:

    > plackup -MPlack::App::CGIBin -e 'Plack::App::CGIBin->new(root => "/path/to/cgi-bin")'

and it will mount the path `/path/to/cgi-bin`. Suppose you have `foo.pl` in that directory, you can access http://localhost:5000/foo.pl to run the CGI application as a PSGI over plackup just like scripts run using the Apache mod_perl Registry mechanism.
