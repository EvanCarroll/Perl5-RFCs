# Improving handling of different Perl versions

## Preamble

    Author:  Evan Carorll <me@evancarroll.com>
    Sponsor:
    ID:      00xx
    Status:  Pre-Draft Email

## Abstract

Accept a template string in environmental variables `PERL5LIB` and
`PERL_MM_OPT` which expands to the current perl version.

## Motivation

Perl should have support in the ecosystem for different versions of perl,
either because multiple versions are installed or because the system was
upgraded from one version of Perl to another. Currently using `local::lib` and
running a distribution upgrade can render the system non-functional. This
proposal provides a fix.

This RFC comes out of a need to resolve the [gotcha observed in this
StackOverflow write up](https://stackoverflow.com/q/74721452/124486).

## Rationale

In order to achieve version isolation, many distributions of Perl compile the
version number into the default paths which populate `@INC`. In Debian, for
example, perl modules distributed will install to `/usr/share/perl/5.36` which
includes the major and minor version number of Perl. This has the advantage
that during a dist-upgrade, a newer Perl can not read the XS compiled for an
older Perl because it will have a different default `@INC`. In order to ensure
that modules installed with `ExeUtils::MakeMaker` and `Module::Build` are
version aware, these modules read from the default `PRIVLIB`, another compile
time variable which typically includes the version number. In summary,

* Having set `@INC` in compile times ensures modules can be found in the
	distro's perl5 moudle install path which includes the version number.
* Having set `PRIVLIB` ensures that Perl build systems installs to a location
	which includes the version number (usually in `/usr/local/`).

The problem with the current solution is that ecosystems built around setting
`PERL5LIB`, and `PERL_MM_OPT` and others have to sacrifice version isolation.
These are environmental variables: they can be thought of as globally unique
for the user for the duration of the session. They're not set with a version
number.

Why aren't they set with a version number? Because they're environmental
variables. Consider what would happen if they were: when you upgrade your
distribution you'd have a newer copy of Perl potentially running in an
environment that has a `PERL5LIB` referencing an older perl's module directory.
In order to fix this, you'd have to close all your shells and log-out.  This
would not be ideal.

What we do NOT want is to have to manage the shell's profile
differently for each version of Perl. Instead we want the ability to set
`PERL5LIB` and the install path in the build system, using environmental
variables, to a location that includes the version of _the currently exected
perl_.

I propose we implement this by making `##` magical such that `##PERL_VERSION##`
is merely replaced when seen in either

* the environmental variable `PERL5LIB`
* the EU::MM build system.

Such as,

```
PERL5LIB="/home/ecarroll/perl5/##PERL_VERSION##/"
PERL_MM_OPT="INSTALL_BASE=/home/ecarroll/perl5/##PERL_VERSION##"; export PERL_MM_OPT;
```

Effectively being a small templating language we can improve on later.

## Alternatives

I think the only alternative which addresses this need is,

* Perl Brew or containerization
* for a different approach [`App::plx`](https://metacpan.org/pod/App::plx).

As Debian is still installed and packaged for many Linux Distros it seems
unlikely people will want to ignore the system Perl, and `local::lib` seems to
be the dominant solution for people who don't want to install CPAN modules as
root. This should facilitate a better user experience for `local::lib`
users.

## Specification

1. Patch Perl such that a path in `PERL5LIB` can be set with the currently
	 running Perl version, and doesn't have to be set at compile time or in the
	 environment. This proposal calls for doing this with a very simple string
	 interpolation for the magical `##PERL_VERSION##` on the path.
2. Patch EU::MM such that `INSTALL_BASE` can be set to the currently running
	 Perl version.

Given the above the following should install modules to and resolve names in
the `~/perl5/` under the Perl version,


```shell
export PERL5LIB="${HOME}/perl5/##PERL_VERSION##/";
export PERL_MM_OPT="INSTALL_BASE=${HOME}/perl5/##PERL_VERSION##";
```

## Other Considerations

This doesn't resolve the outstanding issue with the non-CORE `Module::Build`.
That will also have to be patched so their similar `--install-base` understands
`##PERL_VERSION##`.

## Copyright

Copyright (C) 2022, Evan Carroll

This document and code and documentation within it may be used, redistributed and/or modified under the same terms as Perl itself.
