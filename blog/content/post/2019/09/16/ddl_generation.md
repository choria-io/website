---
title: "Generating DDL Files"
date: 2019-09-16T9:00:00+01:00
tags: ["ddl", "ux"]
draft: false
---

DDL files have been a utter bane of existence for MCollective users - and since Choria provide a compatability layer likewise for Choria users. It's not limited to Choria though most remote invocation systems have these kinds of files and honestly they are all horrible - think WSDL, OpenAPI/Swagger etc, editing and maintaining any of these is really horrible.

I have some plans to improve the situation, as much as we reasonably can, the first of which will land in our next release - `0.12.1` - which revolves around generating these files interactively.

<!--more-->

## Editing JSON DDL files

Our JSON DDL files are described in a [JSON Schema](http://choria.io/schemas/mcorpc/ddl/v1/agent.json) schema and many editors support this.  If you configure your editor to associate it with our files you get nice code completion, descriptions of keys and validation.

![completion](completion.png)

## Generating DDL files

First we now support interactively generating DDL files, you can see this in action below. It will ask questions, explain what the options mean and just generally guide you through the process of adding actions, inputs and outputs.

<script id="asciicast-268302" src="https://asciinema.org/a/268302.js" async></script>

You access this with the command `choria tool generate ddl package.json package.ddl`.

## Conversion from JSON to Ruby

In time we hope to make the ruby ones entirely optional, for now since we have a good way to edit the JSON DDL files thanks to the Schemas and a nice way to generate them, you can now also convert from JSON to Ruby.

The command `choria tool generate ddl package.json package.ddl` will, if the `package.json` already exist, prompt you if you want to convert it to Ruby rather than generate a new one again.

## Status

This feature is a bit new and by it's nature - generating Ruby code - prone to some error, let us know if you note any issues or missing features.