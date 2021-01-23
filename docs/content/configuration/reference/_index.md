+++
title = "Configuration Reference"
toc = true
weight = 210
+++

Choria is configured using a set of text files that resembles key=val files. Values in the files are ordered as a hierarchy almost like a registry with `.` separating levels.

## File locations and names

Choria supports many different files for configuration, the complexity is in the Client - for server and broker you are required to supply a file.

### Client Configuration

For Clients a number of files are parsed giving you the ability to operate entirely without user level configuration or by mapping in additional configuration over the system wide ones on a per project basis.

|File|Description|Behavior|
|----|-----------|--------|
|~/.choriarc|Default location for user configuration|Loaded as the initial configuration, no merging with others|
|~/.mcollective|Backward compatible location for user configuration|Loaded as the initial configuration, no merging with others|
|/etc/choria/client.conf|System wide Choria client configuration|Loaded as the initial configuration, no merging with others|
|/usr/local/etc/choria/client.conf|System wide Choria client configuration|Loaded as the initial configuration, no merging with others|
|/etc/puppetlabs/mcollective/client.cfg|System wide Choria client configuration. **DEPRECATED**|Loaded as the initial configuration, no merging with others|

In the case of the last 3 locations a set of additional files are parsed, for example if the file is `/etc/choria/client.conf` then all files `/etc/choria/plugin.d` are parsed as plugin specific configuration.  See later section about plugin configuration.

After parsing the initial configuration a per-project configuration is supported by loading `choria.conf` in the current directory, its parent directory, its parent directory and so forth up to `/choria.conf` in reverse order. Settings found are merged over those from the system or user configurations.

```nohighlight
$ choria tool config
Configuration Files:

            User Config: /etc/choria/client.conf
   Project Confguration: /home/user/temp/choria.conf, /home/user/temp/project/choria.conf
```

### Server and Broker Configuration

Server and Brokers require the configuration to be supplied on the CLI using `--config`, these will always parse plugin configuration relative to the supplied configuration.

## Configuration Format

Choria configuration files are key=value pairs, they tend to be organised in a hierarchy like `plugin.security.provider`, this is almost like a directory tree of settings.

```ini
plugin.security.provider = puppet
```

Choria supports a few data types when parsing configuration according to this table:

|Type|Description|Go Type|
|----|-----------|-------|
|`comma_split`|List of strings split by comma with spaces around the values trimmed|`[]string`|
|`colon_split`|List of strings split by a colon with spaces around the values trimmed|`[]string`|
|`path_split`|List of strings split by the OS specific PATH separator with spaces around the values trimmed|`[]string`|
|`enum`|List of strings with prescribed possible values|`[]string`|
|`int`|A number like 1, 2, 3 etc, no quotes|`int`|
|`duration`|A duration such as `1h`, `300ms`, `-1.5h` or `2h45m`. Valid time units are `ns`, `us` (or `µs`), `ms`, `s`, `m`, `h`|`time.Duration`|
|`duration`|Numbers where the number is processed as 1 second, so a value of `60` is `60` seconds|`time.Duration`|
|`string`|Any normal string|`string`|
|`title_string`|Strings where the first character is turned into an upper case|`string`|
|`path_string`|Strings used for paths that is `~` aware|`string`|
|`bool`|`1`,`yes`, `true`, `y`, `t` for true and `0`, `no`, `false`, `n`, `f` for false|`bool`|

## Reference

All the known configuration keys can be queried using the `choria tool config` command, there are some unknowns though due to plugins having a dynamic nature, but the bulk of the settings can be found using the CLI.

First lets look at one config options:

```nohighlight
$ choria tool config loglevel
Configuration Files:

            User Config: /etc/choria/client.conf

Configuration item: loglevel

║        Value: warn
║    Data Type: string
║   Validation: enum=debug,info,warn,error,fatal
║      Default: info
║
║ The lowest level log to add to the logfile
╙─
```

Here we look at all configuration keys matching `loglevel` (regular expression supported), it shows us that:

 * Current in use value is `warn`
 * It's a string data type - could be any in the table above
 * It validates so that only the enum list is accepted as valid
 * When not specified it defaults to `info`
 * There's a short description.

In other examples you'll be warned if it's a deprecated setting, and you might also find links to further information. Try `choria tool config .` for all the configuration options.

An online version of this reference can be found in [GitHub](https://github.com/choria-io/go-choria/blob/master/CONFIGURATION.md) and is alwayd up to date.

Additionally, we can also just get a list of available options which might help you explore deeper:

```nohighlight
$ choria tool config --list
activate_agents
classesfile
collectives
color
....
ttl
```

We can also filter this list using regular expressions `choria tool config --list plugin`

## Plugin Configuration

The final piece of the puzzle is plugin configuration, you'll notice many configuration keys are like `plugin.security.provider`, these are traditionally not core `mcollective` configuration options, you can store these in settings files per plugin.

The above could be set in `/etc/choria/plugin.d/security.cfg` as `provider=puppet`.

These files are only parsed when the configuration is not one found in a home directory.
