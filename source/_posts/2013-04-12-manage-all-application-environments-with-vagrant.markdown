---
layout: post
title: "Manage all application environments with Vagrant"
date: 2013-04-12 13:53
comments: true
keywords: vagrant,chef,plugin,digital ocean,production,virtualbox
description: What if Vagrant could be used to manage all application environments and not just development?
---

What if I could use [Vagrant][1] to configure and create reproducible
environments for development, test, and production? This is the question I
thought to myself while working on the [planned activity][2] to setup a
MongoDb cluster. After spending time testing the cluster with VirtualBox and
Vagrant and then switching to [knife-solo][3] to bootstrap droplets on
Digital Ocean, I opted to dig further into my question.

<!-- more -->

With the release of Vagrant 1.1 in mid-March, it was now possible to manage
virtual machines backed by providers other than VirtualBox. HashiCorp, the
creator of Vagrant, provided a [compelling preview][4] showcasing Vagrant
managing Amazon EC2 servers. The timing was right to investigate the
possibility of using Vagrant as *the tool* to define my environment and
subsequently set it up on either VirtualBox or Digital Ocean. As such, I
redirected my remaining few weeks for the first sprint to tackle this challenge.

Value & Exit Criteria
---------------------
Chef had proven extremely useful for defining configuration for
individual servers. Vagrant would compliment Chef by providing a common,
consistent interface for managing an environment of servers. This would help
further minimize system regressions and reduce the operational cost for managing
multiple environments.

To claim success on the task, I defined the end state. As a developer and
administrator, I wanted to run the following commands to stand up a
six-server environment with VirtualBox or Digital Ocean respectively:

    $ vagrant up --provider=[virtualbox|digital_ocean] --no-provision
    $ vagrant provision
    $ curl http://<proxy>/api/tasks

The environment had to be defined with a single `Vagrantfile` creating a proxy
server, two Node.js application servers, and a MongoDb replica set. After
provisioning with a single Chef application cookbook, the system had to be
fully operational accepting requests.

Vagrant Plugins
---------------
In my previous job, we had used Vagrant to create reproducible development
environments for the development team. Beyond creating a basic `Vagrantfile`
to spin up a machine provisioned with Puppet or Chef, my knowledge of Vagrant
was limited. As I dove into the internals of Vagrant to meet my objective, I
was pleasantly surprised to find an extensible plugin framework. The framework
supports new configuration, commands, virtual machine providers, and
action hooks. In fact, most of Vagrant's built-in features are implemented
as plugins.

Operational use cases for creating and managing an environment
could be implemented as a set of plugins. I looked to reuse existing ones where
possible and created new ones when needed. The four plugins that helped
achieve my objective are discussed below.

Digital Ocean Provider Plugin
-----------------------------
To create, rebuild, and destroy Digital Ocean droplets, I needed a Vagrant
[provider plugin][5]. Luckily, [John Bender][6] had already established a GitHub
project that delivered the basic functionality. Unfortunately, it was missing
support for SSH keys (it used the insecure Vagrant key) and relied on a root
account for subsequent provisioning. John was gracious enough to support my
modifications ultimately transitioning the [project][7] over to me. Adding
features to this plugin was a great exercise in understanding the fundamentals
of Vagrant and I highly recommend anyone interested to browse the source code.

As of today, the plugin provides the following features:

- create and destroy droplet instances
- rebuild a droplet instance while retaining its assigned IP address (custom
  command)
- power on and off a droplet instance
- setup a SSH public key for authentication
- create a new user account during droplet creation (allowing me to disable
  the root account during provisioning)
- provision a droplet with the shell or Chef

While this plugin met 80% of my objective, creating new droplet instances
introduced a problem. An IP address is not assigned to a droplet until after
it is created, however, my Chef recipes required knowledge of it upfront.

Host Manager Plugin
-------------------
To solve this dilemma, I opted to reference servers in my Chef recipes using
host names and leverage a Vagrant plugin to synchronize a `/etc/hosts` file across
the environment to resolve the names. I created a Vagrant 1.1 compliant
plugin, [vagrant-hostmanager][8], that hooked into the `up` action for a new
server. After server creation, a line containing the server's new
IP address and host name is added to the `/etc/hosts` file of each active
server within the Vagrant environment. Once the `vagrant up` command completes,
all servers defined within the `Vagrantfile` can resolve to one another using
host names.

