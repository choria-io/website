---
title: "Fisk AI"
date: 2026-07-18T00:00:00+01:00
tags: [ "fisk-ai", "ai", "releases" ]
draft: false
---

Today I would like to introduce an AI Harness I've been working on for the last 2 or 3 months
called [Fisk AI](https://choria-io.github.io/fisk-ai/).

There are many fantastic, all powerful, all featureful AI Harnesses out there like Claude Code, Pi, Hermess, Codex, Open
Code, and many more. At the same time there's been a movement toward creating harnesses that solve just one problem and
do so well and safely.

Fisk AI is a Harness framework that allows you to build *Specialized AI Agents* with just some YAML files.

- Agentic loop in TUI and Shell form
- Easiest possible way to create tools reusing the CLI tools you already have and, optionally, expose those over MCP
- Built-in Memory System
- Built-in Knowledge base (RAG)

Fisk AI can be used entirely locally on your laptop with locally hosted models or can use any AI host that supports the
Anthropic API (most do).

Data safety and tool safety is at the forefront, big harnesses have tons of capabilities and love to try and please you.
It is not uncommon to see Claude run half-page long Shell scripts, Python scripts, jq scripts, or combinations of these
in rapid succession. Mistakes are inevitable and so are non-deterministic outcomes. Claude ran Bashism on my ZSH and did
`rm -rf /`by acccident.

There is no reason to have this power around and a LLM that's keen to go the extra mile when you have a specific problem
to solve, repeatedly and reliably. This is where Specialized AI comes in. Give the AI just the tools and guidance it needs, 
no generic shell access or ability to run arbitrary code. That's what Fisk AI is for. Better, safer, and more reliable outcomes
for tasks you wish to run regularly with LLM assistance.

Down the line there is a broader goal around enterprise AI and infrastructure Ops. For now I am focussing on getting the
libraries and behaviours right, later the libraries that built Fisk AI will become the basis for infrastructure tools onto of the
Choria protocol which will bring strong Identity, Authentication, Authorization, and Auditing to Agent-2-Agent communications.

Read the full entry for examples and more details.

<!--more-->

## Tools

For an Agent to be useful, it needs to be able to interact with the world. The way that this is done is using Tools, and thus far creating tools has been a challenge. You have to write a purpose built service that you run and connect to your agents.

With Fisk AI we support automatically turning any CLI tool written with the [fisk](https://github.com/choria-io/fisk) CLI library into tools for an Agent.

Here is a tool built using [App Builder](https://choria-io.github.io/appbuilder/) that allows the agent to mark a pull request as triaged:

```
commands:
  - name: triage
    description: Marks a Pull Request as triaged
    type: exec
    flags:
      - name: pr
        description: The PR to act on
        type: integer
        required: true
    command: |
      gh pr edit -R you/repo {{.Flags.pr | escape }} --add-label triage
```

Note the various safety features here:

 - We validate that what is being given by the LLM is not going to be trying to inject shell-injection attack code since it must be an integer
 - We escape the argument when calling the `gh` utility ensuring no quoting problems (tough not needed for numbers)
 - We require the input is supplied
 - We wrap the `gh` tool, the LLM never have full access to `gh` only the key behaviors we set and it cannot act on any other repo or pass any flags
 - We can expand this to wrap any other CLI tool from any vendor and expose just the key aspects we need the LLM to use

Fisk AI turns this into a tool, with strict schema, and runs that in the agent loop. No extra services or MCP servers needed by running `fisk-ai mcp`.

If you just like to make tools for other AI in this manner, Fisk AI can also host those in a MCP server that you can call from Claude Code and elsewhere.

You can also implement your tools using the [fisk](https://github.com/choria-io/fisk) library in Go for when you want to use library packages etc:

```
var name string

func main() {
	vm := fisk.New("vm", "Virtual machine manager")
	
    stop := vm.Command("stop", "Stops a virtual machine").Action(stop)
	stop.Flag("name","Virtual machine to stop").Required().StringVar(&name)
	
	// ...
	
	app.MustParseWithUsage(os.Args[1:])
}

func stop(_ *fisk.ParseContext) error {
	// validate and interact with the VM
}
```
The interesting thing here is that these tools are standalone shell tools, you can use them in your day-to-day work and also share them with the AI, they are multi modal.

```
$ abt pr triage --help
usage: abt pr triage [<flags>]

Marks a PR needs human review

Flags:
  --pr=PR  The PR to act on
```

## Knowledge Base

RAG is how an AI model learns information that it did not have access to during training. Such as your personal documents, wiki, Obsidian vault etc

Fisk AI makes creating a AI-enabled knowledge base really easy, you need no extra binaries or services for a basic full-text-search-based RAG system - though you can also, optionally, run an embedding model locally under `ollama` and get full Vector search capabiilties.

Below is a sample agent that has access to the OpenVox documentation and Puppet language reference.

```
$ fisk-ai knowledge index

# .... 1 minute later

$ fisk-ai knowledge status
tier: hybrid (FTS5 + vectors, RRF) - model=text-embedding-embeddinggemma-300m dim=768

store:      knowledge/abt/knowledge.db
documents:  705
chunks:     3625
vectors:    3625
...
```

See below for the fully working, fully local, zero-dependency RAG system and assistant built in less than a minute. There is a video of it in use [on YouTube](https://youtu.be/IdlFSgvmDXQ).

I have a lot I want to achieve here but hopefully this conveys the idea, goals and I hope the guardrails and safety features will help some sceptics of this technology feel they have a safe way to use it - or just to achieve greater control over your outcomes when using LLMs for specific tasks.

See the [full project documentation](https://choria-io.github.io/fisk-ai/) for more details.


## Puppet Q&A

This is the entire agent configuration file, I use `LM Studio` to run `qwen/qwen3.5-35b-a3b` and `text-embedding-embeddinggemma-300m` locally on my MacBook Pro and nothing ever leaves this laptop.

```
# agent.yaml
harness:
  # opt-in to the knowledge system
  knowledge:
    enabled: true
    paths:
      - puppet-specifications/language
      - openvox-docs/docs/_openvox_8x
      - types.md
    # opt-in to the vector search system, but this is optional
    embeddings:
      base_url: http://127.0.0.1:1234/v1
      model: text-embedding-embeddinggemma-300m

llm:
  # A local model
  model: qwen/qwen3.5-35b-a3b
  # Budget controls in case you want to use Publically hosted models like those from Anthropic
  budget:
    max_tokens: 100000
    max_iterations: 50

system_prompt: |
  You help users based on a knowledge base about 
  Puppet, generally you should assume questions
  are about Puppet and how to solve problems 
  using Puppet rather than OS level management tools

  When responding to queries prefer local sources 
  over your trained knowledge and  include your
  citations in a table at the end number the 
  citations and reference the citation by number.

  Show the citation in the paragraph text in the 
  form `(#1)` so that users know exactly where the
  information comes from

  Here's an example

    Answer sentence (#1)

  |Citation |Reference                         |
  |---------|----------------------------------|
  |   #1    | reference path - reference title |
```

Here it is answering a question about Puppet with full citations:

> What is the correct way to structure a complex module installing, configuring, and starting some software?

### Correct Module Structure for Install, Configure, and Start

The best practice approach for structuring a complex Puppet module that installs, configures, and starts a service follows a **three-class pattern** (#1, #5):

#### Core Classes

**1. Main Class (`module::init.pp`)** (#5)
This is the entry point for your module, located in `init.pp`. It should be the only parameterized class and contain sensible defaults:

```
class chrony (
  Array[Stdlib::IP::Address] $bindaddress = [],
  Array[String] $bindcmdaddress = ['127.0.0.1', '::1'],
  Stdlib::Unixpath $config = '/etc/chrony/chrony.conf',
  # ... more parameters
) {
  contain chrony::install
  contain chrony::config
  contain chrony::service

  Class['chrony::install']
    -> Class['chrony::config']
    ~> Class['chrony::service']
}
```

**2. Install Class (`module::install.pp`)** (#7)
Contains all resources related to getting the software onto the node:

```
class chrony::install {
  assert_private()

  package { 'chrony':
    ensure   => $chrony::package_ensure,
    name     => $chrony::package_name,
  }
}
```

**3. Config Class (`module::config.pp`)** (#8)
Contains all resources for configuring the installed software:

```
class chrony::config {
  assert_private()

  file { $chrony::config:
    ensure  => file,
    owner   => 0,
    group   => 0,
    mode    => $chrony::config_mode,
    content => epp($chrony::config_template, {...}),
  }
}
```

**4. Service Class (`module::service.pp`)** (#9)
Contains service resources and anything related to the running state:

```
class chrony::service {
  assert_private()

  if $chrony::service_manage {
    service { $chrony::service_name:
      ensure => $chrony::service_ensure,
      enable => $chrony::service_enable,
    }
  }
}
```

### Key Principles

**Ordering** (#15, #16)
- Base all order-related dependencies on **classes rather than resources**
- Use `contain` to declare subordinate classes so other modules can form ordering relationships with your module
- Chain the classes logically: install → config → service

**Private Classes** (#7)
- All subordinate classes (`install`, `config`, `service`) should call `assert_private()` to prevent direct declaration by users
- Only the main module class should be publicly declared

**Naming Convention** (#2, #4)
- Each manifest should contain only one class or defined type
- File names map to class names: `apache/manifests/install.pp` contains `apache::install`

| Citation | Reference                                                            |
|----------|----------------------------------------------------------------------|
| #1       | docs/_openvox_8x/bgtm.md - Structuring your module                   |
| #2       | docs/_openvox_8x/cheatsheet_module.md - Module cheat sheet manifests |
| #4       | docs/_openvox_8x/cheatsheet_module.md - apache/manifests/vhost.pp    |
| #5       | docs/_openvox_8x/bgtm.md - Class design module                       |
| #7       | docs/_openvox_8x/bgtm.md - Class design module::install              |
| #8       | docs/_openvox_8x/bgtm.md - Class design module::config               |
| #9       | docs/_openvox_8x/bgtm.md - Class design module::service              |
| #15      | docs/_openvox_8x/bgtm.md - Structuring your module Ordering          |
| #16      | docs/_openvox_8x/bgtm.md - Containment and anchoring                 |