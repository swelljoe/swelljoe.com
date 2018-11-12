---
title: "Perl CLI: Subcommands With GetOpt::Long"
date: 2018-11-12T16:01:10-06:00
draft: true
---

Recently, I've been working on a CLI tool for Webmin (cleverly called `webmin`, to match the existing `virtualmin` and `cloudmin` CLI tools) that supports sub-commands. This is an extremely common user interface concept, and most developers have probably used one (git being an extremely popular example). But, it wasn't obvious how to do it with Perl using only core modules. There are several CPAN modules that'll do it, including [GetOpt::Long::Subcommand](https://metacpan.org/pod/Getopt::Long::Subcommand) and [App::Cmd](https://metacpan.org/pod/App::Cmd), and they both look good for their intended audience, but I didn't want to pull in any extra dependencies and I had somewhat different requirements that made it simpler to just use GetOpt::Long (which has been a core Perl module for many years).

The docs only hint at it, but it already has the ability to handle subcommand style options, though you have to write a bit of extra code. But, frankly, it's not as complicated as I expected...the GetOpt::Long developers provided a very good baseline set of functionality, and provided a clever escape hatch for folks who need something more.

==Requirements

As mentioned, I have somewhat uncommon requirements for my program:

  - Support for short (-h) and long (--help) options, obviously
  - Subcommands with global and subcommand options
	- Subcommands are not hard-coded, they are generated at run-time based on installed modules in Webmin
	- Subcommands are separate programs (runnable directly or loadable as a library by the top-level command)
	- Subcommands must receive the global options, for things like config file paths
	- No non-core modules, though sometimes a pure-Perl module is included if there's no other good option

The non-core modules requirement often seems weird to Perl folks, since the CPAN contains so much great stuff, but Webmin has such a huge installed base, across such a huge number of Linux and UNIX versions, and needs to be robust to things like upgrading Perl and library versions, migrations, etc. every new dependency has a pretty big impact, so we're slow to add dependencies when they can be avoided.

==Basic Subcommand Handling

==Validating Subcommands

Subcommands can be provided by Webmin modules themselves, so the `webmin` command is extensible without touching the code of the command itself. There are hundreds of Webmin modules, many are maintained by Jamie, while others are maintained by third parties, so it would be difficult for us to anticipate every option and every subcommand that a module might want to provide.

So, our goal is to accept arbitrary subcommands and options and validate them based on what is currently installed rather than based on a fixed list in the `webmin` command itself. To do that, we'll poll the filesystem looking for a `bin` subdirectory inside each Webmin module. If there is a matching command inside a module bin direcoty, we then pass the options to it, and the subcommand can provide additional validation of those options. If there is no matching command, an error and usage text will be printed.

There are a few things to be wary of with this kind of design. It would be easy to end up with conflicting subcommands, unless we impose some sort of naming scheme. For example, most services configurable in Webmin have a stop and start function, so it's like there will be start and stop subcommands in multiple modules. So, we'll solve that by simply including the module name in subcommands. e.g. a `start` subcommand in the Apache module (in the directory `webmin/apache/bin`) would be called with `apache-start`.
