---
title: Application Orchestrator
layout: default
---

# Application Orchestrator

## Quest objectives

- Understand the role of orchestration in managing your infrastructure.
- Configure the Puppet Application Orchestration service and the Orchestrator
  client.
- Use Puppet code to define components and compose them into an application
  stack.
- Use the `puppet job run` command to apply your application across
  a group of nodes.

## Getting Started

If you manage applications comprised of multiple services distributed across multiple
nodes, you'll know that the orchestration of multiple nodes can pose some special
challenges. Puppet's Application Orchestrator extends Puppet's powerful declarative model
from the level of the single node to the complex application. Describe your app in puppet
code, and let the Application Orchestrator coordinate the rollout.

Before getting started, you should know that this quest will be a significant step
up in complexity from the ones that have come before it, both in terms of the concepts
involved and the varieties of tools and configurations you'll be working with. Keep in
mind that the Puppet Application Orchestrator is a new feature. Though it is already
powerful, it will continue to be improved, refined, and better integrated with the rest
of the Puppet ecosystem.

When you're ready to get started, type the following command:

    quest --start application_orchestrator

## Application orchestrator

> A quotation

> -The person who said it

To understand how the Application Orchestrator works, let's imagine a simple three
tier web application.

(IMAGE)

We have a single load balancer that distributes requests among three webservers, which
all connect to the same database server.

Each of the nodes involved in this application will have some configuration for things
not directly involved in the application. Things like sshd, and ntp will likely
be common to many nodes in your infrastructure, and Puppet won't require specific
information about the application the node is involved in to configure them correctly.
In addition to these classes and resources that are independent of the application,
each node in this example contains some components of the application: the webserver,
database, and load balancer along with whatever other resources are necessary to
support and configure their application-specific content and services.

(IMAGE)

This application specific configuration is called a component. In our example,
we define components for the database, webserver, and loadbalancer. Each
of which contains all the classes and resources necessary for a node to
fulfull its role in the application. A component is generally a defined resource
type, though it can also be a class or a single resource. A defined
resource type is flexible enough to include multiple resources and subclasses, and
unlike a class it can be declared multiple times within the same scope, allowing
our environment to have multiple instances of, for instance, the webserver component.

(IMAGE)

With all the components defined, we next define their relationships with one another
as an application. If your application is packaged as a module, this application
definition will generally go in your `init.pp` manifest.

(IMAGE)

The application definition tells these components how they'll communicate with
one another and allows the Puppet Application Orchestrator determine the order
of Puppet runs needed to correctly deploy the application to nodes in your
infrastructre.

This ordering of Puppet runs is a big part of how the tools in the Application
Orchestrator work. It requires a little more direct control over when and how
the puppet agent runs on the nodes involved in your application. If puppet runs
simply occured at the default scheduled interval of half and hour, we'd have no
way of ensuring that the components of our application would be configured in the
correct order. If, for example, the puppet run on our webserver happened to trigger
before that on the database server, a change to the database name would break our
application. Our webserver would still try to connect to the database from a previous
configuration, and would result in an error when that database wasn't available.

### Node Configuration

To avoid this kind of uncoordinated change, you must set the nodes involved in your
application to use a cached catalog when puppet runs. This allows puppet to run
as scheduled to avoid configuration drift, but will only make changes to the catalog
when you intentionally re-deploy your application. Similarly you must also disable
plugin sync to ensure that any changed functionality provided by plugins (e.g. functions
or providers) doesn't lead to uncoordinated changes to the nodes in your application.

{% task 1 %}
{% endtask %}

Of course, we could log in to each node and make the configuration change directly,
but why not use Puppet to configure Puppet? There is an `ini_setting` resource
that will let us make the necessary changes to the `use_cached_catalog` and
`pluginsync` settings on each agent's `puppet.conf` configuration file.

  vim /etc/puppetlabs/code/environments/production/manifests/site.pp

Because we want to apply this configuration to both the
`database.learning.puppetlabs.vm` and `webserver.learning.puppetlabs.vm` nodes,
we can use a regular expression in the node definition name to match both. Create a
new node group with the name `/^.+\.learning\.puppetlabs\.vm$/`, and include two
`ini_setting` resources for the configuration changes we need to make:

