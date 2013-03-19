---
layout: post
title: "Week 2: Setup node.js servers within a load-balanced configuration"
date: 2013-03-18 13:21
comments: true
keywords: nodejs,haproxy,vagrant,chef,requirejs,backbone,express,startup
description: A step-by-step guide to setup a Node.js project, provision it with Chef, and deploy it within a load-balanced configuration using HAProxy.
---

With an excellent [foundation][1] in place to automate provisioning of secure
servers with Chef, I spent week two establishing an application project
structure using the [Node.js][2] platform. Additionally, I setup a load
balancer configuration with [HAProxy][3] and tested it using a local
multi-machine [Vagrant][4] environment. Read on to find out details about
this setup.

<!-- more -->

Value & Exit Criteria
---------------------
As I mentioned in my [previous post][1], I start each week defining the
business value for the task at hand coupled with concise exit criteria to
know when I'm done. This week's task will ensure the conversion rate for
subscribers is not impacted as the product usage grows. I consider the
first impression of the product's experience extremely important and I don't
want to see that disrupted as I try to scale a system in production.

To complete the task, four critera had to be met:

- Setup a project structure for Node.js supporting a rapid development workflow
  and optimized production deployment.
- Implement a Chef recipe to provision the Node.js application.
- Implement a Chef recipe to provision HAProxy configured to load balance
  multiple Node.js application servers.
- Test the Chef recipes using a local multi-machine Vagrant environment.

Setup a Node.js Project
-----------------------
Given my experience with Node.js from my previous job at Lockheed Martin, I
selected it as the platform to build my product. I intend to implement a
thin server architecture supporting a [single-page application][5].
Essentially, all the user interface logic will reside on the client using
JavaScript libraries such as [jQuery][6], [require.js][7], and
[Backbone][8]. The user interface will query an API layer built on the
[Express][9] web application framework for Node.js.

Below is a step-by-step guide to setup a basic Node.js project using these
libraries. It assumes that Node.js and npm are installed.

- **Setup Express.** I began by creating a new directory, *marinara*, and
  initialized it as a git repository. Next, I created the *package.json* file
  which defines metadata for the project including its dependencies. Within that
  file I specified the Express version I required and then installed it
  using [npm][10].

{% codeblock package.json lang:javascript %}
{
    "name": "marinara",
    "version": "0.1.0",
    "engines": {
        "node": "0.10.x"
    },
    "dependencies": {
        "express": "~3.1.0",
        "log": "~1.3.1",
        "underscore": "~1.4.4",
        "consolidate": "~0.8.0"
    },
    "devDependencies": {
    }
}
{% endcodeblock %}

{% codeblock %}
$ npm install
{% endcodeblock %}

  With Express installed, I created a *server.js* file at the root of the
  project directory and implemented a basic application with a simple logging
  and configuration strategy. Specifically, I defined configuration through
  environment variables (with suitable defaults) that will later be set with an
  [Upstart][11] script. I leveraged the [log.js][12] module to write messages
  using the log levels specified by syslog that will later be consolidated with
  an [rsyslog][13] server.

{% codeblock server.js lang:javascript %}
'use strict';

var express = require('express'),
    Log = require('log'),
    api = require('./lib/api'),
    pkg = require('./package.json'),
    app = express();

// defines app settings with default values
app.set('log level', process.env.MARINARA_LOG_LEVEL || Log.DEBUG);
app.set('session secret', process.env.MARINARA_SESSION_SECRET || 'secret');
app.set('session age', process.env.MARINARA_SESSION_AGE || 3600);
app.set('port', process.env.MARINARA_PORT || 8000);

// configures default logger available for middleware and requests
app.use(function (req, res, next) {
    req.log = new Log(app.get('log level'));
    next();
});

// logs all requests if log level is INFO or higher using log module format
if (app.get('log level') >= Log.INFO) {
    var format = '[:date] INFO :remote-addr - :method :url ' +
                 ':status :res[content-length] - :response-time ms';
    express.logger.token('date', function () { return new Date(); });
    app.use(express.logger(format));
}

// TODO: setup cluster support
app.listen(app.get('port'));
{% endcodeblock %}