Due to this approach, I had to disable provisioning when calling `vagrant up`.
That is why my end state included the `--no-provision` switch and a second
command to provision the servers.

MongoDb Plugin
--------------
Now that servers within my Digital Ocean environment could resolve to one
another, I could move forward with configuring a replica set. Before I had
started my investigation into Vagrant, I had attempted to use the [mongodb][9]
cookbook. Unfortunately, the cookbook required a Chef server to search for
replica set members and I had [opted for Chef solo][10] a few weeks earlier.
Additionally, it felt awkward that a recipe provisioning a single MongoDb
replica set member would also attempt to initiate a replica set if all
members were available.

A Vagrant plugin seemed to be a better fit -- it had access to the
`Vagrantfile` containing configuration for all servers in the environment and
therefore could act on the environment itself. I created a plugin,
[vagrant-mongodb][11], that provided an adminstrator with new configuration
options in the `Vagrantfile` to describe a replica set. Here is a brief example:

```ruby
config.vm.mongodb.replset :rs0 do |rs|
  rs.member :db0, :priority => 0, arbiter => true
  rs.member :db1, :priority => 1
  rs.member :db2, :priority => 2
end
```

With a replica set defined within a `Vagrantfile`, the plugin will check
if all members are available, and if so, initiate it. By default,
this happens automatically after executing `vagrant provision`, although this
behavior may be disabled and a custom command executed instead.

Berkshelf Plugin
----------------
With the three plugins above, I could now meet my objective. However, I took
the time to clean up my use of Chef from earlier weeks. Since I was replacing
aspects of `knife-solo` with Vagrant, I had the opportunity to use
[Berkshelf][12] for cookbook dependency management. I collapsed my
`marinara-kitchen` project into a single application cookbook called
`marinara-cookbook` containing the recipes I wrote earlier. I defined my
cookbook's dependencies in the `metadata.rb` file:

{% codeblock metadata.rb lang:ruby %}
name             'marinara'
maintainer       'Whimsical Bits, Ltd.'
maintainer_email 'shawn@dahlen.me'
license          'All rights reserved'
description      'Provisions the Marinara application'
long_description IO.read(File.join(File.dirname(__FILE__), 'README.md'))
version          '0.1.0'

depends 'user', '~> 0.3.0'
depends 'homesick', '~> 0.3.2'
depends 'openssh', '~> 1.1.4'
depends 'simple_iptables', '~> 0.2.4'
depends 'nodejs', '~> 1.1.1'
depends 'npm', '~> 0.1.1'
depends 'haproxy', '~> 1.2.0'
depends 'apt', '~> 1.9.0'
depends 'sudo', '~> 2.0.4'
{% endcodeblock %}

With the Berkshelf Vagrant [plugin][13] installed, all cookbook dependencies
are automatically available to each server during the provisioning process.

Vagrantfile
-----------
With plugins to manage Digital Ocean droplets, synchronize a `/etc/hosts` file
for server resolution, and initiate MongoDb replica sets, I was ready to
define my application's environment in a single `Vagrantfile`. Below is my
file that works with both VirtualBox and Digital Ocean (with a few manual
tweaks required until Vagrant 1.2 is released):

{% codeblock Vagrantfile lang:ruby %}
# define the number of application and database servers
app_servers = 2.times.map { |i| "app#{i}" }
db_servers = 3.times.map { |i| "db#{i}" }

