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

```go
type Config struct {
    // LogLevel is the logging level
    //
    // @doc The lowest level log to add to the logfile
    LogLevel string `confkey:"loglevel" default:"info" validate:"enum=debug,info,warn,error,fatal" deprecated:"1" url:"http://example.com"`
    
    // The file to write logs to, when set to an empty string logging will be to the console
    LogFile string `confkey:"logfile" type:"path_string" description:"The log file"`
    
    Servers  []string `confkey:"servers" type:"comma_split" environment:"SERVERS"` // Servers to connect to
}
```

You'd set these in configuration file like:

```ini
loglevel = debug
logfile = /var/log/choria.log
```

To tie this together we have a package `github.com/choria-io/go-choria/confkey` that can operate on a structure - though not parse the files, it's just for struct manipulation:

```go
c := &Config{}

// Initialize the struct with default values
err := confkey.SetStructDefaults(c)
panicIfErr(err)

// Next we have to set complex types - number, bool, slices etc - from strings as the data comes from the configuration file as text
// the conversions are all handled by the confkey package

// Set the value in the struct for whatever field matches the key loglevel 
err = confkey.SetStructFieldWithKey(c, "loglevel", "error")
panicIfErr(err)

// automatically handles the split etc, can be overridden using environment variable SERVERS
err = confkey.SetStructFieldWithKey(c, "servers", "h1.example.net:4222, h2.example.net:4222")
panicIfErr(err)

// validate all the inputs matches validate rules, like the enum list
err = confkey.Validate(c)
panicIfErr(err)
```

You can extract various related items from the struct tags:

```go
c := &Config{}

// a list of regex matching fields, would host ["LogLevel", "LogFile"]
fields, _ := confkey.FindFields(c, "log")

// The value of the "description" tag on the field tagged as "logfile", 
// not the comments, also have Validation(), DefaultString(), Environment(), 
// IsDeprecated() and Type()
fmt.Printf("Description: %s\n", confkey.Description(c, "logfile"))
```

For validations we support a lot of systems related things, times, enums, ip addresses, v4 address, v6 address, max length, regex and shell injection protection.  This is all supported by the `github.com/choria-io/go-choria/validator` package.

Finally you can extract a `confkey.Doc` object for any Field:

```go
c := &Config{}

doc := confkey.KeyDoc(c, "loglevel", "Config")

// Description of the field, also have Type(), URL(), Default(), Validation(), 
// Deprecate(), Environment(), ConfigKey() and StructKey()
fmt.Printf("Description: %s\n", doc.Description())
```

## Extracting comments

With the above we can quite easily parse the INI file format and load the values into our Go structs at run time, magical data type conversion and Environment overrides are supported etc.  We even have comments in the struct like in the `LogFile` case.

I wanted to though support normal comment blocks since writing longer descriptions isn't practical in those struct tags.

```go
type Config struct {
    // LogLevel is the logging level
    //
    // @doc The lowest level log to add to the logfile
    LogLevel string `confkey:"loglevel" default:"info" validate:"enum=debug,info,warn,error,fatal" deprecated:"1" url:"http://example.com"`
    
    // The file to write logs to, when set to an empty string logging will be to the console
    LogFile string `confkey:"logfile" type:"path_string" description:"The log file"`
}
```

Above we have the `LogLevel` field with Programmer comments and everything past the `@doc` would be shown to users, the `LogFile` would take the longer comment string and not `The log file` when shown to the user.  So this means we have to look at the code around compile time.

Essentially I want to end up with a bit of code generated that looks like this:

```go
package config

var docStrings = map[string]string{
    "loglevel": "The lowest level log to add to the logfile",
    "logfile": "The file to write logs to, when set to an empty string logging will be to the console",
}
```

I can then easily override the `Description()` from `confkey` when there is a matching `docStrings` entry.

The Go team has made the Go parser accessible via the package `go/parser`, and it's quite easy to use, here I read `config.go` and parse out only the `Config` structure and it's comments.  I've specifically kept it a bit sparse for the sake of blog post length, the full [gen.go](https://github.com/choria-io/go-choria/blob/c0fedce0aab9215d9c158def3a37ffcbeeaa4a40/config/gen.go) shows how the `docStrings` are made.

```go
// reads config.go and includes parsing of comments
d, err := parser.ParseFile(token.NewFileSet(), "config.go", nil, parser.ParseComments)
panicIfErr(err)

ast.Inspect(d, func(n ast.Node) bool {
    // this function filters and parses the ast, return false to stop descending into a
    // branch of the AST, so we skip over everything till we reach Config struct

    switch t := n.(type) {
    case *ast.TypeSpec:
        // once we reach our Config structure keep going deeper, else skip over
        return t.Name.Name == "Config"
    case *ast.StructType:
        // every field in Config
        for _, field := range t.Fields.List {
            tag := ""
            doc := ""

            // get the tag and clean it up a bit, we need to only touch "confkeys" not other stuff
            if field.Tag != nil && field.Tag.Kind == token.STRING {
            	tag = strings.TrimRight(strings.TrimLeft(field.Tag.Value, "`"), "`")
            }

            // extract the comments
            switch {
            case len(strings.TrimSpace(field.Doc.Text())) > 0:   // comments above the field
                doc = field.Doc.Text()
            
            case len(strings.TrimSpace(field.Comment.Text())) > 0:  // comments after the field like in "Servers"
                doc = field.Comment.Text()
            }

            if strings.Contains(tag, "confkey") && doc != "" {
                // process the doc and generate the above docStrings
            }
        }
    }

    return true
})
```

From here it's just grunt work to generate [CLI](https://github.com/choria-io/go-choria/blob/c0fedce0aab9215d9c158def3a37ffcbeeaa4a40/cmd/tool_config.go#L46-L109) and [Markdown](https://github.com/choria-io/go-choria/blob/c0fedce0aab9215d9c158def3a37ffcbeeaa4a40/gen_config_doc.go) outputs.

This was quite easy, certainly a lot easier than any other time I tried to mess around with code parsers so this was quite a nice experience, kudos to the Go team.
