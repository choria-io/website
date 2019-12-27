+++
title = "Inputs"
weight = 410
+++

Playbooks can take arguments that are effectively Inputs into your Playbook.

A basic example would be:

```puppet
plan example::upgrade (
  String $version,
  Enum[alpha, bravo] $cluster = "alpha"
) {
  # use $version and $cluster
}
```

As these are just Puppet variables the type handling will deal with validating your inputs - like the Enum here - and all the usual behaviours remain.  We will add a way to use the MCollective Validation plugins as types so you can get things like shell safe validation for free.

There's a nice trick to use a setting store for setting you often need but that are annoying to type like API tokens:

```puppet
plan example::slack (
  Optional[String] $api_key = undef,
  String $message
) {
  $ds = {
    "type" => "file",
    "file" => "~/.plans.rc",
    "format" => "yaml",
    "create" => true
  }

  # Store if given, lookup if not
  $token = $api_key ? {
    String => choria::data("slack.key", $api_key, $ds),
    default => choria::data("slack.key", $ds)
  }

  unless $token {
    fail("A Slack API token is needed")
  }

  # use the token
}
```

Here we mark the *$api_key* as optional and if it's given we store it into the *~/.plans.rc*, if not given we attempt to read it from the same file.  This way you only have to provide it once and future runs of the playbook will reuse it.  The same can be done with other persistent stores like *consul* and *etcd* in which case it could be team wide behavior.

[Data Stores](../data/) are an advanced topic and covered extensively in the dedicated [Data Stores](../data/) page.
