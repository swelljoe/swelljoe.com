---
title: "Unit Testing and Travis CI For Bash Or POSIX Shell"
date: 2017-08-11T20:40:27-05:00
draft: false
---

Recently I've been working on getting some of our projects working with Travis CI, so that we'll know about regressions in our code. Our software is a a diverse collection of Perl, JavaScript, a tiny bit of Python and C, and an uncomfortably large amount of POSIX shell code. Things are easy for Perl, the language of choice for most of Virtualmin and Webmin: It is easy to test, has great modules, a great test runner, and great tools for parsing and presenting the output. And, of course, Travis CI has good support for it, too.

POSIX Shell, on the other hand, just isn't very easy to test. And, realistically, I want to write as little code in shell as possible, while still getting the job done. So, when it came time to start adding tests to our shell projects, I looked around at the options. There are TAP (Test Anything Protocol) generators for shell, some even look pretty good. But, they meant I'd be writing more shell scripts, and for anything over about a dozen lines long, I would rather do it in a different language. And, honestly, it seemed like it would take me longer to become proficient in them than it would take to write something entirely new (and very simple).

# Perl `prove` to the rescue

Instead of learning my way around a shell testing harness and library, I decided to use something I already know. Perl has excellent support for calling out to external programs and capturing output and return values, so it's easy to call shell scripts and capture the results.

Perl also has the `prove` command, which is a test runner that works with the TAP output provided by the various `Test::` modules in core and in CPAN. So, all I have to do for basic testing of shell scripts is write a shell script that calls it using backticks or the `system()` function, and output results based on that.

# Static Analysis

The primary shell program I maintain, the install script for Virtualmin, has been around for about a dozen years. It doesn't have very good tests, just some scripts to run the install on a virtual machine and check to be sure it completed and some important stuff actually got done. None of which is realistically usable in a CI environment we don't control, like Travis CI (spinning up a VM/container within the Travis CI VM or sshing out to some other VM on our infrastructure both seemed untenable options given the amount of time I wanted to devote to this), so I need new tests. The old testing process will stick around, but now we'll work on some new paths for code improvement that will work in a more restricted environment.

Static analysis is always the low-hanging fruit, and I take advantage of it whenever I can to improve code quality and spot errors without a lot of human labor.

## Shellcheck

I love `shellcheck`! It's been a boon for my project of converting all of our shell code to pretty clean, kinda readable, and reliable, POSIX-compliant shell. It also catches a lot of common errors and prevents entire classes of bugs from reaching production.

My first goal when beginning to test was to run `shellcheck` on every commit. And, it turned out to be really easy.

First, create a directory named `t`. This is the standard location for test code in Perl projects, and the Perl testing tools will look there for test code.

In that directory I added a dead simple Perl program, `shellcheck.t` that looks like this:

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use 5.010;

use Test::Simple tests => 1;

ok( system('shellcheck virtualmin-install.sh') == 0 );
```

That's basically one line of code, when excluding boilerplate, and now my ~900 line shell script gets pretty good static analysis run against it on every check in. That's pretty cool.

All it does is run shellcheck, and checks its return value...if it returns `0` (the "true" value for shell programs), then the program outputs OK, and if it returns anything else, the test fails.

## checkbashisms

Another useful static analysis tool for shell scripts is `checkbashisms`. This one *specifically* looks for use of bash idioms and functions. Since I've written a lot more bash code in my day than POSIX shell code, I tend to accidentally insert bashisms pretty regularly. And, while `shellcheck` catches some of them, it doesn't catch them all (likewise, checkbashisms misses some that `shellcheck` catches, so they're useful companions).

Using the exact same approach  as for `shellcheck`, we can write a simple test to run `checkbashisms` against our code, called `checkbasisms.t`:

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use 5.010;

use Test::Simple tests => 1;

ok( system('checkbashisms virtualmin-install.sh') == 0 );
```

Now, I get an email if I accidentally check-in something that includes bashisms or any common shell mistakes.

