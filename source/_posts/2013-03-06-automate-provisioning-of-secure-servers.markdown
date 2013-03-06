---
layout: post
title: "Week 1: Automate provisioning of secure servers"
date: 2013-03-06 10:43
comments: true
categories:
---

With the [startup plan][1] in place, I spent the first week focusing on
provisioning servers with a cloud provider. Specifically, I intended to
leverage a solution such as Puppet or Chef to automate the setup and
configuration of a server within the [Digital Ocean][2] cloud hosting service
following a few security best practices. Before starting each week's
objective, I will be clarifying the business value and exit criteria to help
keep the end in mind.

<!-- more -->

Value and Exit Criteria
-----------------------
This first task will yield a decrease in operational cost and time to market.
Instead of manually creating new servers through a set of procedures for each
environment (development, test, production), the output of this effort will
provide the scaffolding to rebuild an environment within minutes while
maintaining quality through solid configuration management practices.

To complete the task, two criteria had to be met:

- Create and destroy server instances on the Digital Ocean cloud hosting
  service via the command line
- Implement a Chef [cookbook][3] to configure a server with a set of standard
  security practices

Why Digital Ocean and Chef?
---------------------------
While Amazon AWS is the big name in the cloud hosting space, I opted for
Digital Ocean for its prices, solid state drives, simplicity, and initimate
customer support. Given that my storage requirements will be small and that
subscriber growth should be reasonably predictable, a VPS offering better
price for performance made sense. There are a few things missing that I hope
to see soon: 1) an API for DNS mangement and 2) a HA load balancing solution.

I came to the decision to use Chef based on my experiences with Puppet at my
former company. Specifically, it became a challenge to leverage and baseline
modules developed by third-parties within our infrastrcture. While both products
are great (I would recommend both), I felt that Chef and its community were
pushing the envelope on reuse of infrastructure code with its data-driven
approach (attributes, data bags) and package management solutions
([librarian][4] and [berkshelf][5]). In the next section, I'll demonstrate this
reuse.

In addition to selecting Chef, I also made the decision to use Chef Solo vs
Chef Server. A large part of the decision had to do with keeping things simple.
With the open source project, [knife-solo][6], I was able to meet my exit
criteria while still benefiting from 90% of Chef's features.

Provisioning a Secure Server
----------------------------
Below is the step-by-step guide to provision a secure server within Digital
Ocean using Chef:

- **Install ruby gems.** To get started, I installed two gem packages that
   provide plugins to the Chef [knife][7] command-line tool. knife-solo
   supports the provisioning of Chef cookbooks to a single server from a
   workstation. [knife-digital_ocean][8] provides a wrapper around Digital
   Ocean's API to create and destroy server instances. I added these gems
   to my Gemfile in my [mac bootstrap][9] project, but you could install
   them directly:

{% codeblock %}
$ gem install knife-solo knife-digital_ocean
{% endcodeblock %}

   *Note*: I ran into [issue 177][10] when using knife-solo. To resolve, I
   used version 0.3.0.pre2 (instead of 0.2.0).

- **Create the Chef project repository.** With the gems installed, I created
   a new Chef repository (also known as a kitchen) that will contain configuration
   data and code to provision multiple servers comprising the product. The product
   has a tentative name, Marinara, so I will be using that throughout the rest of the
   article. I initialized the repository to use librarian so I could easily
   reference and use third-party cookbooks.

{% codeblock %}
$ knife solo init --librarian marinara-kitchen
{% endcodeblock %}

- **Initialize git repository and setup development branch.** It is a good practice
   to use version control with your Chef repository. With the decision to use Chef
   Solo, I am unable to use the [environments][11] capability of Chef Server which
   allows an operator to pin cookbook versions to a specific environment. However,
   using a git branching strategy like the one discussed at [nvie.com][12],
   environments can be represented as long-running git branches with varying
   cookbook versions defined in the Cheffile (more on that in a bit). An operator
   can simply checkout a git reference (tag or branch) to build an environment.

{% codeblock %}
$ cd marinara-kitchen
$ git init
$ git add .
$ git commit -m 'initial commit'
$ git checkout -b develop
{% endcodeblock %}

