# NAME

Test::Pod::Links - test Pod for invalid HTTP/S links

# VERSION

Version 0.003

# SYNOPSIS

    use Test::Pod::Links;
    Test::Pod::Links->new->all_pod_files_ok;

# DESCRIPTION

Tests that all HTTP/S links from Pod documentation are reachable by calling
the `head` method of [HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny) on them.

All non HTTP/S links are ignored. You can check them with
[Test::Pod::LinkCheck](https://metacpan.org/pod/Test::Pod::LinkCheck).

This test is an author test and should not run on end-user installations.
Recommendation is to put it into your `xt` instead of your `t` directory.

# USAGE

## new( \[ ARGS \] )

Returns a new `Test::Pod::Link` instance. `new` takes an optional hash
with its arguments.

    Test::Pod::Links->new(
        ignore       => 'url_to_ignore',
        ignore_match => qr{url to ignore},
        ua           => HTTP::Tiny->new,
    );

The following arguments are supported:

### ignore (optional)

The `ignore` argument is either a string or an array ref of strings. URLs
that match one of these strings are not checked. The comparison is done
case-sensitive.

This can be used to exclude URLs that are known to not work like
`http://www.example.com/`. (But if a link doesn't work it most likely
shouldn't be in an `L` tag anyway.)

### ignore\_match (optional)

The `ignore_match` argument is either a regex or an array ref of regexes.
URLs that match one of these regexes are not checked.

### ua (optional)

The `ua` argument is used to supply your own, [HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny) compatible,
user agent. Use this if you need a special configured [HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny) user
agent.

## pod\_file\_ok( FILENAME )

This will run a test for parsing the Pod and another test for every web link
found in the Pod. It is therefore unlikely to know the exact number of
tests that will run in advance. Use `done_testing` from [Test::More](https://metacpan.org/pod/Test::More) if
you call this test directly instead of a `plan`.

`pod_file_ok` returns something _true_ if all web links are reachable
and _false_ otherwise.

## all\_pod\_files\_ok

Calls the `all_files` method of [Test::XTFiles](https://metacpan.org/pod/Test::XTFiles) to get all the files to
be tested. Then, `contains_pod` from [Pod::Simple::Search](https://metacpan.org/pod/Pod::Simple::Search) is used to
identify files that contain Pod.

All files that contain Pod will be checked by calling `pod_file_ok`.

It calls `done_testing` or `skip_all` so you can't have already called
`plan`.

`all_pod_files_ok` returns something _true_ if all web links are reachable
and _false_ otherwise.

Please see [XT::Files](https://metacpan.org/pod/XT::Files) for how to configure the files to be checked.

WARNING: The API was changed with 0.003. Arguments to `all_pod_files_ok`
are now silently discarded and the method is now configured with
[XT::Files](https://metacpan.org/pod/XT::Files).

# EXAMPLES

## Example 1 Default usage

Check the web links in all files returned by [XT::Files](https://metacpan.org/pod/XT::Files).

    use 5.006;
    use strict;
    use warnings;

    use Test::Pod::Links;

    if ( exists $ENV{AUTOMATED_TESTING} ) {
        print "1..0 # SKIP these tests during AUTOMATED_TESTING\n";
        exit 0;
    }

    Test::Pod::Links->new->all_pod_files_ok;

## Example 2 Check non-default directories or files

Use the same test file as in Example 1 and create a `.xtfilesrc` config
file in the root directory of your distribution.

    [Dirs]
    module = lib
    module = tools
    module = corpus/hello

    [Files]
    pod = corpus/7_links.pod

## Example 3 Specify a different user agent for [HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny)

    use 5.006;
    use strict;
    use warnings;

    use Test::Pod::Links;
    use HTTP::Tiny;

    if ( exists $ENV{AUTOMATED_TESTING} ) {
        print "1..0 # SKIP these tests during AUTOMATED_TESTING\n";
        exit 0;
    }

    Test::Pod::Links->new(
        ua => HTTP::Tiny->new(
            agent => 'Mozilla/5.0',
        ),
    )->all_pod_files_ok;

## Example 4 Exclude a URL

    use 5.006;
    use strict;
    use warnings;

    use Test::Pod::Links;

    if ( exists $ENV{AUTOMATED_TESTING} ) {
        print "1..0 # SKIP these tests during AUTOMATED_TESTING\n";
        exit 0;
    }

    Test::Pod::Links->new(
        ignore => 'http://example.com/index.html',
    )->all_pod_files_ok;

## Example 5 Exclude all urls for a domain

    use 5.006;
    use strict;
    use warnings;

    use Test::Pod::Links;

    if ( exists $ENV{AUTOMATED_TESTING} ) {
        print "1..0 # SKIP these tests during AUTOMATED_TESTING\n";
        exit 0;
    }

    Test::Pod::Links->new(
        ignore_match => qr{
            # We are going to exclude every URL that uses a host in the
            # example.com domain.

            ^                           # begin of string
            (?: (?i) http (?:s)? )      # case-insensitive match of http
                                        # and https
            ://                         # the protocol delimiter
            (?: [^/]* [.] )?            # this matches any hostname (or none)
            example[.]com               # the domain example.com
            (?: / | $ )                 # After the domain we get either a /
                                        # or nothing
        }x,
    )->all_pod_files_ok;

## Example 6 Call `pod_file_ok` directly

    use 5.006;
    use strict;
    use warnings;

    use Test::More 0.88;
    use Test::Pod::Links;

    if ( exists $ENV{AUTOMATED_TESTING} ) {
        print "1..0 # SKIP these tests during AUTOMATED_TESTING\n";
        exit 0;
    }

    my $tpl = Test::Pod::Links->new;
    $tpl->pod_file_ok('corpus/7_links.pod');
    $tpl->pod_file_ok('corpus/hello');

    done_testing();

# RATIONALE

## Why this instead of [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s)?

This module is much like [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s). It checks that HTTP/S links
in your Pod are valid.

There are a few differences to [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s):

- [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s) does not cache the result. If you add a link to your
github repository in every `.pm` file it will verify the same link for
every module by connecting to the same URL again and again. That is slow,
excessive and not very nice to the web server. `Test::Pod::Links` caches the
result of every request only issuing a head request once for every URL.
- [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s) converts the Pod to text and then checks everything
that looks like a web URL which will pick up things that look like a URL but
are not a link. `Test::Pod::Links` only checks HTTP/S links inside an `L`
tag.
- `Test::Pod::Links` supports a `ua` argument with the `new` method that
allows you to pass a custom, [HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny) compatible, user agent to it. It
can also be used to configure [HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny) to your liking, e.g. configuring
the user-agent string.
- [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s) uses a hard coded list of hostnames to ignore, with
`Test::Pod::Links` you have the `ignore` and `ignore_match` option to
decide which URLs to skip over.

# SEE ALSO

[HTTP::Tiny](https://metacpan.org/pod/HTTP::Tiny), [Test::More](https://metacpan.org/pod/Test::More), [Test::Pod::LinkCheck](https://metacpan.org/pod/Test::Pod::LinkCheck), [Test::Pod::No404s](https://metacpan.org/pod/Test::Pod::No404s),
[XT::Files](https://metacpan.org/pod/XT::Files)

# SUPPORT

## Bugs / Feature Requests

Please report any bugs or feature requests through the issue tracker
at [https://github.com/skirmess/Test-Pod-Links/issues](https://github.com/skirmess/Test-Pod-Links/issues).
You will be notified automatically of any progress on your issue.

## Source Code

This is open source software. The code repository is available for
public review and contribution under the terms of the license.

[https://github.com/skirmess/Test-Pod-Links](https://github.com/skirmess/Test-Pod-Links)

    git clone https://github.com/skirmess/Test-Pod-Links.git

# AUTHOR

Sven Kirmess <sven.kirmess@kzone.ch>

# COPYRIGHT AND LICENSE

This software is Copyright (c) 2018-2020 by Sven Kirmess.

This is free software, licensed under:

    The (two-clause) FreeBSD License