{% highlight puppet %}
node /^.+\.learning\.puppetlabs\.vm$/ {
  ini_setting { 'use_cached_catalog':
    ensure  => present,
    path    => $settings::config,
    section => 'agent',
    setting => 'use_cached_catalog',
    value   => 'true',
  }
  ini_setting { 'pluginsync':
    ensure  => present,
    path    => $settings::config,
    section => 'agent',
    setting => 'pluginsync',
    value   => 'false',
  }
} 
{% endhighlight %}

We can use the console to trigger puppet runs on our two nodes directly.
Navigate to your PE console by entering `https://<VM's IP ADDRESS>` in the
address bar of your browser. Log in with the following credentials:

* User: admin
* Password: puppetlabs

Go to the *Nodes* > *Inventory* section in the PE console.

Click on your `database.learning.puppetlabs.vm` node, and click on the **Run Puppet...*
button link and **Run** button to start your puppet run. You don't need to wait for
it to finish now. Return to the **Inventory** section and trigger a run on
`webserver.learning.puppetlabs.vm` as well. While these runs are in progress,
feel free to continue with the rest of this quest. We'll check in to make sure
they've completed correctly at the point when we need to apply code to the nodes
again.

### Master Configuration

Before we get to writing and deploying and application, however, there are a few
steps we still need to do to get the Puppet Application Orchestrator tools
configured correctly.

The Puppet Orchestrator tool we'll use in this quest is a command-line interface
that interacts with an Application Orchestration service that runs on the puppet
master. This service is not enabled by default, so our first step will be to
change the configuration of our puppet master to enable application services.

Once you're logged in to the console, navigate to the **Nodes** > **Classification**
section. Here, select the **PE Infrastructure** node group. This group defines the
configuration of for the components of the Puppet stack. Click the **Classes** tab,
and find the *use_application_services* parameter under the *puppet_enterprise*
class. Click the **edit** button to change the value of this parameter from
`false` to `true`. Commit your change.

Return to your terminal session on the puppet master (i.e. the Learning VM) and
trigger a puppet run. (Or, if you prefer, trigger a run via the PE console.)

  puppet agent -t

### Client Configuration and Permissions

There's one piece to put in place before you can use the Puppet Orchestrator tools.
While the Application Orchestration service runs on your puppet master, the
Puppet Orchestrator command-line tools can be run anywhere with network access to
the puppet master. This means that we need to take a few steps to ensure that this
communication goes smoothly.

We need to configure our client to tell it where to find the puppet master.
Remember, we could be running this client on any machine, even one outside of our
Puppetized infrastructure. This is why we have to set this configuration separately,
and can't simply rely the address of the puppet master from the puppet agent
configuration.

First, we need to create the directory structure where this configuration file
will live.

  mkdir -p ~/.puppetlabs/etc/puppet

Now create your orchestrator configuration file.

  vim ~/.puppetlabs/etc/puppet/orchestrator.conf

The file is formatted in JSON. We'll specify the following options

{% highlight json %}
{
  "options": {
    "url": "https://learning.puppetlabs.vm:8143",
    "environment": "production"
  }
}
{% endhighlight %}

(Note that these items could also be specified as flags from the command line.
Creating the configuration file saves you the trouble of typing them out each
time.)

With this configuration file created, the Puppet Orchestrator client knows where
the puppet master is, but the puppet master still needs to be able to verify that
the user running commands from the Puppet Orchestrator has the correct permissions.

This is achieved with PE's Role Based Access Control (RBAC) system, which we can
configure through the PE console.

Return to the PE console and find the **Access control** section in the left
navigation bar.

We will create a new `orchestrator` user and assign permissions to use the
application orchestrator.

Click on the **Users** section of the navigation bar. Add a new user with the
full name "Orchestrator" and login "orchestrator".

Now that this user exists, we need to set its password. Click on the user's name
to see its details, and click the "Generate password reset" link. Copy and paste the
url provided into your browser address bar, and set the user's password to "puppet".

