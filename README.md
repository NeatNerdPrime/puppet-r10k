# r10k Configuration Module
[![Puppet Forge](http://img.shields.io/puppetforge/v/puppet/r10k.svg)](https://forge.puppetlabs.com/puppet/r10k)
[![Build Status](https://travis-ci.org/voxpupuli/puppet-r10k.png?branch=master)](https://travis-ci.org/voxpupuli/puppet-r10k)
[![Github Tag](https://img.shields.io/github/tag/voxpupuli/puppet-r10k.svg)](https://github.com/voxpupuli/puppet-r10k)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/voxpupuli/puppet-r10k?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![Puppet Forge Downloads](http://img.shields.io/puppetforge/dt/puppet/r10k.svg)](https://forge.puppetlabs.com/puppet/r10k)
[![Puppet Forge Endorsement](https://img.shields.io/puppetforge/e/puppet/r10k.svg)](https://forge.puppetlabs.com/puppet/r10k)


#### Table of Contents

1. [Overview](#overview)
1. [Module Description - What the module does and why it is useful](#module-description)
1. [Setup - The basics of getting started with r10k](#setup)
    * [Prefix Example](#prefix-example)
    * [What r10k affects](#what-r10k-affects)
    * [Setup Requirements](#setup-requirements)
    * [Beginning with r10k](#beginning-with-r10k)
    * [Using an internal gem server](#using-an-internal-gem-server)
    * [Mcollective Support](#mcollective-support)
1. [Webhook Support](#webhook-support)
1. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
1. [Limitations - OS compatibility, etc.](#limitations)
1. [Support](#support)
1. [Development - Guide for contributing to the module](#development)
1. [Running tests](#running-tests)

## Overview

This module was built to install and configure r10k. It has a base class to configure r10k to
synchronize [dynamic environments](https://github.com/adrienthebo/r10k/blob/master/doc/dynamic-environments.mkd).
It also has a series of lateral scripts and tools that assist in general workflow, that will be separated into
their own modules into the future.

## Module Description

This module is meant to manage the installation and configuration of r10k using multiple installation methods on multiple platforms.

## Setup

Please refer to the official [r10k docs](https://github.com/puppetlabs/r10k/tree/master/doc) for specific configuration patterns.

### Prefix Example
Instead of passing a single `remote`, you can pass a puppet [hash](https://docs.puppetlabs.com/puppet/latest/reference/lang_datatypes.html#hashes) as the `sources`
parameter. This allows you to configure r10k with [prefix](https://github.com/puppetlabs/r10k/blob/910709a2924d6167e2e53e03d64d2cc1a64827d4/doc/dynamic-environments/configuration.mkd#prefix) support. This often used when multiple teams use separate repos, or if hiera and puppet are distributed across two repos.

```puppet
class { 'r10k':
  sources => {
    'webteam' => {
      'remote'  => 'ssh://git@github.com/webteam/somerepo.git',
      'basedir' => "${::settings::codedir}/environments",
      'prefix'  => true,
    },
    'secteam' => {
      'remote'  => 'ssh://git@github.com/secteam/someotherrepo.git',
      'basedir' => '/some/other/basedir',
      'prefix'  => true,
    },
  },
}
```
### What r10k affects

* Installation of the r10k `gem`
* Installation of ruby when not using an existing ruby stack i.e. when using `puppet_gem`
* Management of the `r10k.yaml` in /etc
* Installation and configuration of a sinatra app when using the [webhook](#webhook-support).


#### Version chart

Gem installation is pinned to a default version in this module, the following chart shows the gem installation tested with the respective module version.
You can override this by passing the `version` parameter.

| Module Version | r10k Version |
| -------------- | ------------ |
| v4.0.0+        | [![Latest Version](https://img.shields.io/gem/v/r10k.svg?style=flat-square)](https://rubygems.org/gems/r10k)        |
| v3.0.x         | 1.5.1        |
| v2.8.2         | 1.5.1        |
| v2.7.x         | 1.5.1        |
| v2.6.5         | 1.4.1        |
| v2.5.4         | 1.4.0        |
| v2.4.4         | 1.3.5        |
| v2.3.1         | 1.3.4        |
| v2.3.0         | 1.3.2        |
| v2.2.8         | 1.3.1        |
| v2.2.x         | 1.1.0        |


### Setup Requirements

r10k connects via ssh and does so silently in the background, this typically requires ssh keys to be deployed in advance of configuring r10k. This includes the known host ( public ) key of the respective git server, and the user running r10k's private key used to authenticate git/ssh during background runs.  If you are going to use git repos to retrieve modules, you also need git installed.

Here is an example of deploying the git package and ssh keys needed for r10k to connect to a repo called puppet/control on a gitlab server.  This is helpful when you need to automatically deploy new masters

```puppet
package { 'git':
  ensure => installed,
}

#https://docs.puppetlabs.com/references/latest/type.html#sshkey
sshkey { 'your.internal.gitlab.server.com':
  ensure => present,
  type   => 'ssh-rsa',
  target => '/root/.ssh/known_hosts',
  key    => '...+dffsfHQ==',
}

# Resource git_webhook is provided by https://github.com/bjvrielink/abrader-gms/tree/fixup
git_deploy_key { 'add_deploy_key_to_puppet_control':
  ensure       => present,
  name         => $facts['networking']['fqdn'],
  path         => '/root/.ssh/id_dsa.pub',
  token        => hiera('gitlab_api_token'),
  project_name => 'puppet/control',
  server_url   => 'http://your.internal.gitlab.server.com',
  provider     => 'gitlab',
}
```
A simple example of creating an ssh private key would use an exec to call `yes y | ssh-keygen -t dsa -C "r10k" -f /root/.ssh/id_dsa -q -N ''`.
The example above shows using `git_deploy_key` which would deploy that key to the remote git server via its api. This is often required in the programtic creation of compile masters.

Given r10k will likely be downloading your modules, often on the first server
it's run on, you will have to `puppet apply` this module to bootstrap this
configuration and allow for ongoing management from there.

### Beginning with r10k

The simplest example of using it would be to declare a single remote that would be written to r10k.yaml.

```puppet
class { 'r10k':
  remote => 'git@github.com:someuser/puppet.git',
}
```
This will configure `/etc/r10k.yaml` and install the r10k gem after installing
ruby using the [puppetlabs/ruby](http://forge.puppetlabs.com/puppetlabs/ruby) module.

It also supports installation via multiple providers, such as installation in the puppet_enterprise ruby stack in versions less than 3.8

Installing into the Puppet Enterprise ruby stack in PE 2015.x

```puppet
class { 'r10k':
  remote   => 'git@github.com:someuser/puppet.git',
  provider => 'puppet_gem',
}
```

_Note: It is recommended you migrate to using the `pe_r10k` module which is basically
a clone of this modules features and file tickets for anything missing._


### Using an internal gem server

Depending on implementation requirements, there are two ways to use alternate gem sources.

#### The gemrc approach
Create a global gemrc for Puppet Enterprise to add the local gem source. See http://projects.puppetlabs.com/issues/18053#note-12 for more information.

```puppet
file { '/opt/puppet/etc':
  ensure => 'directory',
  owner  => 'root',
  group  => '0',
  mode   => '0755',
}

file { 'gemrc':
  ensure  => 'file',
  path    => '/opt/puppet/etc/gemrc',
  owner   => 'root',
  group   => '0',
  mode    => '0644',
  content => "---\nupdate_sources: true\n:sources:\n- http://your.internal.gem.server.com/rubygems/\n",
}

class { 'r10k':
  remote   => 'git@github.com:someuser/puppet.git',
  provider => 'pe_gem',
  require  => File['gemrc'],
}
```

#### The parameter approach
Add gem_source to declaration.

```puppet
class { 'r10k':
  remote      => 'git@github.com:someuser/puppet.git',
  provider    => 'gem',
  gem_source  => 'https://some.alternate.source.com/',
}
```

### Mcollective Support
![alt tag](https://gist.githubusercontent.com/acidprime/7013041/raw/1a99e0a8d28b13bc20b74d2dc4ab60c7e752088c/post_recieve_overview.png)

An mcollective agent is included in this module which can be used to do
on demand synchronization. This mcollective application and agent can be
installed on all masters using the following class
_Note: You must have mcollective already configured for this tool to work,
Puppet Enterprise users will automatically have mcollective configured._
This class does not restart the mcollective or pe-mcollective server on the
nodes to which it is applied, so you may need to restart mcollective for it
to see the newly installed r10k agent.
```puppet
include r10k::mcollective
```

Using mco you can then trigger mcollective to call r10k using

```shell
mco r10k synchronize
```

You can sync an individual environment using:

```shell
mco r10k deploy <environment>
```
Note: This implies `-p`

You can sync an individual module using:

```shell
mco r10k deploy_module <module>
```

If you are required to run `r10k` as a specific user, you can do so by passing
the `user` parameter:

```shell
mco r10k synchronize user=r10k
```

To obtain the output of running the shell command, run the agent like this:

```shell
mco rpc r10k synchronize -v
```

An example post-receive hook is included in the files directory.
This hook can automatically cause code to synchronize on your
servers at time of push in git. More modern git systems use webhooks, for  those see below.

#### Passing proxy info through mco

The mcollective agent can be configured to supply r10k/git environment `http_proxy`, `https_proxy` variables via the following example

```puppet
class { 'r10k::mcollective':
  http_proxy     => 'http://proxy.example.lan:3128',
  git_ssl_no_verify => 1,
}
```

#### Install mcollective support for post receive hooks
Install the `mco` command from the puppet enterprise installation directory i.e.
```shell
cd ~/puppet-enterprise-3.0.1-el-6-x86_64/packages/el-6-x86_64
sudo rpm -i pe-mcollective-client-2.2.4-2.pe.el6.noarch.rpm
```
Copy the peadmin mcollective configuration and private keys from the certificate authority (puppet master)
~~~
/var/lib/peadmin/.mcollective
/var/lib/peadmin/.mcollective.d/mcollective-public.pem
/var/lib/peadmin/.mcollective.d/peadmin-cacert.pem
/var/lib/peadmin/.mcollective.d/peadmin-cert.pem
/var/lib/peadmin/.mcollective.d/peadmin-private.pem
/var/lib/peadmin/.mcollective.d/peadmin-public.pem
~~~
Ensure you update the paths in _~/.mcollective_ when copying to new users whose name is not peadmin.
Ideally mcollective will be used with more then just the peadmin user's certificate
in the future. That said, if your git user does not have a home directory, you can rename .mcollective as /etc/client.cfg
and copy the certs to somewhere that is readable by the respective user.
~~~
/home/gitolite/.mcollective
/home/gitolite/.mcollective.d/mcollective-public.pem
/home/gitolite/.mcollective.d/peadmin-cacert.pem
/home/gitolite/.mcollective.d/peadmin-cert.pem
/home/gitolite/.mcollective.d/peadmin-private.pem
/home/gitolite/.mcollective.d/peadmin-public.pem
~~~
_Note: PE2 only requires the .mcollective file as the default auth was psk_

#### Removing the mcollective agent

```puppet
class { 'r10k::mcollective':
  ensure => false,
}
```
This will remove the mcollective agent/application and ddl files from disk. This likely would be if you are migrating to Code manager in Puppet Enterprise.

## Webhook Support

![alt tag](https://gist.githubusercontent.com/acidprime/be25026c11a76bf3e7fb/raw/44df86181c3e5d14242a1b1f4281bf24e9c48509/webhook.gif)
For version control systems that use web driven post-receive processes you can use the example webhook included in this module.
When the webhook receives the post-receive event, it will synchronize environments on your puppet masters.
These settings are all configurable for your specific use case, as shown below in these configuration examples.

**NOTE: MCollective and Bolt aren't currently supported with Webhook Go. This will be addressed in a future release of Webhook Go, but is an issue related to the complex nature of Bolt and MCollective/Choria commands that cause issues with the way Go executes shell commands.**

### Webhook Github Enterprise - Non Authenticated
This is an example of using the webhook without authentication.
The `git_webhook` type will use the [api token](https://help.github.com/articles/creating-an-access-token-for-command-line-use/) to add the webhook to the "control" repo that contains your puppetfile. This is typically useful when you want to automate the addition of the webhook to the repo.

```puppet
# Instead of running via mco, run r10k directly
class {'r10k::webhook::config':
  use_mcollective => false,
}

class {'r10k::webhook':
  ensure         => true,
  server         => {
    protected => false,
  },
}

# Add webhook to control repository ( the repo where the Puppetfile lives )
#
# Resource git_webhook is provided by https://github.com/bjvrielink/abrader-gms/tree/fixup
git_webhook { 'web_post_receive_webhook' :
  ensure       => present,
  webhook_url  => 'http://master.of.masters:8088/payload',
  token        =>  hiera('github_api_token'),
  project_name => 'organization/control',
  server_url   => 'https://your.github.enterprise.com',
  provider     => 'github',
}


# Add webhook to module repo if we are tracking branch in Puppetfile i.e.
# mod 'module_name',
#  :git    => 'http://github.com/organization/puppet-module_name',
#  :branch => 'master'
# The module name is determined from the repo name , i.e. <puppet-><module_name>
# All characters with left and including any hyphen are removed i.e. <puppet->
#
# Resource git_webhook is provided by https://github.com/bjvrielink/abrader-gms/tree/fixup
git_webhook { 'web_post_receive_webhook_for_module' :
  ensure       => present,
  webhook_url  => 'http://master.of.masters:8088/module',
  token        =>  hiera('github_api_token'),
  project_name => 'organization/puppet-module_name',
  server_url   => 'https://your.github.enterprise.com',
  provider     => 'github',
}
```

### Webhook Github Example - Authenticated
This is an example of using the webhook with authentication.
The `git_webhook` type will use the [api token](https://help.github.com/articles/creating-an-access-token-for-command-line-use/) to add the webhook to the "control" repo that contains your puppetfile. This is typically useful when you want to automate the addition of the webhook to the repo.

```puppet
# Instead of running via mco, run r10k directly
class {'r10k::webhook::config':
  use_mcollective => false,
}

# External webhooks often need authentication and ssl and authentication
# Change the url below if this is changed

class {'r10k::webhook':
  ensure => true,
  server => {
      protected => true,
  },
  tls    => {
      enabled     => true,
      certificate => '/path/to/ssl/certificate',
      key         => '/path/to/ssl/key',
  },
}

# Add webhook to control repository ( the repo where the Puppetfile lives )
#
# Resource git_webhook is provided by https://github.com/bjvrielink/abrader-gms/tree/fixup
git_webhook { 'web_post_receive_webhook' :
  ensure             => present,
  webhook_url        => 'https://puppet:puppet@hole.in.firewall:8088/payload',
  token              =>  hiera('github_api_token'),
  project_name       => 'organization/control',
  server_url         => 'https://api.github.com',
  disable_ssl_verify => true,
  provider           => 'github',
}

# Add webhook to module repo if we are tracking branch in Puppetfile i.e.
# mod 'module_name',
#  :git    => 'http://github.com/organization/puppet-module_name',
#  :branch => 'master'
# The module name is determined from the repo name , i.e. <puppet-><module_name>
# All characters with left and including any hyphen are removed i.e. <puppet->
#
# Resource git_webhook is provided by https://github.com/bjvrielink/abrader-gms/tree/fixup
git_webhook { 'web_post_receive_webhook_for_module' :
  ensure       => present,
  webhook_url  => 'https://puppet:puppet@hole.in.firewall:8088/module',
  token        =>  hiera('github_api_token'),
  project_name => 'organization/puppet-module_name',
  server_url   => 'https://api.github.com',
  disable_ssl_verify => true,
  provider     => 'github',
}
```

### Webhook Bitbucket Example
This is an example of using the webhook with Atlassian Bitbucket (former Stash).
Requires the `external hooks` addon by https://marketplace.atlassian.com/plugins/com.ngs.stash.externalhooks.external-hooks/server/overview
and a specific Bitbucket user/pass.
Remember to place the `stash_mco.rb` on the bitbucket server an make it executable.
Enable the webhook over the repository settings `External Async Post Receive Hook`:
 - Executable: e.g. `/opt/atlassian/bitbucket-data/external-hooks/stash_mco.rb` (see hook_exe)
 - Positional parameters: `-t http://git.example.com:8088/payload`

```puppet
# Add deploy key
git_deploy_key { 'add_deploy_key_to_puppet_control':
  ensure       => present,
  name         => $facts['networking']['fqdn'],
  path         => '/root/.ssh/id_rsa.pub',
  username     => 'api',
  password     => 'pass',
  project_name => 'project',
  repo_name    => 'puppet',
  server_url   => 'https://git.example.com',
  provider     => 'stash',
}

# Add webhook
git_webhook { 'web_post_receive_webhook' :
  ensure       => present,
  webhook_url  => 'https://puppet:puppet@hole.in.firewall:8088/module',
  password     => 'pass',
  username     => 'api',
  project_name => 'project',
  repo_name    => 'puppet',
  server_url   => 'https://git.example.com',
  provider     => 'stash',
  hook_exe     => '/opt/atlassian/bitbucket-data/external-hooks/stash_mco.rb',
}
```

### Webhook - remove webhook init script and config file.
For use when moving to Code Manager, or other solutions, and the webhook should be removed.
```puppet
class {'r10k::webhook':
  ensure => false,
}
```

### Webhook Prefix Example

Prefixing the command is currently not supported in Webhook Go. This support is expected to be added with a later release.

### Webhook FOSS support with MCollective

MCollective is currently unsupported by Webhook Go. This is expected to be added in a future release and documentation will be updated for that then.

### Webhook Slack notifications

You can enable Slack notifications for the webhook. You will need a
Slack webhook URL and the `slack-notifier` gem installed.

To get the Slack webhook URL you need to:

1. Go to
   [https://slack.com/apps/A0F7XDUAZ-incoming-webhooks](https://slack.com/apps/A0F7XDUAZ-incoming-webhooks).
2. Choose your team, press `Configure`.
3. In configurations press `Add configuration`.
4. Choose channel, press `Add Incoming WebHooks integration`.

Then configure the webhook to add your Slack Webhook URL.

```puppet
class { 'r10k::webhook':
  . . .
  chatops => {
      enabled    => true,
      service    => 'slack',
      server_uri => 'http://slack.webhook/webhook', # mandatory for usage
      channel    => '#channel', # defaults to #default
      user       => 'r10k', # the username to use
      auth_token => "SLACKAUTHTOKEN",
    }
  }
```

### Webhook Rocket.Chat notifications

You can enable Rocket.Chat notifications for the webhook. You will need a
Rocket.Chat incoming webhook URL and the `rocket-chat-notifier` gem installed.

To get the Rocket.Chat incoming webhook URL you need to:

1. Go to your Rocket.Chat and then select `Administration-Integrations`.
2. Choose `New integration`.
3. Choose `Incoming WebHook`. In the webhook form configure:
  * `Enabled`: `True`.
  * `Name`: A name for your webhook.
  * `Post to Channel`: The channel to post to by default.
4. Save changes with `Save Changes` bottom.

Then configure the webhook to add your Rocket.Chat Webhook URL.

```puppet
class { 'r10k::webhook':
  . . .
  chatops => {
    enabled    => true,
    service    => 'rocketchat',
    server_uri => '<your incoming webhook URL>',
    user       => 'username',
    channel    => '#channel',
    auth_token => 'ROCKETCHATAUTHTOKEN',
  }
}
```

### Webhook Default Branch

The default branch of the controlrepo is commonly called `production`. This value can be overridden if you use another default branch name, such as `master`.

```puppet
class { 'r10k::webhook':
  ensure => true,
  r10k   => {
    default_branch => 'master', # Optional. Defaults to 'production'
  },
}
```

### Triggering the webhook from curl

To aid in debugging, or to give you some hints as to how to trigger the webhook by unsupported systems, here's a curl command to trigger the webhook to deploy the 'production' environment:

```bash
curl --header "X-Gitlab-Event: Push Hook" -d '
  {
    "repository": {"name": "foo", "owner": {"login": "foo"}},
    "ref": "production"
  }' http://puppet-master.example:4000/api/v1/r10k/environment
```

If you are utilizing environment prefixes, you'll need to specify the full environment title (including the prefix) in the 'ref' parameter:

```bash
curl --header "X-Gitlab-Event: Push Hook" -d '
  {
    "repository": {"name": "bar", "owner": {"login": "foo"}},
    "ref": "bar_production"
  }' http://puppet-master.example:4000/api/v1/r10k/environment
```

### Troubleshooting

If you're not sure whether your webhook setup works:

- Try to make a GET request to the `heartbeat` endpoint (e.g. http://puppet-master.example:8088/heartbeat).
  You should see a short JSON answer similar to `{"status":"success","message":"running"}`.
- Watch the webhook logfile at `/var/log/webhook/access.log`, and send requests (e.g. using curl).
  Example output if successful:

``` bash
$ journalctl -f -u webhook-go.service
...
Jun 05 11:24:54 pop-os systemd[1]: Started Puppet Deployment API Server....
```

### Docker

If you are building your image with the puppet, you need to prevent the webhook process from starting as a daemon.

The following is an example of declaring the webhook without a background mode

```puppet
class { 'r10k::webhook':
  ensure => false,
}
```

### Ignore deploying some environments

Since [2.10.0](https://github.com/voxpupuli/webhook-go/releases/tag/v2.10.0) the webhook has support for ignoring certain branches.
This is not yet configureable via the puppet module.

### configuring the webservice/deploy user

For historic reasons, webhook-go runs as root and executes r10k with the same user.
Via `r10k::webhook::service_user` you can change the user.
With the 15.0.0 release the default will switch from root to puppet.

## Reference

#### Class: `r10k`
This is the main public class to be declared , handingly installation and configuration declarations

**Parameters within `r10k`:**

##### `remote`
A string to be passed in as the source with a hardcode prefix of `puppet`

##### `sources`
A hash of all sources, this gets read out into the file as yaml. Must not be declared with `remote`

##### `cachedir`
A single string setting the `r10k.yaml` configuration value of the same name

##### `configfile`
A path to the configuration file to manage. Be aware Puppet Enterprise 4.0 and higher may conflict if you manage `/etc/puppetlabs/puppet/r10k.yaml`

##### `version`
A value passed to the package resource for managing the gem version

##### `modulepath`
Deprecated: for older [configfile](https://docs.puppetlabs.com/puppet/latest/reference/environments_classic.html) environments configuration of modulepath in puppet.conf

##### `manage_modulepath`
Deprecated: declare a resource for managing `modulepath` in Puppet.conf

##### `proxy`
A string setting the`r10k.yaml` configuration value of the same name

##### `gem_source`
An optional string specifying location to retrieve gem

##### `pool_size`
Integer defining how many threads should be spawn while updating modules. Only available for r10k >= 3.3.0.

##### `r10k_basedir`

##### `package_name`
The name of the package to be installed via the provider

##### `provider`
The supported installation modes for this module

* bundle
* puppet_gem
* gem

##### `install_options`
Options to pass to the `provider` declaration

##### `mcollective`
Install mcollective application and agents. This does NOT configure mcollective automatically

##### `manage_configfile_symlink`
Manage a symlink to the configuration file, for systems installed in weird file system configurations

##### `git_settings`
This is the `git:` key in r10k, it accepts a hash that can be used to configure
rugged support.

```puppet
    $git_settings = {
      'provider'    => 'rugged',
      'private_key' => '/root/.ssh/id_rsa',
    }

    class {'r10k':
      remote       => 'git@github.com:acidprime/puppet.git',
      git_settings => $git_settings,
    }
```

##### `forge_settings`
This is the `forge:` key in r10k, it accepts a hash that contains settings for downloading modules from the Puppet Forge.

```puppet
    $forge_settings = {
      'proxy'   => 'https://proxy.example.com:3128',
      'baseurl' => 'https://forgeapi.puppetlabs.com',
    }

    class {'r10k':
      remote         => 'git@github.com:acidprime/puppet.git',
      forge_settings => $forge_settings,
    }
```

##### `deploy_settings`
This is the `deploy:` key in r10k, it accepts a hash that contains setting that control how r10k code deployments behave. Documentation for the settings can be found [here](https://docs.puppet.com/pe/latest/r10k_custom.html#deploy).

```puppet
    $deploy_settings = {
      'purge_levels' => ['puppetfile'],
    }

    class {'r10k':
      remote          => 'git@github.com:voxpupuli/puppet.git',
      deploy_settings => $deploy_settings,
    }
```

##### `configfile_symlink`
boolean if to manage symlink

##### `include_prerun_command`
Deprecated: Add [prerun_command](https://docs.puppetlabs.com/references/latest/configuration.html#preruncommand) to puppet.conf to run r10k when the agent on the master runs.
Suggest instead declaring `r10k::postrun_command ` as that will run after the agent runs which prevents r10k from stopping configuration management of masters from occurring as it does with `prerun_command`s

##### `include_postrun_command`
```
r10k::include_postrun_command: true
```

The concept here is that this is declared on the puppet master(s) that have
been configured with r10k. This will cause r10k to synchronize after each
puppet run. Any errors synchronizing will be logged to the standard puppet run.

## Limitations
The 4.1.x release *deprecates* support for:
* Puppet 3
* Ruby 1.9.3

These items are planned for removal in v5.0.0.

## Support

Please log tickets and issues at our [Projects site](https://github.com/voxpupuli/puppet-r10k/issues)

## Development

### Contributing

Modules on the Puppet Forge are open projects, and community contributions are essential for keeping them great. We can’t access the huge number of platforms and myriad of hardware, software, and deployment configurations that Puppet is intended to serve.

We want to keep it as easy as possible to contribute changes so that our modules work in your environment. There are a few guidelines that we need contributors to follow so that we can have a chance of keeping on top of things.

Please see [CONTRIBUTING](.github/CONTRIBUTING.md) for more details.

### Running tests

This project contains tests for [rspec-puppet](http://rspec-puppet.com/) to
verify functionality. For in-depth information please see their respective
documentation, as well as [CONTRIBUTING](.github/CONTRIBUTING.md).

Quickstart:

```
    gem install bundler
    bundle install --without system_tests
    bundle exec rake test
    bundle exec rake lint
```

Check the .travis.yml for supported Operating System Versions