Vagrant.configure('2') do |config|
  # define the user account and ssh key to authenticate with
  # NOTE: these can be replaced with environment variables to allow a
  # continuous integration server to also create and provision environments
  config.ssh.username = 'smdahlen'
  config.ssh.private_key_path = '~/.ssh/id_rsa'
  config.ssh.forward_agent = true

  # disable snycing of the root project folder
  config.vm.synced_folder '/vagrant', '.', :id => 'vagrant-root', :disabled => true

  # TODO move into provider override when Vagrant 1.2 is released
  config.hostmanager.ignore_private_ip = true

  # define the base box for virtual box
  # TODO move into provider override when Vagrant 1.2 is released
  config.vm.box = 'precise64-chef11.2'
  config.vm.box_url = 'https://opscode-vm.s3.amazonaws.com/vagrant/opscode_ubuntu-12.04_chef-11.2.0.box'

  # define virtual box provider defaults
  config.vm.provider :virtualbox do |vb|
    vb.customize [
      'setextradata',
      :id,
      'VBoxInternal2/SharedFoldersEnableSymlinksCreate/v-root',
      '1'
    ]
    vb.customize ['modifyvm', :id, '--memory', '512']
    vb.customize ['modifyvm', :id, '--cpus', 2]
  end

  # define digital ocean provider defaults
  config.vm.provider :digital_ocean do |ocean|
    ocean.client_id = ENV['DO_CLIENT_ID']
    ocean.api_key = ENV['DO_API_KEY']
    ocean.ssh_key_name = 'MacBook Pro'
  end

  # define the proxy server
  config.vm.define :proxy do |proxy|
    proxy.vm.hostname = 'proxy'
    proxy.vm.network :private_network, ip: '10.0.5.2'
    proxy.vm.provision :chef_solo do |chef|
      chef.data_bags_path = 'data_bags'
      chef.add_recipe 'marinara::default'
      chef.add_recipe 'marinara::proxy'
      chef.json = {
        marinara: {
          application: {
            servers: app_servers
          }
        }
      }
    end
  end

  # define the application servers
  app_servers.each_index do |index|
    ip = "10.0.5.#{index + 3}"

    config.vm.define app_servers[index] do |app|
      app.vm.hostname = app_servers[index]
      app.vm.network :private_network, ip: ip
      app.vm.provision :chef_solo do |chef|
        chef.data_bags_path = 'data_bags'
        chef.add_recipe 'marinara::default'
        chef.add_recipe 'marinara::application'
        chef.json = {
          marinara: {
            application: {
              reference: 'develop'
            },
            proxy: {
              server: 'proxy'
            },
            database: {
              servers: db_servers
            }
          }
        }
      end
    end
  end

  # define the database servers
  db_servers.each_index do |index|
    config.mongodb.replset :rs0 do |rs|
      rs.ignore_private_ip = true
      rs.member db_servers[index]
    end

    ip = "10.0.5.#{index + app_servers.size + 3}"

    config.vm.define db_servers[index] do |db|
      db.vm.hostname = db_servers[index]
      db.vm.network :private_network, :ip => ip
      db.vm.provision :chef_solo do |chef|
        chef.data_bags_path = 'data_bags'
        chef.add_recipe 'marinara::default'
        chef.add_recipe 'marinara::database'
        chef.json = {
          marinara: {
            application: {
              servers: app_servers
            },
            database: {
              replset: 'rs0',
              servers: db_servers
            }
          }
        }
      end
    end
  end
end
{% endcodeblock %}

You will note the declaration of private networks and static IP addresses
in the file. This enables the machines to communicate with one another
when running locally using VirtualBox. Currently, I have to ignore the
private IP addresses when using Digital Ocean.

Vagrant 1.2
-----------
When Vagrant 1.2 is released, a user will be able to configure
override attributes for a specific provider. In the file above, I can move
the `ignore_private_ip` configuration into the Digital Ocean provider without
manually changing values.

More importantly, Vagrant 1.2 introduces the possibility of executing actions
in parallel for a multi-server environment. This will cut down my current time
to build a new six-server environment from 15 minutes to ~3 minutes.

What's Next
-----------
Overall, I'm quite pleased with the outcome. With this sprint complete, I
can now create and manage a multi-server environment for development, test, and
production with the combination of Vagrant and Chef. Over the next six weeks
I'll be shifting my focus from infrastructure to product management
defining my product's scope and drafting a user experience in preparation
for development. Stay tuned for more.

[1]: http://www.vagrantup.com/
[2]: http://shawn.dahlen.me/blog/2013/03/05/prepare-business-operations/
[3]: https://github.com/matschaffer/knife-solo
[4]: http://www.hashicorp.com/blog/preview-vagrant-aws.html
[5]: http://docs.vagrantup.com/v2/plugins/providers.html
[6]: https://twitter.com/johnbender
[7]: https://github.com/smdahlen/vagrant-digitalocean
[8]: https://github.com/smdahlen/vagrant-hostmanager
[9]: https://github.com/edelight/chef-mongodb
[10]: http://shawn.dahlen.me/blog/2013/03/06/automate-provisioning-of-secure-servers/
[11]: https://github.com/smdahlen/vagrant-mongodb
[12]: http://berkshelf.com/
[13]: https://github.com/riotgames/berkshelf-vagrant