Next, we need to give this new user permissions to run the Puppet Orchestrator.
Go to the **User Roles** section and create a new role with the name
"Orchestrators" and description "Run the Puppet Orchestrator."

Once this new role is created, click on its name to modify it. Select your
"Orchestrator" user from the drop-down menu and add it to the role.

Finally, go to the **Permissions** tab. Select "Orchestration" from the **Type**
drop-down menu, and "Use orchestration" from the **Permission** drop-down.
Click **Add permission** and commit your change. 

### Client token

Now that you have a user with correct permissions, you can generate an RBAC
access token to authenticate to the Orchestration service.

The `puppet access` tool helps manage authentication. Use the
`puppet access login` command to authenticate, and it will save a token.

  puppet access access login --service-url https://learning.puppetlabs.vm:4433/rbac-api

If you get an error message, double check that you entered the url correctly.

(Note that this expires!)

## Puppetized Applications

Now that you're configured to run the Puppet Orchestrator, you're ready to define
your application.

Just like the Puppet code you've worked with in previous quests, an application definition
should be packaged in a Puppet module. The application you'll be creating in this quest
will be based on the simple Linux Apache MySQL PHP (LAMP) stack pattern.

Before we dive into the code, let's take a moment to review the plan for this application.
What we do here will be a little bit simpler than the three tier application we discussed
above. We can save you a little typing, and still demonstrate the key features of the Application
Orchestrator.

(IMAGE)

In this case, we're going to have two separate components. One will define our MySQL database
configuration and will be applied to the `database.learning.puppetlabs.vm` node. The other
will define the configuration for an Apache web server and a simple PHP application and
be applied to the `webserver.learning.puppetlabs.vm` node.

To orchestrate this kind of deployment, we need to make sure two things happen. First,
we have to make sure our nodes are deployed in the correct order so we have our dependencies
set up before the dependant components in other nodes. Second, we need a method for passing
for passing information among our nodes.

Both of these are met through the use of environment resources. Unlike the node-specific
resources (like `user` or `file`) that tell puppet how to configure a single machine,
environment resources carry data and define relationships across multiple nodes in an
environment. We'll get more into the details of how this works as we implement our
application.

So if we think through the orchestration of our LAMP application, we should first ask what
the webserver needs to know about the database server.

1. **Host**: Our webserver needs to know the hostname of the database server.
1. **Database**: We need to know the name of the specific database to connect to.
1. **User**: If we want to connect to the database, we'll the name of a database user.
1. **Password**: We'll also need to know the password associated with that user.

This specifies exactly what our database server *produces* and what our webserver
*consumes*. If we pass this information to our webserver, it will have everything
it needs to connect to the database hosted on the database server.

To orchestrate a deployment of these two servers, we need an environment resource
that allows all of this information to be produced when we run puppet on our database
server, then consumed by our webserver. Like the node resources you're used to using
this resource will have a *type* that describes what it does. To pass information
between our webserver and database nodes, we'll create a new resource type called a
`sql`. Unlike a typical node resource, however, our `sql` resource won't directly make
any changes on a system. Its only job is to communicate its hash of parameters and values
between the application component that produces it and the one that consumes it.

So let's go ahead and create a new `sql` resource type. We'll have to take a quick detour
into writing Ruby to do this, but don't worry if you're not familiar with the language—because
we don't have to worry about defining any providers, the syntax will be very simple.

As before, the first step will be creating your module directory structure. Make sure
you're in your modules directory:

  cd /etc/puppetlabs/code/environments/production/modules

And create your directories:

  mkdir -p lamp/{manifests,lib/puppet/type}

(Note that we're burying our type in the `lib/puppet/type` directory. The `lib/puppet/`
directory is where you keep any extensions to the core puppet language that your
module provides. For example, in addition to types, you might also define new providers
or functions.)

Now let's go ahead and create our new `sql` resource type.

  vim lamp/lib/puppet/type/sql.rb

The new type is defined by a block of Ruby code, like so:

