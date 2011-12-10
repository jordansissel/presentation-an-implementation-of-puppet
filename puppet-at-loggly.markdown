title: Puppet at Loggly
gradient-colors: #000000
color: white

!SLIDE title

<h2> Puppet at Loggly </h2>

Jordan Sissel. hacker. Loggly, Inc.

# Puppet at Loggly

Our puppet deployment is:

* in EC2 (using RightScale)
* also on vmware/physical hosts (for random dev work)
* nodeless (pure fact-driven)
* masterless (no puppet master)

Uses:

* custom facts
* extlookup
* custom parser functions
* exported resources
* exported custom defines

# Infrastructure in MVC

* Model (Puppet Resources)
* View (applied configuration)
* Controller (Truth + Puppet)

A source of Truth feeds the Model and results in an applied configuration.

# Agenda

* Style
* Truth
* Nodeless Puppet
* Exported Resoruces
* Masterless Puppet

# Style - Modules

I never use 'import'. I always let puppet determine the path of a class.

Your puppet repository should look like this:

    manifests/site.pp
    modules/MODULENAME/{manifests,files,templates,plugins}/

    # Examples:
    modules/apache/manifests/server.pp
    modules/apache/templates/httpd.conf.erb
    modules/ssh/manifests/server.pp
    modules/ssh/files/sshd_config

* Let puppet be smart:
  * use a class: 'include foo::bar::baz'
  * automatically looks for it here: modules/foo/manifests/bar/baz.pp

# Style - File Paths

Puppet lets you abstract things away from the madness of your OS, let it.

* Use module-relative paths.
    * source => `puppet:///modules/MODULE/PATH`
    * maps to: _modulepath_/MODULE/files/PATH
* Don't include the full target path in a file name.

Example:

    class ssh::server {
      file {
        "/etc/ssh/sshd_config":
          #source => "puppet:///modules/ssh/etc/ssh/sshd_config", # Bad!
          source => "puppet:///modules/ssh/sshd_config";          # Good!
      }
    }
  
# Style - Avoid Coupling (variables)

* Don't use variables from other classes
* Avoid trying to expose variables to other classes; instead use custom
  defines.

      # Bad
      include apache::server     # sets '$configpath'
      include apache::package
      file {
        "${apache::server::configpath}/mysite.conf":
          source => "puppet:///...",
          require => Class["apache::package"]
          notify => Class["apache::server"];
      }

# Styles - Avoid Coupling (variables)

* Use a custom define to abstract a general-purpose apache config entry:

      # In modules/apache/manifests/config.pp
      define apache::config($source) {
        include apache::server
        include apache::package
        file {
          "/etc/apache/conf.d/$name.conf":
            source => $source,
            require => Class["apache::package"]
            notify => Class["apache::server"];
        }
      }

      # Abstract all the complexity into a reusable define:
      apache::config {
        "mysite":
          source => "puppet:///...";
      }

# Style - Avoid Coupling (resources)

* Don't require resources you haven't declared in a class.

      class apache::server {
        include apache::package    # defines Package["apache2"]

        service {
          "apache":
            require => Package["apache2"],        # Bad!
            require => Class["apache::package"],  # Good!
            ... ;
        }
      }

# Style - Avoid Coupling (resources)

* Split 'service' and 'package' resources. Require the 'package', notify the 'service'

      class apache::package {
        # any package/files/users for apache
      }
      class apache::server {
        include apache::package
        # any services for apache
      }
      class nagios::web { # custom define, includes apache::package and ::server
        apache::config {
          "nagios":
            source => "puppet:///modules/nagios/web/nagios.httpd.conf";
        }
      }

# Style - Manifest Files

* Don't define multiple classes or defines in a single manifest file.
* Don't nest classes or defines in a file.

Example:

<table>
  <tr>
  <td>
  <pre>
    # modules/foo/manifests/init.pp
    class foo {
      class bar {
        # Bad!
      }
    }
  </pre>
  </td>
  <td>
  <pre>
    # Good
    # modules/foo/manifests/init.pp
    class foo { ... }

    # modules/foo/manifests/bar.pp
    class foo::bar { ... }
  </pre>
  </td>
  </tr>
</table>

# Style - Custom Defines

* Custom defines so we can reuse code.
* Write custom defines to abstract resources.
* Split complex classes and defines into smaller ones.
* Even single-line classes are OK.
* Dependencies on custom defines:

      some::custom::define {
        "myname": ... ;
      }

      file {
        "/tmp/x"
          require => Some::Custom::Define["myname"];
      }

