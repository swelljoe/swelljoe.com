---
title: "The Trickiest Thing About Perl (for beginners)"
date: 2017-07-07T18:23:09-05:00
draft: false
---

I was recently talking with our awesome UI/UX designer, Ilia, about a problem he was having with Perl. He mostly develops in JavaScript and web technologies, but his front-end obviously has to interact sometimes with the Perl in Webmin. And, sometimes he has to write some Perl, which is not an area he has had any experience with before working with us. Perl can be a tricky language for beginners to the language because of a few quirks that make it behave in seemingly bizarre ways.

I'm going to be a little bit repetitive in this article, as I approach the problem from a couple of different angles, and rewrite the same example over and over, to try to really demonstrate in a concrete way how Perl handles function arguments. It's a fundamental concept that has to be understood to be successful with Perl, and it's a source of a lot of confusion for a lot of developers new to Perl.

# The Baffling Case of Argument Order

Our story began when I received a ping from Ilia in the middle of the night with the following question:

![The Flattening](/img/perl-flat-chat.png)

In summary, if you don't want to read through all of that, he wanted his function, `get_extended_sysinfo` to receive two arguments: an array named `@info` and a scalar named `$new_string`. So, he did what any programmer might do, and added it to the arguments definition at the start of his function like so:

```perl
my ( @info, $new_string ) = @_;
```

Simple enough, right? Wrong.

This doesn't do what Ilia thought it would do (and what many programmers would think it would do), because Perl doesn't handle function arguments the way some other languages do. When you call a Perl function, Perl sends a *list of values* to the function. It doesn't send variables to the function, it always sends values and they're always flattened into a list. Also important in this discussion is the fact that arrays and hashes in Perl are "slurpy" or greedy. They will consume everything in the list, even if there are other variables after it that could receive some of the values.

So, if `@info` contained `(1, 2, 3, 5, 8, 13)` and `$new_string` contained `"doot doot"` calling `get_extended_sysinfo(@info, $new_string)` would actually pass the following list of values to the function:

```perl
(1, 2, 3, 5, 8, 13, "doot doot")
```

Weird, right? And, after you parse your arguments into named variables, `@info` would contain the entirety of that list, and `$new_string` would contain nothing! Ilia's bafflement over this is understandable (and it's something that caused me a lot of grief when I started learning Perl, too).

# Another Example

I was having a hard time explaining what the implications of this are, so I wrote up another concrete example using a hash and two strings, to show exactly what it means that Perl flattens complex data structures into lists when calling functions with them. I'd suggest you read over this, make a guess about what it'll output, and then read on to see what it actually does.

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use 5.010;

my $extrakey = "doot";
my $extraval = "toot";

my %hash = (
  name => 'Joe',
  age => 'Grumpy',
);

sub myFunction {
  my %hash = @_;

  say $hash{name};
  say $hash{doot};
  say $hash{age};
}

myFunction($extrakey, $extraval, %hash);
```

OK, have you thought about it? Got some theories about what it'll do?

This will print the following:

```bash
Joe
toot
Grumpy
```

Did you catch what happened there? When we passed in `$extrakey` and `$extraval` and received them in the `%hash` variable, Perl helpfully included the *values* of those variables in the newly generated `%hash` (the `%hash` in myFunction is a completely different variable from the `%hash` defined earlier...Perl has block scope when you use `my`).

It's important to understand that if we were to dump the output of `%hash` outside of myFunction, it would still only contain `name` and `age` fields. We really are not passing variables into the function, we are passing the *values* within the variables into the function in a list, and Perl tries to parse it into whatever data structure you've asked it to put them in.

At risk of harping on this too long, I want to make one more point. What if we add a third scalar variable to our function call? Now our function call looks like:

```perl
my $whatisthis = "WTF?";
myFunction($extrakey, $extraval, $whatisthis, %hash);
```

And our argument parsing stays the same:

```perl
my %hash = @_;
```

What happens if we run this new code?

```bash
Odd number of elements in hash assignment at perl-flat.pl line 16.
Use of uninitialized value in say at perl-flat.pl line 18.

toot
Use of uninitialized value in say at perl-flat.pl line 20.
```

![What in tarnation?](/img/what-in-tarnation-gif.gif)

What in initialization?

So, we're now seeing another side effect of the way Perl handles arguments. Once again, I'll harp on the the fact that Perl sends a *list of values* to the function. What Perl does with that list of values when handed into a hash is to try to create a hash out of it; in order to create a hash, it needs to have an even number of elements. It may be helpful here to remember that the `=>` syntax in Perl is known as a "fat comma". It is (mostly) syntactically equivalent to using a comma. You can define the above hash like so:

```perl
my %hash = ( 'name', 'Joe', 'age', 'Grumpy' );
```

Notice I had to put quote marks around the keys; otherwise, I would have gotten a run-time error, since I have strict turned on, if I'd left them as bare words. The fat comma forces the left-hand item to be interpreted as a string, so bare words can be used (and it is considered good style to use the fat command and bare-word keys, except when quotes are necessary).

Coming back to our function call example, the version with three strings and a hash being passed in now evaluates to the following definition for `%hash`:

```perl
my %hash = ('doot', 'toot', 'WTF?', 'name', 'Joe', 'age', 'Grumpy');
```

This has the silly effect of making `'WTF?'` and `'Joe'` into keys with values of `'name'` and `'age'` respectively, and `'Grumpy'` gets nothing because it's not part of a pair, and a hash must contain key=>value pairs. Notice in our above program output, we still got `'toot'` to print out, because the `'doot'` key doesn't get mixed up by the random `'WTF?'` in the middle. Obviously this is not the data structure we intended.

# So, Perl is completely useless for complex data then, right?

Nope. There *is* a way to pass complex data types into a Perl function, without them being flattened into lists of values. It's done by making a value that refers to your data structure. Such a value is called, surprisingly enough, a "reference".

You can take a reference of any variable in Perl, using the `\` operator. So, `\%hash` provides a reference to `%hash`. C programmers might think of it like a pointer to a variable (they'd be wrong to think of it that way, but not terribly wrong, and I suspect most C programmers aren't tripped up too badly by the way Perl handles data in function calls, since there are some similarities).

We can rewrite our example as follows:

```perl
#!/usr/bin/env perl
use strict;
use warnings;
use 5.010;