{% highlight ruby %}
Puppet::Type.newtype :sql, :is_capability => true do
  newparam :name, :is_namevar => true
  newparam :user
  newparam :password
  newparam :host
  newparam :database
end
{% endhighlight %}

See, not too bad! Note that it's the `is_capability => true` bit that lets this
resource live on the environment level, rather than being applied to a specific
node. Everything else should be reasonably self-explanatory. Again, we don't
actually have to *do* anything with this resource, so all we have to do is tell
it what we want to name our parameters.

Now that we have our new `sql` resource type, we can move on to the database
component that will produce it. This component will lives in our `lamp` module
and defines a configuration for a `MySQL` server, so we'll name it `lamp::myslq`.

  vim lamp/manifests/mysql.pp

It will look like this:

{% highlight puppet %}
define lamp::mysql
(
  $db_user,
  $db_password,
  $host = $::fqdn,
  $port = 3306,
  $database = $name,
) {

  class { '::mysql::server':
    service_provider => 'debian',
    override_options => {
      'mysqld' => { 'bind-address' => '0.0.0.0' }
    },
    grants => {
      "${db_user}@${host}/${database}.*" => {
        ensure     => present,
        options    => ['GRANT'],
        privileges => ['SELECT', 'INSERT', 'UPDATE', 'DELETE'],
        table      => "${database}.*",
        user       => "${db_user}@${host}",
      },
    }
  }

  mysql::db { $name:
    user =>     $db_user,
    password => $db_password,
  }

  class { '::mysql::bindings':
    php_enable       => true,
    php_package_name => 'php5-mysql',
  }

}
Lamp::Mysql produces Sql {
  user     => $db_user,
  password => $db_password,
  host     => $host,
  database => $name,
  port     => $port,
}
{% endhighlight %}

Next, create we'll create a webapp component to configure an Apache server
and a simple PHP application:

  vim lamp/manifests/webapp.pp

It will look like this:

{% highlight puppet %}
define lamp::webapp(
  $db_name,
  $db_user,
  $db_host,
  $db_password,
  ){
  class { 'apache':
    default_mods  => false,
    mpm_module    => 'prefork',
    default_vhost => false,
  }

  apache::vhost { $name:
    port    => '80',
    docroot => '/var/www/html',
  }

  package { 'php5-mysql':
    ensure => installed,
    notify => Service['httpd'],
  }

  include apache::mod::php

  $indexphp = @("EOT"/)
    <?php
    \$conn = mysql_connect($db_host, $db_user, $db_password);
    if (\$conn->connect_error) {
      echo "Connection to $db_host as $db_user failed";
    } else {
      echo "Connected successfully to $db_host as $db_user";
    }
    ?>
    | EOT

  file { "${docroot}/index.php":
    ensure  => file,
    content => $indexphp,
  }

}
Lamp::Webapp consumes Sql {
  db_name     => $name,
  db_user     => $user,
  db_host     => $host,
  db_password => $password,
}
{% endhighlight %}

Now that we have all of our components ready to go, we can define the application
itself. Because this is the main thing provided by the `lamp` module, it goes
in the `init.pp` manifest.

  vim lamp/manifests/init.pp

We've already done the bulk of the work work in our components, so this one will be pretty
simple. The syntax for an application is similar to that of a class or defined resource type.
The only difference is that we use the `application` keyword instead of `define` or
`class`.

{% highlight puppet %}
application lamp (
  $db_user,
  $db_password,
) {

  lamp::mysql { $name:
    db_user     => $db_user,
    db_password => $db_password,
    export      => $Sql[$name],
  }

  lamp::webapp { $name:
    consume => Sql[$name],
  }

}
{% endhighlight %}

The application has two parameters, `db_user` and `db_password`. The body of the
application declares the `lamp::mysql` and `lamp::webapp` components. We pass our
`db_user` and `db_password` parameters through to the `lamp::mysql` component. This is
also where we use the special `export` metaparameter to tell Puppet we want this component
to create a `sql` environment resource, which can then be consumed by the `lamp::webapp`
component. Remember that `Lamp::Mysql produces Sql` block we put after the component
definition?

