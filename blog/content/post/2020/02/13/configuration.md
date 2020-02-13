---
title: "Choria Configuration"
date: 2020-02-13T13:28:00+01:00
tags: ["documentation"]
draft: false
---

Choria configuration, despite efforts with Puppet module and so, is still very challenging.  Just knowing what settings are available has been a problem.

We tried to hide much of the complexity behind Puppet models but for people who don't conform to the norm it's been a challenge.

I eventually want to move to a new configuration format - perhaps HCL? - but this is a massive undertaking both for me and users.  For now we've made some effort to give insights to all the known configuration settings on the CLI and in our documentation.

First we'll publish a generated configuration reference in [CONFIGURATION.md](https://github.com/choria-io/go-choria/blob/master/CONFIGURATION.md) - for now it's in the Git repository we'll move it to the doc site eventually.  

As of the upcoming version of Choria Server you'll be able to query the CLI for any setting using regular expressions. The list will show descriptions, data types, validation rules, default values, deprecation hints and URLs to additional information.

![choria tool config](/blog/post/2020/02/13/configuration/cli.png)

And get a list:

```nohiglight
$ choria tool config puppet -l
plugin.choria.puppetca_host
plugin.choria.puppetca_port
plugin.choria.puppetdb_host
plugin.choria.puppetdb_port
plugin.choria.puppetserver_host
plugin.choria.puppetserver_port
```

These references are extracted from the Go code - something that I never imagine is possible - read on for details on how that is done.

<!--more-->
## Setting, reading, introspecting and validating struct fields

In our code a typical Configuration data item looks something like this:

{{< gist ripienaar 655b47ae325e9be36aca29b1aaa8f25e >}}

You'd set these in configuration file like:

```ini
loglevel = debug
logfile = /var/log/choria.log
```

To tie this together we have a package `github.com/choria-io/go-choria/confkey` that can operate on a structure - though not parse the files, it's just for struct manipulation:

{{< gist ripienaar 882a4b929cc562042f5e964c5c604a43 >}}

You can extract various related items from the struct tags:

{{< gist ripienaar 4dcf1d5a490bdc2f529ff8e15b6cd07f >}}

For validations we support a lot of systems related things, times, enums, ip addresses, v4 address, v6 address, max length, regex and shell injection protection.  This is all supported by the `github.com/choria-io/go-choria/validator` package.

Finally you can extract a `confkey.Doc` object for any Field:

{{< gist ripienaar 4dcf1d5a490bdc2f529ff8e15b6cd07f >}}

## Extracting comments

With the above we can quite easily parse the INI file format and load the values into our Go structs at run time, magical data type conversion and Environment overrides are supported etc.  We even have comments in the struct like in the `LogFile` case.

I wanted to though support normal comment blocks since writing longer descriptions isn't practical in those struct tags.

{{< gist ripienaar 6a9da85b33532c52930ebeb720b09384 >}}

Above we have the `LogLevel` field with Programmer comments and everything past the `@doc` would be shown to users, the `LogFile` would take the longer comment string and not `The log file` when shown to the user.  So this means we have to look at the code around compile time.

Essentially I want to end up with a bit of code generated that looks like this:

{{< gist ripienaar b3e7547f442a4bc941e24afac5a1190e >}}

I can then easily override the `Description()` from `confkey` when there is a matching `docStrings` entry.

The Go team has made the Go parser accessible via the package `go/parser`, and it's quite easy to use, here I read `config.go` and parse out only the `Config` structure and it's comments.  I've specifically kept it a bit sparse for the sake of blog post length, the full [gen.go](https://github.com/choria-io/go-choria/blob/c0fedce0aab9215d9c158def3a37ffcbeeaa4a40/config/gen.go) shows how the `docStrings` are made.

{{< gist ripienaar 3ca3501d0bf26710cc45c9c61ab11667 >}}

From here it's just grunt work to generate [CLI](https://github.com/choria-io/go-choria/blob/c0fedce0aab9215d9c158def3a37ffcbeeaa4a40/cmd/tool_config.go#L46-L109) and [Markdown](https://github.com/choria-io/go-choria/blob/c0fedce0aab9215d9c158def3a37ffcbeeaa4a40/gen_config_doc.go) outputs.

This was quite easy, certainly a lot easier than any other time I tried to mess around with code parsers so this was quite a nice experience, kudos to the Go team.
