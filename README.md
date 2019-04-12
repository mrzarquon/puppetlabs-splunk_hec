puppet-splunk_hec
==============

Description
-----------

Puppet collects a wide variety of useful information about the servers it manages. When you have key Puppet data in Splunk, you can save time and make richer analyses. For example you can have Splunk send an alert if Puppet sees unexpected change on a server. To make this possible this is a Puppet report processor designed to send a report summary of useful information to the [Splunk HTTP Endpoint Collector "HEC"](http://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector) service. These summaries are designed to be informative but also not too verbose to make logging a burden to the end user. The summaries are meant to be small but sufficient to determine if a Puppet run was successful on a node, and to include metadata such as code-id, transaction-id, and other details to allow for more detailed actions to be done.

It is best used with the [Puppet Report Viewer](https://splunkbase.splunk.com/app/4413/) Splunk addon that adds sourcetypes to make ingesting this data easier into Splunk. Sourcetypes can also be associated with specifc HEC tokens to make event viewing/processing easier. 

Because Puppet and Bolt excel at taking action, the Report Viewer also adds an actionable alert for Puppet Enterprise users. Using the data from a `puppet:summary` event, the Detailed Report Builder actionable alert will create a new event with the type of `puppet:detailed`. These events contain information such as the node in questions, its facts, the resource_events from the node, and links to relevant reports in the Puppet Enterprise Console.

There are two Tasks included in this module, `splunk_hec:bolt_apply` and `splunk_hec:bolt_result` that provide similar data for Bolt Plans to submit data to Splunk. Also included are Plans showing example useage of the Tasks.


Requirements
------------

* Puppet or Puppet Enterprise
* Splunk

This was tested on both Puppet Enterprise 2018.1.4 & Puppet 6, using stock gems of yaml, json, net::https

Report Processor Installation & Usage
--------------------


The steps below will help install and troubleshoot the report processor on a single Puppet Master, including manual steps to configure a puppet-server, and to use the included splunk_hec class. Because one is modifying production machines, these steps allow you to validate your settings before deploying the changes live.

1. Install the Puppet Report Viewer Addon in Splunk. This will import the needed sourcetypes that make setting up the HEC easier in the next steps, and also some overview dashboards that make it a lot easier to see if you're sending Puppet run reports into Splunk.

2. Create a Splunk HEC Token or use an existing one that sends to main index and does not have acknowledgement enabled. Follow the steps provided by Splunk's [Getting Data In Guide](http://docs.splunk.com/Documentation/Splunk/latest/Data/UsetheHTTPEventCollector) if you are new to HTTP Endpoint Collectors.

3. Install this Puppet module in the environment that manages your Puppet Servers are using (probably `production`)

4. Run `puppet plugin download` on your Puppet Server to sync the content

5. Create a `/etc/puppetlabs/puppet/splunk_hec.yaml` (see examples directory for one) adding your Splunk Server URL to the collector (usually something like `https://splunk-dev:8088/services/collector`) & Token from step 1
  - You can add 'timeout' as an optional parameter, default value is 1 second for both open and read sessions, so take value x2 for real world use
  - Provide a `pe_console` value that is the hostname for the Puppet Enterprise Console, which Splunk can use to lookup further information if the installation is a multimaster setup (it is best practice to set this if you're anticipating scaling out more masters in the future).

  ```
---
"url" : "https://splunk-dev.testing.local:8088/services/collector"
"token" : "13311780-EC29-4DD0-A796-9F0CDC56F2AD"
```

6. Run `puppet apply -e 'notify { "hello world": }' --reports=splunk_hec` from the Puppet Server, this will load the report processor and test your configuration settings without actually modifying your Puppet Server's running configuration. If you are using the Puppet Report Viewer app in Splunk then you will see the page update with new data. If not, perform a search by the sourcetype you provided with your HEC configuration.

7. If configured properly the Puppet Report Viewer app in Splunk will show 1 node in the Overview tab.

8. Now it is time to roll these settings out to the fleet of to the Puppet Masters in the installation. For Puppet Enterprise users: 
	- Navigate to Classification -> PE Infrastructure -> PE Master
	- Select Configuration
	- Press Refresh to ensure the splunk_hec class is loaded
	- Add new class `splunk_hec`
	- From the `Parameter name` select atleast `url` and `token` and provide the same attributes from the testing configuration file
	- Optionally set `enable_reports` to `true` if there isn't another component managing the servers reports setting, otherwise manually add `splunk_hec` to the settings as described in the manual steps
	- Commit changes and run Puppet. It is best to navigate to the PE Certificate Authority Classification gorup and run Puppet there first, before running Puppet on the remaining machines

### Manual steps:

- Add `splunk_hec` to `/etc/puppetlabs/puppet/puppet.conf` reports line under the master's configuration block

```
[master]
node_terminus = classifier
storeconfigs = true
storeconfigs_backend = puppetdb
reports = puppetdb,splunk_hec
```

- Restart the puppet-server process for it to reload the configuration and the plugin

- Run `puppet agent -t` somewhere, if you are using the suggested name, use `source="http:puppet-report-summary"` in your Splunk search field to show the reports as they arrive


SSL Support
-----------
Configuring SSL support for this report processor and tasks requires that the Splunk HEC service being used has a [properly configured SSL certificate](https://docs.splunk.com/Documentation/Splunk/latest/Security/AboutsecuringyourSplunkconfigurationwithSSL). Once the HEC service has a valid SSL certificate, the CA will need to be made available to the report processor to load. The supported path is to install a copy of the Splunk CA to a directory called `/etc/puppetlabs/puppet/splunk_hec/` and provide the file name to `splunk_hec` class.

One can update the splunk_hec.yaml file with these settings:

```
"ssl_ca" : "splunk_ca.cert"
```

Or create a profile that copies the `splunk_ca.cert` as part of invoking the splunk_hec class:

```
class profile::splunk_hec {
  file { '/etc/puppetlabs/puppet/splunk_hec':
    ensure => directory,
    owner  => 'pe-puppet',
    group  => 'pe-puppet',
    mode   => 0644,
  }
  file { '/etc/puppetlabs/puppet/splunk_hec/splunk_ca.cert':
    ensure => file,
    owner  => 'pe-puppet',
    group  => 'pe-puppet',
    mode   => '0644',
    source => 'puppet:///modules/profile/splunk_hec/splunk_ca.cert',
  }
}
```

Fact Terminus Support
-----------

The `splunk_hec` module provides a fact terminus that will send a configurable set of facts to the same HEC that the report processor is using, however as a `puppet:facts` sourcetype. This populates the Details and Inventory tabs in the Puppet Report Viewer. 

- In the PE Master configuration group, add the parameter setting `facts_terminus` and set it to `splunk_hec`.
- To configure which facts to collect (such as custom facts) add the `collect_facts` parameter in the `splunk_hec` class and modify the array of facts presented. The following facts are collected regardless to ensure the functionality of the Puppet Report Viever:

```
'os'
'memory'
'puppetversion'
'system_uptime'
'load_averages
'ipaddress'
'fqdn'
'trusted'
'producer'
'environment'
```


Tasks
-----

Two tasks are provided for submitting data from a Bolt plan to Splunk. For clarity, we recommend using a different HEC token to distinguish between events from Puppet runs and those generated by Bolt. The Puppet Report Viewer addon includes a puppet:bolt sourcetype to faciltate this. Currently SSL validation for Bolt communications to Splunk is not supported.

`splunk_hec::bolt_apply`: A task that uses the remote task option to submit a Bolt Apply report in a similar format to the puppet:summary. Unlike the summary, this includes the facts from a target because those are available to bolt at execution time and added to the report data before submission to Splunk.

`splunk_hec::bolt_result`: A task that sends the result of a function to Splunk. Since the format is freeform and dependent on the individual function/tasks being called, formatting of the data is best done in the plan itself prior to submitting the result hash to the task. 

To setup, add the splunk_hec endpoint as a remote target in `inventory.yml`:

```
---
nodes:
  - name: splunk_bolt_hec
    config:
      transport: remote
      remote:
        hostname: <hostname>
        token: <token>
        port: 8088
```

See the `plans/` directory for working examples of apply and result usage.




Known Issues
------------
* SSL Validation is under active development and behavior may change
* Automated testing could use work


Author
------
Chris Barker <cbarker@puppet.com>