{% highlight puppet %}
Lamp::Mysql produces Sql {
  user     => $db_user,
  password => $db_password,
  host     => $host,
  database => $name,
}
{% endhighlight %}

This tells Puppet how to map variables parameters in our `lamp::mysql` component into
a `sql` environment resource when we use this `export` metaparameter. Note that even
though we're only explicitly setting the `db_user` and `db_password` parameters in this
component declaration, the parameter defaults from the component will pass through as well. 

The matching `Lamp::Webapp consumes Sql` block in the `webapp.pp` manifest tells Puppet
how to map the parameters of the `sql` environment resource to our `lamp::webapp` component
when we include the `consume => Sql[$name]` metaparameter.

{% highlight puppet %}
Lamp::Webapp consumesSql {
  db_name     => $name,
  db_user     => $user,
  db_host     => $host,
  db_password => $password,
}
{% endhighlight %}

Now that your application is defined, the final step is to declare it in your `site.pp`
manifest.

  vim /etc/puppetlabs/code/environments/production/manifests/init.pp

Until now, declarations you've made in your `site.pp` manifest have been contained by
the `learning.puppetlabs.vm` node block. An application, however, is applied to your
environment independently of any classification defined in your node blocks or the PE
console node classifier. To express this distinction, we put our application declaration
in a special block called `site`.

{% highlight puppet %}
site { 
  lamp { 'app1':
    db_user     => 'roland',
    db_password => '12345',
    nodes       => {
      Node['database.learning.puppetlabs.vm'] => Lamp::Mysql['app1'],
      Node['webserver.learning.puppetlabs.vm'] => Lamp::Webapp['app1'],
    }
}
{% endhighlight %}

The syntax for declaring an application is similar to that of a class or resource.
The `db_user` and `db_password` parameters are set as usual. These will be passed through
to the `lamp::mysql` component, then, along with the `$host` and `$database' parameters
set to defaults in that component definition, exported to a `sql` resource and finally
consumed by the `lamp::webapp` instance.

The `nodes` parameter is where the real orchestration magic happens. This parameter takes
a hash of nodes paired with one or more components. In this case, we've assigned the
`Lamp::Mysql['app1']` component to `database.learning.puppetlabs.vm` and the
`Lamp::Webapp['app1'] component to `webserver.learning.puppetlabs.vm`. When the
Application Orchestrator runs, it uses the `exports` and `consumes` metaparameters
in your application definition (i.e. in your `lamp/manifests/init.pp` manifest)
to determine the correct order of Puppet runs across the nodes in the application.

Now that the application is declared in our `site.pp` manifest, we can use the
`puppet app` tool to view it.

  puppet app show

You should see a result like the following:

  Lamp['app1']
    Lamp::Mysql['app1'] => database.learning.puppetlabs.vm
        - produces Sql['app1']
    Lamp::Webapp['test'] => webserver.learning.puppetlabs.vm
        - consumes Sql['app1']

Now that the application is ready to go, you can test it by running the `puppet job`
command with the `--noop` flag:

  puppet job run Lamp['app1'] --noop 

After reviewing the similated changes, go ahead and trigger a real run:

  puppet job run Lamp['app1']

Now that your nodes are configured with your new application, let's take a moment
to check out the result. First, we can log in to the database server and have a
look our MySQL instance.

  docker exec -it database bash

Remember, no matter what OS you're on, you can use the `puppet resource` command
to check the status of a service. Let's see if the MySQL server is running:

  puppet resource service mysql

You should see that the service is running. If you like, you can also open the client
with the `mysql` command. When you're done, use `\q` to exit.

Now go ahead and disconnect from the database node.

  exit

Instead of logging in to our webserver node, let's just check if the server is running.
In the pre-configured docker setup for this quest, we mapped port 80 on the
`webserver.learning.puppetlabs.vm` container to port 10080 on `learning.puppetlabs.vm`.
In a web browser on your host machine, go to `http://<IP_ADDRESS>:10080` to see your
php website.

## Review

Review