# Style - Testing

* Currently have bare minimum puppet testing. Want more.
* Syntax check all puppet manifests:

      find modules manifests -name '*.pp' \
        | xargs -t -n1 -P2 sh -c 'puppet --parseonly "$@" || exit 255' -

* Syntax check templates:

      find modules/*/templates/ -maxdepth 1 -type f -not -name .svn \
        | xargs -n1 sh -c 'erb -x -T - $1 | ruby -c 2>&1 | sed -e "s,^,$1: ,"' -

* For now, can only check one manifest per puppet invocation (see workaround: bug #4177)
* If any syntax test fails, we abort the package build.
* Prevents accidental push of bad puppet configs.

# Truth

Two main kinds of truth:

<% step do %>
* Machine truth, "I am a frontend"
* Sources of machine truth: extlookup (by hostname), RightScale API, EC2
  userdata, etc.
<% end %>
<% step do %>
* Deployment truth, "The frontend role has package 'loggly-frontend' with version 1.2345"
* Sources of deployment truth: extlookup (by environment/deployment), RightScale API, etc
<% end %>

<% step do %>
* Possible synonyms for 'deployment': cluster, environment, site.
<% end %>

# Machine Truth on RightScale

<% step do %>
* Tag each machine with roles: frontend, solr, zookeeper, monitor, etc.
* Use API to query all machines and tags in a deployment.
* Also tag with any other machine-specific data (zookeeper id, etc).
<% end %>

<% step do %>
<center>
<table style="background-color: white; color: black">
  <td valign="top"> <b> Tag(s): </b> </td>
  <td> 
    role:solr=true   <br/>
    role:zookeeper=true  <br/>
    solr:levels=4,5  <br/>
    zookeeper:id=3 <br/>
  </td>
</table>
</center>
<% end %>

# Deployment Truth

* Deployments are 'production,' 'staging,' etc.
* Each has the same roles available (frontend, memcache, etc), but different
  configurations/tuning.  
* Could have totally independent roles between deployments. Depends on your
  business needs.
* I use extlookup.

      # in site.pp:
      # '$deployment' fact comes from machine truth. What deployment are we?
      $extlookup_precedence = [ "%{deployment}/config", "common" ]

# Deployment Truth (example)

* Example _production_/config.csv (specifying tunables)

      package/loggly-frontend,3045.trunk
      package/loggly-monitoring-membase,latest
      config/s3_enable,true
      config/s3_enable_customers,all

* Example `modules/loggly/manifests/frontend.pp` (uses extlookup!)

      package {
        "loggly-frontend":
          ensure => extlookup("package/loggly-frontend");
      }

# Truth - Role-based deployment

<% step do %>
* Every application component gets a role. 
<% end %>
<% step do %>
* Even tiny stuff like standalone cron jobs.
<% end %>
<% step do %>
* Roles: frontend, mysql-master, db-backup, s3-cleaner, monitor, etc.
<% end %>

# Truth - Defining a Role

    class loggly::frontend {
      # include any other required classes

      package {
        "loggly-frontend": ensure => extlookup("package/loggly-frontend");
      }

      iptables::rule {
        "allow http": ports => 80;
        "allow https": ports => 443;
      }

      apache::site {
        "loggly-frontend":
          source => template("loggly/frontend/loggly-frontend.httpd.conf.erb");
      }
    }

# Truth - Defining a Role

Result of previous slide's config:

* dpkg -l loggly-frontend: `loggly-frontend   3044.trunk`
* /etc/apache2/sites-enabled/loggly-frontend.conf exists
* /etc/iptables.d/...

      -A INPUT -s 0.0.0.0/0 -p tcp --syn --dport 80 -j ACCEPT -m comment \
        --comment "allow http (any source)"
      -A INPUT -s 0.0.0.0/0 -p tcp --syn --dport 443 -j ACCEPT -m comment \
        --comment "allow https (any source)"

# Truth - Defining a Role

All features of a role should be defined in that class.

* Define firewall rules (even if you export this to a network device)
* Define monitoring checks (even if you export this to another server)
* Define the user the role programs run as
* Define configs, packages, etc.
* Abstract what you can into custom defines (iptables::rule, apache::site)

# Nodeless Puppet

* Use truth/facts to guide puppet.
* No `node somehost { }` definitions
* If you use an external classifier, do not specify 'classes' - only send 'parameters'

# Why Nodeless?

* One place for configuration logic.

Example:

* Are we a frontend? Ok, install the frontend.
* Not a frontend? Ok, make sure the frontend is not installed.

      if has_role("frontend") {
        include frontend
      } else {
        include frontend::remove
      }

# Nodeless Puppet (site.pp)

A nodeless site.pp is practically empty.

    # manifests/site.pp
    node default {
      include truth::enforcer
    }

# Nodeless Puppet (truth::enforcer)

    # modules/truth/manifests/enforcer.pp
    class truth::enforcer {
      # For each role, include 1 class that is that role.
      if has_role("statsserver") {
        include loggly::statsserver
      } else {
        # Ensure this server has no 'statsserver' if we are not a statsserver.
        include loggly::stattserver::remove
      }

      ...
    }

* `has_role()` is a custom puppet function I wrote 
* makes it easy to handle the `role:rolename=true` tags.

# Custom Defined Resources

Custom defines let you create your own resource types that wrap other resources.

    # Simplified version:
    define supervisor::program($command, $user, $notifycmd="", $directory="/",
                               $ensure="present") {
      include supervisor

      file {
        "/etc/supervisor/conf.d/$name.conf":
          ensure => $file_ensure,
          content => template("supervisor/program.erb"),
          notify => Exec["poke supervisord"];
      }

    }


# Example Custom Defines - supervisor

    supervisor::program {
      "loggly-solrserver":
        command => "/opt/loggly/solrserver/solrserver.sh",
        user => "root", # for ulimit, will drop privs before launching.
        subscribe => Class["loggly::config", "loggly::solr"],
        require => [User["loggly-solrserver"],
                    Class["loggly::common", "zeromq::java"],
                    File["/opt/loggly/solrserver/solrserver.sh",
                         "/opt/solr/dist/etc/log4j.properties"]];
    }

# Example Custom Defines - iptables

    iptables::rule {
      "zookeeper client":
        roles => ["solr", "zookeeper"],
        ports => 2181;
      "zookeeper ensemble":
        roles => ["zookeeper"],
        ports => [2888, 3888];
    }

# Example Custom Defines - nagios

    # The '@@' notation means 'export this resource'
    # More on exported resources later.
    @@nagios::host {
      "$fqdn":
        address => $ipaddress_eth0,
        tag => "deployment::$deployment";
    }

    @@nagios::check {
      "disk space on $fqdn":
        command => "check-disk-space",
        remote => true,
        host => $fqdn,
        contacts => "pagerduty",
        tag => "deployment::$deployment";
    }

# Exported Resources

* Export resource definitions to a central database
* Collect exported resources on nodes
* You can export custom defines.

Biggest win: 

**Treat your infrastructure as a collective (or multiple collectives) rather
than as individual, standalone hosts**

# Exported Resources

* Keeping all your role/service logic in one place.
* Generating nagios configs
* Generating DNS configs
* Sharing node ssh public keys

# Exported Resources: Nagios

* Export NRPE checks
* Export host definitions
* Export service definitions
* I use custom defines for this.

# Exported Resources: Nagios

Exporting a check (a custom define):

    @@nagios::check {
      "httpinput end-to-end test from $fqdn":
        command => "endtoend-httpinput",   # Command to run
        remote => true,                    # Is an NRPE check
        host => $fqdn,                     # Host to target
        contacts => "pagerduty",           # Contact
        tag => "deployment::$deployment";  # Workaround for bug#5239
    }

    nagios::command {
      "endtoend-httpinput":
        command => "/usr/local/bin/endtoend.py http",
        remote => true; # register this command with NRPE
    }


# Collecting Exported Checks

Collect all checks exported in our deployment (the ```<<| ... |>>``` syntax):

    # Query by tag to work around puppet bug #5239
    Nagios::Check <<| tag == "deployment::$deployment" |>> {
      notify => Class["nagios::server"]
    }

Results:

    % ls /etc/nagios3/checks.d 
    check-frontend1.example.com-disk space on frontend1.example.com.cfg
    check-frontend2.example.com-disk space on frontend2.example.com.cfg
    check-ops.example.com-disk space on ops.example.com.cfg
    check-proxy1.example.com-disk space on proxy1.example.com.cfg
    check-proxy2.example.com-disk space on proxy2.example.com.cfg
    ...

Code to be released soon.

# Exported Resources (caveats)

What if you shutdown an EC2 instance and it no longer needs monitoring?

* No default way to 'purge' old resources
* Can use 'tidy' to purge files not updated recently:

      tidy {
        [ "/etc/nagios3/hosts.d", "/etc/nagios3/checks.d" ]:
          age => 1d,
          notify => Class["nagios::server"],
          recurse => true,
          matches => [ "*.cfg" ];
      }

* Not ideal. Another option is hacking in resource timestamps.
  Google for 'puppet exported resource expiration'

# Exported Resources (caveats)

Scaling problems:

* Puppet users are not necessarily Rails/ActiveRecord scaling experts. 
* thin_storeconfigs and async_storeconfigs can help, but don't solve everything.
* Schema: Expensive read and write query model.
* Schema: Reads: Nodes X Resources X Parameter Names X Parameter Values
* Geographically distant mysql servers hard to query quickly. Scaling not obvious.

# Exported Resources (caveats)

* Chicken and egg: Seems obvious, but exports aren't available until they are exported.
* Bug: Exported facts override local facts (really really bad; #5212)
* Bug: Can't query by environment (workaround: use tags; #5239)
* Bug: Can't query for your own exports (workaround: use tags; #????)
* Bug: Can't use SSL to MySQL (patch available; #5235)
* Bug: Can't update environment value (#5213)

# Masterless Puppet

Overview:

* No puppet master required - standalone execution
* Can still use storeconfigs, tags, exported resources, etc.
* Works perfectly with a nodeless configuration.
* Instead of `puppet agent` you `puppet apply`

# Masterless Puppet

* You can run puppet code from the command line and as non-root.
* Great for testing things even if you have a puppet master.

      % puppet apply -e 'file { "/tmp/x": content => "Hello world\n"; }'
      notice: /Stage[main]//File[/tmp/x]/ensure: defined content as 
      '{md5}f0ef7081e1539ac00ef5b761b4fb01b3'

      % cat /tmp/x
      Hello world

* Alternately, run a puppet manifest: `puppet apply mymanifest.pp`

# Masterless and Storeconfigs

<% step do %>
* I still wanted storeconfigs (for exported resources).
<% end %>
<% step do %>
* No obvious place to run the database.
<% end %>
<% step do %>
* Amazon RDS to the rescue! (Launches a managed mysql instance for me. Win!)
<% end %>

# Masterless Benefits

* No master/agent SSL problems (common in the #puppet IRC channel)
* No mongrel/webrick/passenger madness
* No 'how do I scale my master' question
* Storeconfigs still works! :)
* Less chicken-and-egg; can start a standalone deployment with simply one
  `puppet apply`.

      # Masterless example:
      % puppet --modulepath /path/to/modules /path/to/manifests.site.pp

# Masterless Caveats

In general: You have to solve problems already solved with the master:

* Not a popular solution. Few people run it.
* Puppet kick, puppet queue, and other features probably don't work. (needs verification)
* Probably no puppet reports (haven't tried)
* Can't rely on fileserver auth for security. Requires your own deployment mechanism (for each node)
* You can't run the agent. You have to periodically run 'puppet apply'

# Masterless: Lifecycle of a Puppet Run

We have a script run via cron doing:

<% step do %>
* apt-get install loggly-puppet
    * This downloads our latest puppet manifests and modules.
    * Needs to be outside of puppet so we can repair 'broken puppet' problems
      like syntax errors or other catalog problems.
<% end %>

<% step do %>
* puppet --environment prerun manifests/prerun.pp
    * Fetches latest truth (Query RightScale for machines/tags/properties)
    * Does basic sanity checking and bootstrapping fixes for the real puppet run.
    * Upgrades puppet, makes storeconfigs work, etc.
<% end %>

<% step do %>
* puppet --storeconfigs manifests/site.pp
    * This is the main puppet run.
    * The 'environment' is set in puppet.conf by puppet itself, based on truth.
<% end %>

# The End

Find me later:

* IRC: whack on irc.freenode.org and in #puppet
* Twitter: @jordansissel

Community:

* IRC: #puppet on irc.freenode.org
* Mailing list (puppet-users): http://groups.google.com/group/puppet-users
* File bugs: http://projects.puppetlabs.com/issues/