my $extrakey = "doot";
my $extraval = "toot";
my $whatisthis = "WTF?";

my %hash = (
  name => 'Joe',
  age => 'Grumpy',
);

sub myFunction {
  my ($extrakey, $extraval, $whatisthis, $hash_ref) = @_;

  say $hash_ref->{name};
  say $hash_ref->{age};
  say $extrakey;
  say $extraval;
  say $whatisthis;
}

myFunction($extrakey, $extraval, $whatisthis, \%hash);
```

There's one new piece of syntax here with the deref operator, `->`. There's actually more than one way to do it (as always in Perl). We could also have used `$$hash_ref{name}` or `${$hash_ref}{name}` and it would have worked, as well. (There's tons more options with references, but I won't dive too deep into it, for fear of getting lost in the weeds. If you want to read about it in more depth, see [perlreftut](http://perldoc.perl.org/perlreftut.html).)

I should also point out that the variable containing the reference could be named anything; I append `_ref` to the end of my reference variable names for my own sanity, but it could be named anything, including just calling it `$hash`.

We could even reverse the order of variables passed into our function:

```perl
myFunction(\%hash, $whatisthis, $extraval, $extrakey);
```

And, change our argument extraction in myFunction:

```perl
my ($hash_ref, $whatisthis, $extraval, $extrakey) = @_;
```

And everything else in our function would work the same, including the keys and values in the hash, and the other values into their own variables. This works because a reference is a single value, which refers to the actual data structure we're working on. There's nothing to flatten...it's just a value that tells the compiler where to find the actual data structure, and the receiving scalar isn't slurpy.

# Are We Done Here?

Not quite. There are some other things to be aware of when using references. One of the most important things to know is that because a reference merely points to a variable and does not actually contain the value(s) of the variable, if your function makes changes without explicitly creating a new data structure for the value(s) it will actually change the referred-to variable; effectively crossing scope boundaries without warning.

So, I could add the following line to `myFunction` in the above example:

```perl
$hash_ref->{name} = 'SwellJoe';
```

And, it would change `$hash{name}` in the outer scope to 'SwellJoe'.

This effect is often used intentionally to pass global data around within a program; it provides a little bit of encapsulation (by convention only, though). I've often seen it used for a configuration hash, for instance. I'm not endorsing this usage, as it is effectively a global variable, but it is an idiom you'll see often (including in Webmin since it is a 20 year old codebase). More modern programs might use an actual object for the same purpose, possibly with accessor methods for fully encapsulated data.

To avoid that, we could explicitly create a new hash containing the values in the variable referred to be `$hash_ref`, like so:

```perl
my %private_hash = %$hash_ref;
```

Now, `%private_hash` is only defined within the `myFunction` block, and changes to it will not affect `%hash` in the outer scope. I could add keys and values, change existing values, etc.

Again, there's more than one way to do it, but, all of this is pretty common in Perl code you'll find in the wild. There's something new on the horizon, but you won't really see it a lot for a couple more years, but I'll touch on it briefly.

# Subroutine Signatures

Perl 5.20 (finally) added subroutine signatures to core Perl (there have been CPAN modules to add it for many years, but they were rarely used, and often had some tradeoffs in performance or correctness). I won't go into detail about them, but will mention that they don't fundamentally change how one passes around complex data in Perl.

With signatures, arrays and hashes are still greedy or "slurpy", in that they'll suck up all the rest of your arguments, no matter what other variables are in the signature after the greedy variable. And, the arguments are still passed to the function as a list of values.

For example (you'll need Perl 5.20 or above for this example, you can use [plenv](https://github.com/tokuhirom/plenv) or [perlbrew](https://perlbrew.pl/) to easily install newer Perl versions in your home directory):

```perl
#/usr/bin/env perl
use strict;
use warnings;
use v5.20;
use feature qw(signatures);
no warnings qw(experimental::signatures);

my $extrakey = "doot";
my $extraval = "toot";

my %hash = (
  name => 'Joe',
  age => 'Grumpy',
);

sub myFunction (%hash, $extrakey, $extraval) {
  say $hash{name};
  say $hash{doot};
  say $hash{age};
}

myFunction(%hash, $extrakey, $extraval);
```

That won't work the way we intend; luckily Perl will catch this problem and complain:

```bash
Slurpy parameter not last at perl-flat-sigs.pl line 16.
Execution of perl-flat-sigs.pl aborted due to compilation errors.
```

So, that's probably the trickiest thing about Perl for beginners. If you made it this far and understand the basics of argument passing and references, you're probably gonna be OK programming in Perl.

# See also

[Perl data types](http://perldoc.perl.org/perldata.html)

[Perl references tutorial](http://perldoc.perl.org/perlreftut.html)

[Using 5.20 subroutine signatures](https://www.effectiveperlprogramming.com/2015/04/use-v5-20-subroutine-signatures/)