- **Serve static assets and api requests.** With the basic application
  structure complete, I implmented support to serve static assets both in
  development and production. During development, static assets are served
  from the *app/* directory using require.js to keep code modular. For
  production, assets are optimized (concatnated and minified) and placed in
  the *public/* directory. The single html page, *index.html*, is an
  [underscore.js micro-template][14] that replaces script and stylesheet
  references depending on the mode.

{% codeblock server.js lang:javascript %}
// configures underscore view engine
// TODO: set caching for production
app.engine('html', require('consolidate').underscore);
app.set('view engine', 'html');
app.set('views', __dirname + '/templates');

// enables compression for all requests
app.use(express.compress());

// serves index and static assets from app/ directory in development
app.configure('development', function () {
    app.get('/', function (req, res) {
        res.render('index', {
            jsFile: 'public/components/requirejs/require.js',
            cssFile: 'public/styles/main.css'
        });
    });
    app.use('/public', express.static(__dirname + '/app'));
});

// serves index and static assets from optimized public/ directory in prod
// TODO: replace staticCache middleware with varnish
app.configure('production', function () {
    var oneYear = 60*60*24*365,
        baseFile = 'public/' + pkg.name + '-' + pkg.version;

    app.get('/', function (req, res) {
        res.render('index', {
            jsFile: baseFile + '.min.js',
            cssFile: baseFile + '.min.css'
        });
    });
    app.use('/public', express.staticCache());
    app.use('/public', express.static(__dirname + '/public', { maxAge: oneYear }));
});
{% endcodeblock %}

{% codeblock templates/index.html lang:html %}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Marinara</title>
    <link type="text/css" rel="stylesheet" href="<%= cssFile %>">
</head>
<body>
    <script data-main="public/scripts/main" src="<%= jsFile %>"></script>
</body>
</html>
{% endcodeblock %}

  While I have not flushed out the api for the product yet, I went ahead and
  stubbed out a basic implementation. I created a sub-application in
  *lib/api.js* where I configured middleware to handle secure cookie sessions,
  cross-site request forgery, and body parsing. This sub-application is
  exported and mounted at the */api* path on the main Express application in
  *server.js*.

{% codeblock lib/api.js lang:javascript %}
'use strict';

var express = require('express'),
    app = express();

module.exports = function (options) {
    app.use(express.cookieParser());
    app.use(express.cookieSession({
        secret: options.sessionSecret,
        cookie: { maxAge: options.sessionAge }
    }));
    app.use(express.bodyParser());
    app.use(express.csrf());
    app.use(app.router);

    app.get('/hello', function (req, res) {
        req.log.debug('serving /api/hello request');
        res.send('Hello Shawn');
    });

    return app;
};
{% endcodeblock %}

{% codeblock server.js lang:javascript %}
// mounts and configures rest api
app.use('/api', api({
    sessionSecret: app.get('session secret'),
    sessionAge: app.get('session age')
}));
{% endcodeblock %}

- **Setup Bower and install client dependencies.** With the server ready to
  handle requests, I shifted to the client-side by defining libraries in
  *component.json* that [Bower][15], a browser package manager, would install.
  (I will discuss the installation of Bower shortly). I configured Bower
  (in the *.bowerrc* file) to install libraries to *app/components* so they
  could be served by Express during development.

{% codeblock component.json lang:javascript %}
{
    "name": "marinara",
    "version": "0.1.0",
    "dependencies": {
        "requirejs": "~2.1.5",
        "jquery": "~1.9.1",
        "backbone": "~0.9.10",
        "underscore": "~1.4.4",
        "almond": "~0.2.5"
    }
}
{% endcodeblock %}

{% codeblock .bowerrc lang:javascript %}
{
    "directory": "app/components"
}
{% endcodeblock %}

{% codeblock %}
$ bower install
{% endcodeblock %}

- **Setup require.js configuration and main entry point.** As I mentioned
  earlier, require.js supports modular development of both css and javascript.
  I went ahead and defined a main entry point for require.js, *main.js*,
  which includes configuration to use the libraries installed by Bower. I also
  stubbed out a simple Backbone view to test asset optimization later on.

{% codeblock app/scripts/main.js lang:javascript %}
requirejs.config({
    paths: {
        jquery: '../components/jquery/jquery',
        backbone: '../components/backbone/backbone',
        underscore: '../components/underscore/underscore'
    },
    shim: {
        backbone: {
            deps: ['underscore', 'jquery'],
            exports: 'Backbone'
        },
        underscore: {
            exports: '_'
        }
    }
});

require(['jquery', 'marinara'], function ($, marinara) {
    'use strict';

    var view = new marinara.SampleView();
    $('body').html(view.render().el);
});
{% endcodeblock %}

{% codeblock app/scripts/marinara.js lang:javascript %}
define(['jquery', 'backbone', 'underscore'], function ($, Backbone, _) {
    'use strict';

    var exports = {};

    var SampleView = exports.SampleView = Backbone.View.extend({
        id: 'sample',
        template: _.template('<h1><%= msg %></h1>'),
        events: {
            'click': function () { alert('Hello Shawn.'); }
        },
        render: function () {
            this.$el.html(this.template({ msg: 'Click me please.' }));
            return this;
        }
    });

    return exports;
});
{% endcodeblock %}