But, just as usefully, I can test my code locally with `prove`.

# Testing Locally

The standard test runner in Perl (among several) is `prove`. Run it from your project top-level directory, and it'll try to run all of your programs in the `t` subdirectory with a `.t` file extension. Those programs can do anything you want them to do, and as long as they return valid TAP output, `prove` will produce friendly output for you.

```bash
$ prove
t/checkbashisms.t .. ok   
t/shellcheck.t ..... ok   
All tests successful.
Files=2, Tests=2,  2 wallclock secs ( 0.01 usr  0.01 sys +  1.57 cusr  0.03 csys =  1.62 CPU)
Result: PASS
```

This also sets a return code of `0`, if all tests passed, and a non-zero value if any tests failed. This leads us to our next step.

# Enabling Travis CI

I'm gonna gloss over the usual Travis CI setup, as the Travis CI folks have good docs for hooking up github projects to Travis CI.

In our project top-level directory, we need a `.travis.yml` file that tells Travis CI how to test our code. And, it turns out this couldn't be any simpler, either.

```yaml
before_install:
- sudo apt-get update -qq
- sudo apt-get install -qq perl shellcheck devscripts
script: prove
```

This installs the three programs we need (Perl probably doesn't need to be specified, but Perl has gone away in the default installation of some Linux distros, so we'll prepare for that unlikely possibility), and then runs our tests with `prove` in the `script` field. Note that `checkbashisms` resides in the `devscripts` package.

Travis CI only cares about the return value, and `prove` provides exactly what we need.

So, there's a basic framework for testing shell scripts, including integration with Travis CI, and it requires almost no new code, since it leverages the already-excellent Perl testing facilities, and it has very few dependencies, since the Perl modules we're using are in core.

# Going Further

Now, static analysis is great, but it'd also be good to have unit and functional tests. In its simplest form you can test shell scripts by running them and capturing their output.

## A Simple Functional Test

For example, for a basic functional test, I can test to be sure the usage summary prints whenever `--help`, `-h`, or any invalid option is given:

 ```perl
 #!/usr/bin/env perl
 use strict;
 use warnings;
 use 5.010;

 use Test::Simple tests => 3;

 my @usage = `sh virtualmin-install.sh --help`;
 ok( grep { /^Usage:/ } @usage );
 @usage = `sh virtualmin-install.sh -h`;
 ok( grep { /^Usage:/ } @usage );
 @usage = `sh virtualmin-install.sh --invalid-option`;
 ok( grep { /^Usage:/ } @usage );
```

Unit testing the Virtualmin installer isn't currently possible because it can't be sourced the script without actually running it. But, I've pulled a large percentage, about 2/3rds, of the functions out into their own projects, which get merged into a function library called `slib` (for `shell library`, because I'm super creative), and this script *can* be sourced and unit tested.

## A Simple Unit Test

So, we have some code we want to source and then run functions within it to insure we get the right behavior. Perl can't do this directly, unfortunately; every time we run an external command with backticks or `system()`, it spawns a new shell, so the environment doesn't persist from one invocation to the next.

So, we need a shell script that sets up our environment and then runs whatever functions we want to test. This is also really simple (at least, for the limited needs I have). I made a script called `run.sh` and placed it inside the `t` directory:

```bash
#!/bin/sh

# Source it
. ./slib.sh

res="$($1)"
echo "$?"
echo "$res"
```

All this does is imports `slib.sh` into the environment, and then executes whatever argument we gave it. There's lots of ways this could be improved or made more robust, but we don't need it right now, so we're not gonna worry about it. This little program prints out the error code on one line and the output, if any, on the second line. We can capture both in our Perl script and work with whichever one we want, or both.

Now, I can write Perl code that calls any function in my shell library, like this `is_fully_qualified.t` test of the function that checks to see if a given domain name is fully qualified or not:

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use 5.010;

use Test::Simple tests => 2;

