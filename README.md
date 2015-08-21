# Courseware Learning VM

## Synopsis

The Puppet Learning VM (Virtual Machine) is an artisinal, open-source,
learning environment for the discerning Puppet user.

This repository contains the base collection of quests pre-installed
on the Learning VM.

While these quests are written with certain filepaths and settings specific
to Puppet Enterprise (PE) in mind, a user interested in Puppet Open Source will
find the majority of the content easily adaptable.

## Quest content

The prose content of quests is written in (Markdown)[http://daringfireball.net/projects/markdown/].
For formatting not supported in the Markdown standard, we use the Liquid template
system. Specific liquid templates available are covered below. 

### Title and Headers

Each quest should start with the title of that quest as an h1. In Markdown,
h1 is designated with a single `#` at the beginning of the lines, e.g.:

	# Resource Ordering
	
Next is a Quest Objectives section under an h2 with that name:

	## Quest Objectives
	
In this section, include a brief one paragraph summary of the quest
objectives. At the end of this section, include the a sentence like
"If you're ready to get started, type the following command:",
followed by the start quest command, indented to display as a code
block, and with the quest name as its argument. This argument
refers to the spec file for that quest, so replace any spaces
with underscores to match the spec filename.

    quest --start resource_ordering
    
After this, begin the main content of the quest, using h2 and h3
level headers as appropriate.

### Liquid templates

We use the Liquid templating system to extend the syntax available
in Markdown and allow us to include asides and custom-formatted
task notation.

#### Code Highlighting

Blocks of code must be wrapped in `{% highlight <language> %} {% endhighlight %}`
tags, like so:

	{% highlight puppet %}
	user { 'root':
  		ensure => 'present',
  }
  {% endhighlight %}

#### Asides

There are three types of aside that will be displayed floating
to the right of the text. Each has an icon according to its
type and will include the content between the template tags.
This content should be kept brief, and you should avoid using
too many of these in close succession to avoid clutter.

```
{% tip %}
Here's a tip!
{% endtip %}

{% warning %}
Here's a warning!
{% endwarning %}

{% fact %}
Here's a fact!
{% endfact %}
```

There is also an inline-aside style. This is displayed in the
main body of the text, but is set off from the flow of the document.
Unlike the asides above, this can include a title, which can include
spaces. (Some special characters work here, but this hasn't been
tested, so be sure to double check that your content renders properly
before making a PR.)

```
{% aside Title of aside %}
Content of the aside.
{% endaside %}
```

#### Tasks

(This testing system is deprecated!)

Finally, there is a template to be used in numbering tasks and specifying
completion steps for the automated testing system. Place this before
each task. Because task numbering is associated directly with the VM
task tracking function, you must enter each task number manually.

The content of the task template is a block of YAML that scripts
the steps required to complete a task. See the Rakefile for the
code that implements this content.

The YAML content is parsed into a list of steps. A step can either
specify a command to execute and an optional list of inputs to supply
if that command opens a subprocess, or give a filename and the content
for that file.

An excute command will look something like this:

```
{% task 2 %}
---
- execute: "puppet describe user | less"
  input:
    - 'q'
{% endtask %}
```

Note that if you open a subprocess, you must also include an
input that will exit that process. In the example above,
we send the `q` command to exit `less`.

A File command will look like this:

```
{% task 6 %}
---
- file: /etc/puppetlabs/puppet/environments/production/modules/vimrc/manifests/init.pp
  content: |
    class vimrc {
      file { '/root/.vimrc':
        ensure => 'present',
        source => 'puppet:///modules/vimrc/vimrc',
      }
    }
{% endtask %}
```

## Quest ordering

The `index.json` contains a list of quests in the order they will appear in the
navigation bar of the quest guide.