- **Setup Grunt.js build system.** To test changes during development, I setup
  an automated watch and reload workflow using [Grunt][16]. When files change
  in the *app/* directory, I automatically reload the browser window with the
  [LiveReload][17] Chrome extension. When javascript files change in the */* or
  *lib/* directories, I restart the Node.js server using Upstart. Additionally,
  I also lint both css and javascript files when changes occur. To kickoff this
  workflow, I run *grunt* in the root directory.

  I also used Grunt to optimize static assets. The *grunt-contrib-requirejs*
  plugin uses require.js's [optimizer][18] to concatnate and minify javascript.
  It also concatnate's css by scanning for *@import* statements. I paired this
  with the *grunt-contrib-cssmin* plugin to minify css assets. To optimize
  assets and move them into the *public/* directory to be served by Express,
  I run:

{% codeblock %}
$ grunt optimize
{% endcodeblock %}

  Check out this [gist][19] for the *Gruntfile.js* that supports this setup.
  With the build system complete, the first exit criteria was met.

Implement Chef Application Recipe
---------------------------------
With the Node.js project structure complete, I shifted back to the
*marinara-kitchen* repository I worked on in the [last post][1]. I created
a new recipe, *application.rb*, that would provision Node.js and related
dependencies and deploy the *marinara* git repository. Below is the
step-by-step guide to create the recipe:

- **Provision Node.js and npm packages.** I leveraged the [nodejs][20] cookbook
  to install Node.js and npm by source defining the default versions in the
  *default.rb* attributes file. Additionally, I defined npm packages to be
  installed globally on the server (Grunt, Bower) using the LWRP provided
  by the [npm][21] cookbook.

{% codeblock site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# includes nodejs default attributes first to override them
include_attribute 'nodejs'

# defines node.js and npm version configuration
default.nodejs.version = '0.10.0'
default.nodejs.npm = '1.2.14'

# defines npm packages to install globally
default.marinara.application.npm_packages = {
  'grunt-cli' => '0.1.6',
  'bower'     => '0.8.5'
}
{% endcodeblock %}

{% codeblock site-cookbooks/marinara/recipes/application.rb lang:ruby %}
# provisions node.js and npm
include_recipe 'nodejs::install_from_source'
include_recipe 'nodejs::npm'

# provisions global npm packages
node.marinara.application.npm_packages.each_pair do |pkg, ver|
  npm_package pkg do
    version ver
  end
end
{% endcodeblock %}

- **Provision Upstart script.** Since I am using Ubuntu, I implemented an
  Upstart script to manage the Node.js application as a service. As I mentioned
  earlier, I configure the application using environment variables defined in
  the Upstart script injected in by a Chef template. The template also
  checks whether the application is being provisioned in production mode, and
  if so, includes statements to start the service on startup and run under
  an application-specific user account.

{% codeblock site-cookbooks/marinara/templates/default/marinara.conf.erb lang:ruby %}
description 'marinara upstart script'
author 'Shawn Dahlen <shawn@dahlen.me>'

<% if not node.marinara.application.development -%>
start on [2345]
setuid <%= node.marinara.application.user %>
setgid <%= node.marinara.application.user %>
env NODE_ENV=production
<% end -%>

env MARINARA_LOG_LEVEL=<%= node.marinara.application.log_level %>
env MARINARA_PORT=<%= node.marinara.application.port %>

stop on [06]

respawn

exec /usr/local/bin/node <%= node.marinara.application.deploy_path %>/server.js > <%= node.marinara.application.log_path %>/marinara.log 2>&1
{% endcodeblock %}

{% codeblock site-cookbooks/marinara/recipes/application.rb lang:ruby %}
# provisions upstart script
template '/etc/init/marinara.conf' do
  source 'marinara.conf.erb'
  mode 0440
end
{% endcodeblock %}

- **Deploy application.** If the application will be provisioned in
  production mode (as opposed to development mode where Vagrant mounts the
  project directory), the recipe provisions the application user, creates
  the deployment and log directories, copies over a ssh deploy key, clones
  the *marinara* git repository, install dependencies, and optimizes assets. A
  number of default attributes drive the deployment -- the most important
  of which is the *reference* attribute defining the git tag or branch to
  deploy.

{% codeblock site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# defines marinara application deployment configuration
default.marinara.application.development = false
default.marinara.application.user = 'marinara'
default.marinara.application.repository = 'git@github.com:smdahlen/marinara.git'
default.marinara.application.reference = 'master'
default.marinara.application.deploy_path = '/opt/marinara'
default.marinara.application.log_path = '/var/log/marinara'
default.marinara.application.log_level = 6
default.marinara.application.port = 8000
default.marinara.application.servers = {
  'app0' => '127.0.0.1'
}
{% endcodeblock %}

{% codeblock site-cookbooks/marinara/recipes/application.rb lang:ruby %}
if not node.marinara.application.development
  # stops marinara service if running
  service 'marinara' do
    action :stop
    provider Chef::Provider::Service::Upstart
  end

  # provisions system user to run application
  user node.marinara.application.user do
    system true
    shell '/bin/false'
    home '/home/marinara'
    supports manage_home: true
  end

  # provisions deploy key and ssh wrapper
  cookbook_file '/tmp/deploy_id_rsa' do
    source 'deploy_id_rsa'
    mode 0400
    owner node.marinara.application.user
  end
  cookbook_file '/tmp/deploy_ssh_wrapper' do
    source 'deploy_ssh_wrapper'
    mode 0500
    owner node.marinara.application.user
  end

  # provisions deploy directory with app user permissions
  directory node.marinara.application.deploy_path  do
    owner node.marinara.application.user
    group node.marinara.application.user
  end

  # provisions log directory with app user permissions
  directory node.marinara.application.log_path  do
    owner node.marinara.application.user
    group node.marinara.application.user
  end

  # provisions git repository
  git node.marinara.application.deploy_path  do
    repository node.marinara.application.repository
    reference node.marinara.application.reference
    user node.marinara.application.user
    group node.marinara.application.user
    ssh_wrapper '/tmp/deploy_ssh_wrapper'
  end

  # provisions npm application dependencies
  execute 'npm install' do
    cwd node.marinara.application.deploy_path
    command '/usr/local/bin/npm install'
    user node.marinara.application.user
    group node.marinara.application.user
    env 'HOME' => "/home/#{node.marinara.application.user}"
  end

  # provisions bower application dependencies
  execute 'bower install' do
    cwd node.marinara.application.deploy_path
    command 'bower install'
    user node.marinara.application.user
    group node.marinara.application.user
    env 'HOME' => "/home/#{node.marinara.application.user}"
  end

  # optimizes browser assets
  execute 'grunt optimize' do
    cwd node.marinara.application.deploy_path
    command 'grunt optimize'
    user node.marinara.application.user
    group node.marinara.application.user
  end

  # starts marinara service
  service 'marinara' do
    action :start
    provider Chef::Provider::Service::Upstart
  end
end
{% endcodeblock %}

- **Provision application firewall rule.**To complete the second exit criteria,
  I defined a firewall rule in the application recipe to allow traffic on the
  specified application port with the upstream HAProxy server as the source.

{% codeblock site-cookbooks/marinara/recipes/application.rb lang:ruby %}
# provisions firewall rule to support incoming application traffic
include_recipe 'simple_iptables'
server = node.marinara.proxy.server
port = node.marinara.application.port
simple_iptables_rule 'application' do
  rule "-p tcp -s #{server} --dport #{port}"
  jump 'ACCEPT'
end
{% endcodeblock %}

Implement Chef Proxy Recipe
---------------------------
To scale the application servers horizontally with the growth of product
usage, I implemented a Chef recipe to install and configure HAProxy. This
recipe leveraged the [haproxy][22] cookbook for most of the heavy lifting.
However, I had to *reopen* the template resource defined in that cookbook
to use my configuration template instead. The code snippet below demonstrates
this. In addition to providing custom configuration that references
deployed application servers, I implemented a firewall rule to accept traffic
on port 80 (the default value for the incoming_port attribute). In a future
week, I intend to tune HAProxy's configuration based on performance testing.

{% codeblock site-cookbooks/marinara/attributes/default.rb lang:ruby %}
# includes default haproxy cookbook attributes first to override
include_attribute 'haproxy'

# defines haproxy configuration
default.haproxy.install_method = 'source'
default.haproxy.source.version = '1.5-dev17'
default.haproxy.source.url = 'http://haproxy.1wt.eu/download/1.5/src/devel/haproxy-1.5-dev17.tar.gz'
default.haproxy.source.checksum = 'b8deab9989e6b9925410b0bc44dd4353'

default.marinara.proxy.server = '127.0.0.1'
{% endcodeblock %}

{% codeblock site-cookbooks/marinara/recipes/proxy.rb lang:ruby %}
# provisions haproxy
include_recipe 'haproxy'

# overrides haproxy cookbook default configuration
template = resources('template[/etc/haproxy/haproxy.cfg]')
template.source 'haproxy.cfg.erb'
template.cookbook 'marinara'

# provisions firewall rule to support incoming http traffic
include_recipe 'simple_iptables'
simple_iptables_rule 'proxy' do
  rule "-p tcp --dport #{node.haproxy.incoming_port}"
  jump 'ACCEPT'
end
{% endcodeblock %}

{% codeblock site-cookbooks/marinara/templates/default/haproxy.cfg.erb lang:ruby %}
global
  log 127.0.0.1 local0
  maxconn 4096
  user haproxy
  group haproxy

defaults
  log global
  mode http
  option httplog
  option dontlognull
  option redispatch
  option forwardfor
  timeout connect 5s
  timeout client 50s
  timeout server 50s
  balance <%= node.haproxy.balance_algorithm %>

frontend http
  bind 0.0.0.0:<%= node.haproxy.incoming_port %>
  default_backend application

backend application
  <% node.marinara.application.servers.each_pair do |name, addr| -%>
  server <%= name %> <%= addr %>:<%= node.marinara.application.port %> check
  <% end -%>
{% endcodeblock %}

Test with Multi-Machine Vagrant Environment
-------------------------------------------
To finish off the week, I created a *Vagrantfile* in the *marinara-kitchen*
repository that creates three virtual machines using VirtualBox. Specifically,
I created a definition for the HAProxy server and a definition for the
Node.js application servers running on a private network so they could
communicate with one another. Here is the configuration below using the new
Vagrant 1.1 syntax:

{% codeblock Vagrantfile lang:ruby %}
app_servers = {
  app0: '10.0.5.3',
  app1: '10.0.5.4'
}

Vagrant.configure('2') do |config|
  config.vm.box = 'precise64-chef11.2'
  config.vm.box_url = 'https://opscode-vm.s3.amazonaws.com/vagrant/opscode_ubuntu-12.04_chef-11.2.0.box'

  config.vm.provider :virtualbox do |vb|
    vb.customize ['setextradata', :id, 'VBoxInternal2/SharedFoldersEnableSymlinksCreate/v-root', '1']
  end

  config.ssh.forward_agent = true

  config.vm.define :proxy do |proxy|
    proxy.vm.network :private_network, ip: '10.0.5.2'
    proxy.vm.provision :chef_solo do |chef|
      chef.cookbooks_path = ['cookbooks', 'site-cookbooks']
      chef.data_bags_path = 'data_bags'
      chef.add_recipe 'marinara::default'
      chef.add_recipe 'marinara::security'
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

  app_servers.each_pair do |name, ip|
    config.vm.define :"#{name}" do |app|
      app.vm.network :private_network, ip: ip
      app.vm.provision :chef_solo do |chef|
      chef.cookbooks_path = ['cookbooks', 'site-cookbooks']
      chef.data_bags_path = 'data_bags'
        chef.add_recipe 'marinara::default'
        chef.add_recipe 'marinara::security'
        chef.add_recipe 'marinara::application'
        chef.json = {
          marinara: {
            application: {
              reference: 'develop'
            },
            proxy: {
              server: '10.0.5.2'
            }
          }
        }
      end
    end
  end
end
{% endcodeblock %}

  With the Vagrantfile implemented, I brought the servers up and
  executed a quick test with curl against my endpoints. With a successful
  response, the fourth criteria had been met and the week's task complete.

{% codeblock %}
$ vagrant up
$ curl http://10.0.5.2/
$ curl http://10.0.5.2/api/hello
{% endcodeblock %}

[1]: http://shawn.dahlen.me/blog/2013/03/06/automate-provisioning-of-secure-servers/
[2]: http://nodejs.org/
[3]: http://haproxy.1wt.eu/
[4]: http://www.vagrantup.com/
[5]: http://en.wikipedia.org/wiki/Single-page_application
[6]: http://jquery.com/
[7]: http://requirejs.org/
[8]: http://documentcloud.github.com/backbone/
[9]: http://expressjs.com/
[10]: https://npmjs.org/
[11]: http://upstart.ubuntu.com/cookbook/
[12]: https://github.com/visionmedia/log.js/
[13]: http://www.rsyslog.com/
[14]: http://underscorejs.org/#template
[15]: http://twitter.github.com/bower/
[16]: http://gruntjs.com/
[17]: http://livereload.com/
[18]: http://requirejs.org/docs/optimization.html
[19]: https://gist.github.com/smdahlen/5190695
[20]: https://github.com/mdxp/nodejs-cookbook
[21]: https://github.com/balbeko/chef-npm/
[22]: https://github.com/opscode-cookbooks/haproxy