my @res = `sh t/run.sh 'is_fully_qualified localhost.localdomain'`;
ok( $res[0] != 0 );
@res = `sh t/run.sh 'is_fully_qualified dootdoot.com'`;
ok( $res[0] == 0 );
@res = `sh t/run.sh 'is_fully_qualified doot'`;
ok( $res[0] != 0 );

```

I wrap the arguments to `run.sh` in quotes to insure they're passed as one argument and won't be broken up (this is required to make sure it executes with the arguments.

So, now, I can make unit tests for many of the functions in my shell libraries. And, they all work with `prove`, so any new tests I add will automatically be checked by Travis CI as soon as I check them in.

As an aside, this would also be a reasonable way to begin porting shell scripts to some other language. It allows calling shell functions from within other languages, so functions can be rewritten in the new language over time. Of course, a lot of shell scripts don't offer pure functions, and rely on side effects throughout the execution of the code, and my code is no exception...so this isn't something I'll be experimenting with anytime soon. If I did, I'd probably wrap it up in something a little more elegant; one could abstract away the call to `run.sh` and automatically import all of the functions as Perl (or whatever language) functions.

# Testing With Other Languages

There's nothing unique about Perl that makes it work for this purpose. It's just the language we use for most of our projects, and so it's familiar, and we already have our editors and environments setup.

But, you could use the framework above with Ruby, or Python, or whatever scripting language you like. As long as it can call out to external commands, and has good testing tools, you could use it in place of Perl. The `run.sh` script would also work.

# Coverage Reports

One last little thing. A good testing framework will also have coverage reporting. Things like `Devel::Cover` won't work for shell scripts, obviously. But, we can whip up something stupid but functional with a regex to pull out the names of the functions in our library (now we've got two problems) and do the same for the functions called in our tests.

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use 5.010;
use List::MoreUtils qw(uniq);

my $file = $ARGV[0];
my $testdir = 't';

open my $handle, '<', $file;
chomp(my @lines = <$handle>);
close $handle;

# Get a list of functions that ideally would be tested
my @funcs;
for( @lines ) {
	my $func = $1 if m/^([a-zA-Z0-9\_]+)\W*\(\)\W*{/; # / Make highlighter happy
	push @funcs, $func if $func;
}

# Get a list of functions that have a test that calls them
my @tests = <./t/*>;
my @tested;
for my $test (@tests) {
	open my $handle, '<', $test;
	chomp(my @lines = <$handle>);
	close $handle;
	for ( @lines ) {
		my $func = $1 if m/run\.sh\W'([a-zA-Z0-9\_]+)\W.*/;
		push @tested, $func if $func;
	}
}

# Figure out what's missing
my @missing;
for my $f (@funcs) {
	push @missing, $f unless grep( /$f/, @tested );
}

# Print a report
my $coverage = sprintf '%.2f', $#tested / $#funcs * 100;
say "Test Coverage: $coverage%";
say "Untested functions:";
for my $f (@missing) {
	say $f;
}
```

I'm not going to go into too much detail about what this is doing, but it's a useful little script if using the above testing micro-framework. Perhaps most importantly for now, is that it's small enough to bundle with all of my scripts and shell libraries, since the total testing infrastructure only requires two small scripts. No need to make a package or add extra dependencies to the Travis CI configuration.

Ideally this would output something compatible with Devel::Cover or be able to export to Coveralls or similar test coverage tools. But, for my purposes, I've spent an afternoon on the problem of "we need better testing tools for our shell scripts", and there are many other projects demanding my time, so I'm gonna call it done for now.

# References

[Shellcheck](https://www.shellcheck.net/)

[checkbashisms](https://sourceforge.net/projects/checkbaskisms/)

[Perl Testing Tutorial](https://metacpan.org/pod/distribution/Test-Simple/lib/Test/Tutorial.pod)

[RegExr regular expression test tool](http://regexr.com/) - This was helpful in developing the regexes to parse out functions from POSIX shell scripts.