- **Setup knife configuration for Digital Ocean.** With the Chef repository created
   and under version control, I provided API credentials to communicate with the
   Digital Ocean service in my *knife.rb* file located at *marinara-kitchen/.chef/*.
   These credentials may be found within the *My Settings > API Access* section of
   the Digital Ocean dashboard.

{% codeblock knife.rb lang:ruby %}
knife[:digital_ocean_client_id] = 'xxxxxxxxxxxxxxxxxxxxx'
knife[:digital_ocean_api_key]   = 'yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy'
{% endcodeblock %}

- **Upload your ssh public key on Digital Ocean dashboard.** To create servers
   that do not require a root password, I uploaded my workstation's ssh public key
   within the *SSH Keys* section of the Digital Ocean dashboard. Once uploaded,
   I ran the following command (within the repository directory) to retrieve the ID
   of the ssh key to be used in the next step:

{% codeblock %}
$ knife digital_ocean sshkey list
{% endcodeblock %}

- **Create a new server and create a DNS A record.** With the knife configuration
   in place and the ssh key ID at hand, I was ready to create a new server (known
   as a droplet) on Digital Ocean via the command line. The command below creates
   a new Ubuntu 12.10 server with 512MB of RAM, 1 CPU, and a 20 GB SSD drive within
   the New York region using my public ssh key for access. Upon success, the
   command yields an IP address that I used to create a A record for
   *mycompany.com* in the Digital Ocean dashboard. Finally, I could shell into the
   new server.

{% codeblock %}
$ knife digital_ocean droplet create --server-name ops --image 25489 \
    --size 66 --location 1 --ssh-keys 4444
$ ssh root@mycompany.com
{% endcodeblock %}

- **Create an application cookbook.** With the first exit criteria met, I created
  a new application cookbook that would define recipes configuring the product's
  servers. By default, librarian manages third-party cookbooks in the *cookbooks*
  directory so I created the *marinara* cookbook within the *site-cookbooks*
  directory.

{% codeblock %}
$ knife cookbook create -o ./site-cookbooks marinara
{% endcodeblock %}

- **Install base packages.** I use vim, git, and tmux on a regular basis and wanted
  these packages installed on all servers. I defined a default attribute within the
  cookbook referencing an array of packages to install. Within the default recipe (
  all configuration will be placed here to simplify the article), I loop over this
  array (that may be redefined on a server by server basis) to install the packages.

{% codeblock site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# defines base packages
default.packages = %w(vim git tmux)
{% endcodeblock %}

{% codeblock site-cookbooks/marinara/recipes/default.rb lang:ruby %}
# provisions base packages for a node
node.packages.each do |pkg|
  package pkg
end
{% endcodeblock %}

- **Add administrator accounts and their associated dotfiles.** System
  administration should not be conducted using the root account. Each administrator
  should use their own account and password invoking all commands with sudo for
  audit purposes. To add administrator accounts, I leveraged the [chef-user][13]
  cookbook. This cookbook looks at the node attribute, *users*, which
  subsequently references users within the *users* data bag, to create new
  accounts. For administrators with dotfiles managed with [homesick][14], they
  would be deployed using the [chef-homesick][15] cookbook. These third-party
  cookbooks are referenced in two places: 1) the *Cheffile* at the root of the
  repository, and 2) the *metadata.rb* file at the root of the application cookbook.

  For each administrator account, the ssh public key is provisioned, the
  hashed password is set (using mkpasswd -m sha-512), and it is added to the sudo
  admin group.

{% codeblock ./data_bags/users/shawn.json lang:javascript %}
{
    "id": "shawn",
    "password": "xxx",
    "groups": [
        "sudo"
    ],
    "ssh_keys": [
        "public key here"
    ],
    "homesick_castles": [
        {
            "name": "dotfiles",
            "source": "git://github.com/smdahlen/dotfiles.git"
        }
    ]
}
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/metadata.rb lang:ruby %}
depends 'user', '~> 0.3.0'
depends 'homesick', '~> 0.3.2'
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# defines user accounts and overriden user cookbook attributes
default.users = ['shawn']
default.user.ssh_keygen = false
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/recipes/default.rb lang:ruby %}
# provisions user accounts and their dotfiles defined in a data bag
include_recipe 'user::data_bag'
include_recipe 'homesick::data_bag'
{% endcodeblock %}

- **Restrict ssh access.** With the adminstrator accounts provisioned, it was time
  to tighten access to ssh by not permitting root login and disabling password
  authentication. This was accomplished by leveraging the [openssh][16] cookbook
  and defining the appropriate default attributes.

{% codeblock ./site-cookbooks/marinara/metadata.rb lang:ruby %}
depends 'openssh', '~> 1.1.4'
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# defines sshd configuration that differ from /etc/ssh/sshd_config defaults
default.openssh.server.permit_root_login = 'no'
default.openssh.server.password_authentication = 'no'
default.openssh.server.allow_groups = 'sudo'
default.openssh.server.login_grace_time = '30'
default.openssh.server.use_p_a_m = 'no'
default.openssh.server.print_motd = 'no'
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/recipes/default.rb lang:ruby %}
# provisions ssh configured for no password auth and root login
include_recipe 'openssh'
{% endcodeblock %}

- **Setup notifications for package updates.** Another security best practice is
   to ensure that security updates are applied in a timely manner. While I could
   have setup automatic updates, this would inevitably lead to issues in production.
   It is best to try updates within a test environment first so I opted to receive
   daily email notifications instead. I setup *ssmtp* to relay mail through a
   smarthost (in my case, [Mailgun][17]). My recipe loads smarthost configuration
   from a data bag and passes it to a template to create the ssmtp.conf file. The
   package notifications are delivered using the [apticron][18] tool.

{% codeblock ./data_bags/smarthosts/mailgun.json lang:javascript %}
{
    "id": "mailgun",
    "host": "smtp.mailgun.org",
    "port": "587",
    "username": "postmaster@mycompany.com",
    "password": "xxx"
}
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# defines smtp relay configuration
default.smtp.smarthost = 'mailgun'
default.smtp.rewrite_domain = 'mycompany.com'

# defines apitcron configuration
default.apticron.email = 'ops@mycompany.com'
default.apticron.diff_only = false
default.apticron.notify_no_updates = false
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/recipes/default.rb lang:ruby %}
# provisions ssmtp configured for a smarthost defined in a data bag
# TODO: use encrypted data bag for smarthost credentials
smarthost = data_bag_item(:smarthosts, node.smtp.smarthost)
unless smarthost.nil?
  package 'ssmtp'
  template '/etc/ssmtp/ssmtp.conf' do
    source 'ssmtp.conf.erb'
    owner 'root'
    group 'root'
    mode '0644'
    variables(
      smarthost: smarthost,
      domain: node.smtp.rewrite_domain
    )
  end
end

# provisions apticron for package update notifications
package 'apticron'
template '/etc/apticron/apticron.conf' do
  source 'apticron.conf.erb'
  owner 'root'
  group 'root'
  mode '0644'
  variables(
    email: node.apticron.email,
    diff_only: node.apticron.diff_only,
    notify_no_updates: node.apticron.notify_no_updates
  )
end
{% endcodeblock %}

- **Setup a restrictive firewall.** The final step to configuring the server was to
  setup a restrictive firewall denying all inbound traffic except ssh. I set the
  INPUT chain policy for iptables to DENY and created a custom chain for my
  server rules. The rules were defined using a Chef [LWRP][19] from the
  [simple-iptables][20] cookbook.

{% codeblock ./site-cookbooks/marinara/metadata.rb lang:ruby %}
depends 'simple_iptables', '~> 0.2.4'
{% endcodeblock %}

{% codeblock ./site-cookbooks/marinara/recipes/default.rb lang:ruby %}
# provisions iptables firewall rules restricting all ports except ssh
include_recipe 'simple_iptables'
simple_iptables_policy 'INPUT' do
  policy 'DROP'
end
simple_iptables_rule 'system' do
  rule '-i lo'
  jump 'ACCEPT'
end
simple_iptables_rule 'system' do
  rule '-m conntrack --ctstate ESTABLISHED,RELATED'
  jump 'ACCEPT'
end
simple_iptables_rule 'system' do
  rule '-p icmp'
  jump 'ACCEPT'
end
simple_iptables_rule 'system' do
  rule '-p tcp --dport ssh'
  jump 'ACCEPT'
end
{% endcodeblock %}

- **Bootstrap the server.** With the basic application cookbook complete, all that
  was left was to tell Chef to apply the default cookbook recipe to the
  *mycompany.com* server. This was accomplished by defining a run list for the node:

{% codeblock ./nodes/mycompany.com.json %}
{
    "run_list": [
        "recipe[marinara]"
    ]
}
{% endcodeblock %}

  Finally, I could bootstrap the server with the configuration code defined above
  with the following command:

{% codeblock %}
$ knife solo bootstrap root@mycompany.com
{% endcodeblock %}

  With that, the second exit criteria had been met.

[1]: http://shawn.dahlen.me/blog/2013/03/05/prepare-business-operations/
[2]: https://www.digitalocean.com
[3]: http://docs.opscode.com/essentials_cookbooks.html
[4]: https://github.com/applicationsonline/librarian
[5]: http://berkshelf.com
[6]: http://matschaffer.github.com/knife-solo/
[7]: http://docs.opscode.com/knife.html
[8]: https://github.com/rmoriz/knife-digital_ocean
[9]: https://github.com/smdahlen/mac-bootstrap
[10]: https://github.com/matschaffer/knife-solo/issues/177
[11]: http://docs.opscode.com/essentials_environments.html
[12]: http://nvie.com/posts/a-successful-git-branching-model/
[13]: https://github.com/fnichol/chef-user
[14]: https://github.com/technicalpickles/homesick
[15]: https://github.com/fnichol/chef-homesick
[16]: https://github.com/opscode-cookbooks/openssh
[17]: http://www.mailgun.com/
[18]: http://www.debian-administration.org/articles/491
[19]: http://docs.opscode.com/lwrp.html
[20]: https://github.com/dcrosta/cookbook-simple-iptables
