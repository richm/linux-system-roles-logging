# Rsyslog - Proposal for New Input and Output Role Configuration

## Background

The logging role defines `rsyslog_default` variable to deploy the original, all-in-one rsyslog configuration file rsyslog.conf.
It configures to get inputs from `imjournal` and output to the local files in /var/log.
It was originally added to provide "noop" to the logging role.
Then, it was utilized to support for the local system logs to store into the local files combining with the openshift logging in the ovirt logging.
In the linux-system-roles review process, some critical questions on `rsyslog_default` were raised, such as the original intention - providing "noop" is not the style ansible recommends and no other linux-system-roles do not nor will not have such a default variable.
Considering the inputs, the functionality that `rsyslog_default` offers has to be achieved by the logging role itself instead of copying the hardcoded rsyslog.conf to the target host.

This proposal will discuss about these items:
- how to replace the `rsyslog_default` variable with the logging role input/output role configuration.
- how to support more generic way to connect input roles and output roles.

Please note that this github commit starting this doc implements the first cut of the proposal including:
- input roles - basics input role (`imjournal`, `imuxsock`, `imudp`, `imptcp`) and files input role (`imfile`), and
- output roles - files output role (`omfile`) and forwards output role (`omfwd`).
Leftover tasks, other roles to be changed and eliminating `rsyslog_default`, are discussed in [To Do's](#to-dos).

## Proposals

### Output roles
Each output role would be stored in a configuration file which suffix is a type name.

Files and Forwards output configuration files in /etc/rsyslog.d:
```
  10-output-modules.conf
  30-output.files
  30-output.forwards
```

### Ruleset to switch the output
Using rsyslog rulesets, we prepare all possible output combinations in the ruleset configuration file as follows.
Note: The ruleset name is made from the output types in the alphabetical order.
```
==> 50-default-rulesets.conf <==
# Ansible managed

#
# Rules for the local system logs
#
ruleset(name="files") {
  $IncludeConfig /etc/rsyslog.d/*.files
}
ruleset(name="forwards") {
  $IncludeConfig /etc/rsyslog.d/*.forwards
}
```
In this PR, just files and forwards outputs are supported.  As described in [To Do's](#to-dos), elasticsearch is going to be added next.

### Input roles
Based upon the inventory file that users provide, input config jumps to the ruleset.

The next `imjournal` (basics) example shows log messages read from journald by the `imjournal` plugin are logged in the local files.

**`imjournal` example**
```
==> 10-basics-modules.conf <==
# Ansible managed

#
# System log messages
#
# Read from journald
module(load="imjournal"
       StateFile="/var/lib/rsyslog/imjournal.state"
       RateLimit.Burst="20000"
       RateLimit.Interval="600"
       PersistStateInterval="10")
call files
stop
```

**`imfile` example**
The next `imfile` example shows log messages read from /var/log/inputdirectory/*.log by the `imfile` plugin are logged in the local files as well as forwarded to the remote host.
```
==> 90-imfile-input.conf <==
# Ansible managed

#
# Log messages from log files
#

input(type="imfile" file="/var/log/inputdirectory/*.log" tag="container")
call files
call forwards
stop
```

### Inventory file changes
Here is an example that the inputs from `imjournal` (basics) are logged in the local file;
the inputs from `imfile` (files) are forwarded to the remote host in addition to the local file,

**Proposed style**
```
logging_outputs:
  - name: forward-output
    type: forwards
    rsyslog_forwards_actions:
      - name: to-remote
        protocol: tcp
        target: remote_host_name.remote_domain_name
        port: 514
  - name: file-output
    type: files
logging_inputs:
  - name: basic-input
    type: basics
    rsyslog_use_files_ruleset: true
  - name: file-input
    type: files
    rsyslog_use_files_ruleset: true
    rsyslog_use_forwards_ruleset: true
```
In this PR, I propose to make the logging_inputs (input roles) separate from logging_outputs (output roles) and specify the relationship using boolean parameters rsyslog_use_files_ruleset and/or rsyslog_use_forwards_ruleset.

For the comparison, here is the previous style, in which logging_inputs (input roles) used to belong to logging_outputs (output roles) to specify the relationship between the input and the output roles.

**Previous style**
```
logging_outputs:
  - name: forward-output
    type: forwards
    rsyslog_forwards_actions:
      - name: to-remote
        protocol: tcp
        target: remote_host_name.remote_domain_name
        port: 514
    logging_inputs:
      - name: basic-input
        type: basics
  - name: file-output
    type: files
    logging_inputs:
      - name: file-input
        type: files
```
In the above example, the inputs from imjournal (basic-input) are forwarded to the remote host (forward-output); the inputs from imfile (file-input) are stored in the local file (file-output).

With this style, it is not straightforward to implement one input to send multiple outputs in the logging roles.
Moreover, to configure the input from `imjournal` to send both the remote host and the local file,
the basics input would be placed under both forwards and files outputs as follows.
It could give an impression basic-input0 and basic-input1 could be configured independently, but it is not.
There is only one `imjournal` input and most likely the first one is wiped out.
If possible, we should avoid this type of confusions in the designing phase.

**Previous style example with uncertainty**
```
logging_outputs:
  - name: forward-output
    type: forwards
    rsyslog_forwards_actions:
      - name: to-remote
        protocol: tcp
        target: remote_host_name.remote_domain_name
        port: 514
    logging_inputs:
      - name: basic-input0
        type: basics
        rsyslog_imjournal_persist_state_interval: 1000
        rsyslog_imjournal_ratelimit_interval: 10
        rsyslog_imjournal_ratelimit_burst: 100000
  - name: file-output
    type: files
    logging_inputs:
      - name: basic-input1
        type: basics
        rsyslog_imjournal_persist_state_interval: 3000
        rsyslog_imjournal_ratelimit_interval: 30
        rsyslog_imjournal_ratelimit_burst: 300000
```

### logging_outputs, logging_inputs and rsyslog_use_OUTPUT_ruleset
This is a typical example of the configuration.
In logging_outputs, the to-be-configured output roles are listed.
In logging_inputs, the to-be-configure input roles are listed.
Each input role has `rsyslog_use_OUTPUT_ruleset: true|false` to specify which output the log messages are sent to.
```
logging_outputs:
  - name: forward-output
    type: forwards
    rsyslog_forwards_actions:
      - name: to-remote
        protocol: tcp
        target: remote_host_name.remote_domain_name
        port: 514
  - name: file-output
    type: files
logging_inputs:
  - name: basic-input
    type: basics
    rsyslog_use_files_ruleset: true
  - name: file-input
    type: files
    rsyslog_use_files_ruleset: true
    rsyslog_use_forwards_ruleset: true
```
There are some exceptions.
- The files output is configured, by default. That is, even if there is no input roles that write logs to the local files, the files output config file is deployed. The filters and the actions in the configuration file (30-output.files) are equivalent to the ones in the original rsyslog.conf.
- Boolean parameter rsyslog_use_file_ruleset is true, by default, in the basics input role. For the basics input (imjournal) to skip writing to the local system logs into the local log files, `rsyslog_use_files_ruleset: false` needs to be added, explicitly.

Using the exception rules, this snippet of the inventory file will configure equivalent to the original rsyslog.conf. Note: rsyslog_default is going to be eliminated next.
```
logging_enabled: true
rsyslog_default: false
logging_inputs:
  - name: basic_input
    type: basics
```
## To Do's

### Elasticsearch
There is another output role `elasticsearch`.
If this proposal is approved,
I'm planning to rename the 30-output-elasticsearch.conf to 30-output.elasticsearch and add the following rulesets.
```
ruleset(name="elasticsearch") {
  $IncludeConfig /etc/rsyslog.d/*.elasticsearch
}
```
And introduce rsyslog_use_elasticsearch_ruleset parameter to specify the elasticsearch output as follows.
```
logging_inputs:
  - name: files-input
    type: files
    rsyslog_use_elasticsearch_ruleset: true
```
Extending this idea, when we add a new output, e.g., kafka to the existing 3 output roles, we are adding following ruleset with the boolean rsyslog_uses_kafka_ruleset variable and deploying the output config file 30-output.kafka.
```
ruleset(name="kafka") {
  $IncludeConfig /etc/rsyslog.d/*.kafka
}
```

### Input role ovirt
The input role ovirt is made from two inputs (`imfile`, `imtcp`) and the formatting part.
The target outputs are fixed and no need to be configured by the users in the inventory file. (Please correct if this assumption is wrong.)
Reading the ovirt/defaults/main.yml,
if the condition `{% if collect_ovirt_vdsm_log or collect_ovirt_engine_log %}` or
` $syslogtag startswith 'collectd'` is true, the log message is stored in the elasticsearch only. -- [1]
Otherwise, the logs are stored in the elasticsearch as well as in the local files (by the original rsyslog.conf).
That is, the code would be changed so that:
- If the conditions [1] are satisfied, the logs are to be sent to the elasticsearch output only by `call elasticsearch`.
- Otherwise, `call elasticsearch` and `call files`, by which the logs are sent to the both outputs.
If this change works as expected, we could safely eliminate rsyslog_default and 40-send-targets-only.conf with .send_targets_only that reduces an extra config file.

### Input role viaq + viaq-k8s
The input roles viaq and viaq-k8s are tightly coupled,
where viaq is made from the `imfile` module call and the viaq formatting; viaq-k8s contains openshift specific code such as the location of the logging and code to get the meta data from kubernetes.
Elasticsearch is only an expected output, for now.
So, viaq + viaq-k8s also could skip the output configuration in the inventory file and just do `call elasticsearch`.
But I'm not certain how high the priority of viaq is.

### README.md are to be update.
