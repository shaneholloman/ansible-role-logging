# logging

[![ansible-lint.yml](https://github.com/linux-system-roles/logging/actions/workflows/ansible-lint.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/ansible-lint.yml) [![ansible-test.yml](https://github.com/linux-system-roles/logging/actions/workflows/ansible-test.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/ansible-test.yml) [![codespell.yml](https://github.com/linux-system-roles/logging/actions/workflows/codespell.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/codespell.yml) [![markdownlint.yml](https://github.com/linux-system-roles/logging/actions/workflows/markdownlint.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/markdownlint.yml) [![qemu-kvm-integration-tests.yml](https://github.com/linux-system-roles/logging/actions/workflows/qemu-kvm-integration-tests.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/qemu-kvm-integration-tests.yml) [![tft.yml](https://github.com/linux-system-roles/logging/actions/workflows/tft.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/tft.yml) [![tft_citest_bad.yml](https://github.com/linux-system-roles/logging/actions/workflows/tft_citest_bad.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/tft_citest_bad.yml) [![woke.yml](https://github.com/linux-system-roles/logging/actions/workflows/woke.yml/badge.svg)](https://github.com/linux-system-roles/logging/actions/workflows/woke.yml)

## Background

Logging role is an abstract layer for provisioning and configuring the logging system. Currently, rsyslog is the only supported provider.

In the nature of logging, there are multiple ways to read logs and multiple ways to output them. For instance, the logging system may read logs from local files, or read them from systemd/journal, or receive them from the other logging system over the network. Then, the logs may be stored in the local files in the /var/log directory, or sent to Elasticsearch, or forwarded to other logging system. The combination between the inputs and the outputs needs to be flexible. For instance, you may want to inputs from journal stored just in the local file, while inputs read from files stored in the local log files as well as forwarded to the other logging system.

To satisfy such requirements, logging role introduced 3 primary variables `logging_inputs`, `logging_outputs`, and `logging_flows`. The inputs are represented in the list of `logging_inputs` dictionary, the outputs are in the list of `logging_outputs` dictionary, and the relationship between them are defined as a list of `logging_flows` dictionary. The details are described in [Logging Configuration Overview](#logging-configuration-overview).

## Requirements

This role is supported on RHEL-7+, CentOS Stream-8+ and Fedora distributions.

### Collection requirements

The role requires the `firewall` role and the `selinux` role from the
`fedora.linux_system_roles` collection, if `logging_manage_firewall`
and `logging_manage_selinux` is set to true, respectively.
(Please see also the variables in the [`Other options`](#other-options) section.)

If the `logging` is a role from the `fedora.linux_system_roles`
collection or from the Fedora RPM package, the requirement is already
satisfied.

The role requires external collections for management of `rpm-ostree` nodes.
These are listed in the `meta/collection-requirements.yml`.  You do not need
them if you do not want to manage `rpm-ostree` systems.

If you need to install additional collections based on the above, please run:

```bash
ansible-galaxy collection install -r meta/collection-requirements.yml
```

## Definitions

* `logging_inputs` - List of logging inputs dictionary to specify input types.
  * `basics` - basic inputs configuring inputs from systemd journal or unix socket.
  * `files` - files inputs configuring inputs from local files.
  * `remote` - remote inputs configuring inputs from the other logging system over network.
  * `ovirt` - ovirt inputs configuring inputs from the oVirt system.
* `logging_outputs` - List of logging outputs dictionary to specify output types.
  * `elasticsearch` - elasticsearch outputs configuring outputs to elasticsearch. It is available only when the input is `ovirt`.
  * `files` - files outputs configuring outputs to the local files.
  * `forwards` - forwards outputs configuring outputs to the other logging system.
  * `remote_files` - remote files outputs configuring outputs from the other logging system to the local files.
* `logging_flows` - List of logging flows dictionary to define relationships between logging inputs and outputs.
* [`Rsyslog`](https://www.rsyslog.com/) - The logging role default log provider used for log processing.
* [`Elasticsearch`](https://www.elastic.co/) - Elasticsearch is a distributed, search and analytic engine for all types of data. One of the supported outputs in the logging role.

## Logging Configuration Overview

Logging role allows to have variables `logging_inputs`, `logging_outputs`, and `logging_flows` with additional options to configure logging system such as `rsyslog`.

Currently, the logging role supports four types of logging inputs: `basics`, `files`, `ovirt`, and `remote`.  And four types of outputs: `elasticsearch`, `files`, `forwards`, and `remote_files`.  To deploy configuration files with these inputs and outputs, specify the inputs as `logging_inputs` and the outputs as `logging_outputs`. To define the flows from inputs to outputs, use `logging_flows`.  The `logging_flows` has three keys `name`, `inputs`, and `outputs`, where `inputs` is a list of `logging_inputs name` values and `outputs` is a list of `logging_outputs name` values.

This is a schematic logging configuration to show log messages from input_nameA are passed to output_name0 and output_name1; log messages from input_nameB are passed only to output_name1.

```yaml
---
- name: a schematic logging configuration
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: input_nameA
        type: input_typeA
      - name: input_nameB
        type: input_typeB
    logging_outputs:
      - name: output_name0
        type: output_type0
      - name: output_name1
        type: output_type1
    logging_flows:
      - name: flow_nameX
        inputs: [input_nameA]
        outputs: [output_name0, output_name1]
      - name: flow_nameY
        inputs: [input_nameB]
        outputs: [output_name1]
```

## Variables

### Logging_inputs options

`logging_inputs`: A list of the following dictionaries to configure inputs.

#### logging_inputs common keys

* `name`: Unique name of the input. Used in the `logging_flows` inputs list and a part of the generated config filename.
* `type`: Type of the input element. Currently, `basics`, `files`, `ovirt`, and `remote` are supported. The `type` is used to specify a task type which corresponds to a directory name in roles/rsyslog/{tasks,vars}/inputs/.
* `state`: State of the configuration file. `present` or `absent`. Default to `present`.

#### logging_inputs basics type

`basics` input supports reading logs from systemd journal or systemd unix socket.

Available options:

* `kernel_message`: Load `imklog` if set to `true`. Default to `false`.
* `use_imuxsock`: Use `imuxsock` instead of `imjournal`. Default to `false`.
* `ratelimit_burst`: Maximum number of messages that can be emitted within ratelimit_interval. Default to `20000` if use_imuxsock is false. Default to `200` if use_imuxsock is true.
* `ratelimit_interval`: Interval to evaluate ratelimit_burst. Default to `600` seconds if use_imuxsock is false. Default to `0` if use_imuxsock is true. 0 indicates ratelimiting is turned off.
* `journal_persist_state_interval`: Journal state is persisted every value messages. Default to `10`. Effective only when use_imuxsock is false.

#### logging_inputs files type

`files` input supports reading logs from the local files.

Available options:

* `input_log_path`: File name to be read by the imfile plugin. The value should
  be full path. Wildcard '\*' is allowed in the path.  No default - this parameter
  is mandatory.
* `facility`: Facility to filter the inputs from the files.
* `severity`: Severity to filter the inputs from the files.
* `startmsg_regex`: The regular expression that matches the start part of a message.
* `endmsg_regex`: The regular expression that matches the last part of a message.
* `reopen_on_truncate`: Tells rsyslog to reopen input file when it was truncated
  (inode unchanged but file size on disk is less than current offset in memory).
  Default is `false`

#### logging_inputs ovirt type

`ovirt` input supports oVirt specific inputs.

Available options:

* `subtype`: ovirt input subtype. Value is one of `engine`, `collectd`, and `vdsm`.
* `ovirt_env_name`: ovirt environment name. Default to `engine`.
* `ovirt_env_uuid`: ovirt uuid. Default to none.

Available options for engine and vdsm:

* `ovirt_elasticsearch_index_prefix`: Index prefix for elasticsearch. Default to `project.ovirt-logs`.
* `ovirt_engine_fqdn`: ovirt engine fqdn. Default to none.
* `ovirt_input_file`: ovirt input file. Default to `/var/log/ovirt-engine/test-engine.log` for `engine`; default to `/var/log/vdsm/vdsm.log` for `vdsm`.
* `ovirt_vds_cluster_name`: vds cluster name. Default to none.

Available options for collectd:

* `ovirt_collectd_port`: collectd port number. Default to `44514`.
* `ovirt_elasticsearch_index_prefix`: Index prefix for elasticsearch. Default to `project.ovirt-metrics`.

#### logging_inputs relp type

`relp` input supports receiving logs from the remote logging system over the network using relp.

Available options:

* `port`: Port number Relp is listening to. Default to `20514`. See also [Port Managed by Firewall and SELinux Role](#port-managed-by-firewall-and-selinux-role).
* `tls`: If true, encrypt the connection with TLS. You must provide key/certificates and triplets {`ca_cert`, `cert`, `private_key`} and/or {`ca_cert_src`, `cert_src`, `private_key_src`}. Default to `true`.
* `ca_cert`: Path to CA cert to configure Relp with tls. Default to `/etc/pki/tls/certs/basename of ca_cert_src`.
* `cert`: Path to cert to configure Relp with tls. Default to `/etc/pki/tls/certs/basename of cert_src`.
* `private_key`: Path to key to configure Relp with tls. Default to `/etc/pki/tls/private/basename of private_key_src`.
* `ca_cert_src`: Local CA cert file path which is copied to the target host. If `ca_cert` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `cert_src`: Local cert file path which is copied to the target host. If `cert` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `private_key_src`: Local key file path which is copied to the target host. If `private_key` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `pki_authmode`: Specifying the authentication mode. `name` or `fingerprint` is accepted. Default to `name`.
* `permitted_clients`: List of hostnames, IP addresses, fingerprints(sha1), and wildcard DNS domains which will be allowed by the `logging` server to connect and send logs over TLS. Default to `['*.{{ logging_domain }}']`

#### logging_inputs remote type

`remote` input supports receiving logs from the remote logging system over the network.

Available options:

* `udp_ports`: List of UDP port numbers to listen. If set, the `remote` input listens on the UDP ports. No defaults. If both `udp_ports` and `tcp_ports` are set in a `remote` input item, `udp_ports` is used and `tcp_ports` is dropped. See also [Port Managed by Firewall and SELinux Role](#port-managed-by-firewall-and-selinux-role).
* `tcp_ports`: List of TCP port numbers to listen. If set, the `remote` input listens on the TCP ports. Default to `[514]`. If both `udp_ports` and `tcp_ports` are set in a `remote` input item, `udp_ports` is used and `tcp_ports` is dropped. If both `udp_ports` and `tcp_ports` are not set in a `remote` input item, `tcp_ports: [514]` is added to the item. See also [Port Managed by Firewall and SELinux Role](#port-managed-by-firewall-and-selinux-role).
* `tls`: Set to `true` to encrypt the connection using the default TLS implementation used by the provider. Default to `false`.
* `pki_authmode`: Specifying the default network driver authentication mode. `x509/name`, `x509/fingerprint`, or `anon` is accepted. Default to `x509/name`.
* `permitted_clients`: List of hostnames, IP addresses, fingerprints(sha1), and wildcard DNS domains which will be allowed by the `logging` server to connect and send logs over TLS. Default to `['*.{{ logging_domain }}']`

**Note:** There are 3 types of items in the remote type - udp, plain tcp and tls tcp. The udp type configured using `udp_ports`; the plain tcp type is configured using `tcp_ports` without `tls` or with `tls: false`; the tls tcp type is configured using `tcp_ports` with `tls: true` at the same time. Please note there might be only one instance of each of the three types. E.g., if there are 2 `udp` type items, it fails to deploy.

```yaml
  # Valid configuration example
  - name: remote_udp
    type: remote
    udp_ports: [514, ...]
  - name: remote_ptcp
    type: remote
    tcp_ports: [514, ...]
  - name: remote_tcp
    type: remote
    tcp_ports: [6514, ...]
    tls: true
    pki_authmode: x509/name
    permitted_clients: ['*.example.com']
```

```yaml
  # Invalid configuration example 1; duplicated udp
  - name: remote_udp0
    type: remote
    udp_ports: [514]
  - name: remote_udp1
    type: remote
    udp_ports: [1514]
```

```yaml
  # Invalid configuration example 2; duplicated tcp
  - name: remote_implicit_tcp
    type: remote
  - name: remote_tcp
    type: remote
    tcp_ports: [1514]
```

### logging_custom_templates

`logging_custom_templates`: A list of custom template definitions, for use with
`logging_outputs` `type` `files` and `type` `forwards`.  You can specify the
template for a particular output to use by setting the `template` field in a
particular `logging_outputs` specification, or by setting the default for all
such outputs to use in `logging_files_template_format` and
`logging_forwards_template_format`.

Specify custom templates like this, in either the legacy format or the new style
format:

```yaml
logging_custom_templates:
  - |
    template(name="tpl1" type="list") {
        constant(value="Syslog MSG is: '")
        property(name="msg")
        constant(value="', ")
        property(name="timereported" dateFormat="rfc3339" caseConversion="lower")
        constant(value="\n")
        }
  - >-
    $template precise,"%syslogpriority%,%syslogfacility%,%timegenerated::fulltime%,%HOSTNAME%,%syslogtag%,%msg%\n"
```

Then use like this:

```yaml
logging_outputs:
  - name: custom_file_output
    type: files
    path: /var/log/custom_file_output.log
    template: tpl1  # override logging_files_template_format if set
```

### Logging_outputs options

`logging_outputs`: A list of following dictionary to configure outputs.

#### logging_outputs common keys

* `name`: Unique name of the output. Used in the `logging_flows` outputs list and a part of the generated config filename.
* `type`: Type of the output element. Currently, `elasticsearch`, `files`, `forwards`, and `remote_files` are supported. The `type` is used to specify a task type which corresponds to a directory name in roles/rsyslog/{tasks,vars}/outputs/.
* `state`: State of the configuration file. `present` or `absent`. Default to `present`.

#### logging_outputs general queue parameters

* `queue`: A dict containing general queue parameters that can be used by all output module types, see the official rsyslog documentation for full list of parameters and their function.

```yaml
logging_outputs:
  - name: files_output
    type: files
    queue:
      size: 100
```

#### logging_outputs general action parameters

* `action`: A dict containing general action parameters, see the official rsyslog documentation for full list of parameters and their function.

```yaml
logging_outputs:
  - name: forwards_output
    type: forwards
    target: your_target_host
    action:
      writeallmarkmessages: "on"
```

#### logging_outputs elasticsearch type

`elasticsearch` output supports sending logs to Elasticsearch. It is available only when the input is `ovirt`. Assuming Elasticsearch is already configured and running.

Available options:

* `server_host`: Host name Elasticsearch is running on. The value is a single host or list of hosts. **Required**.
* `server_port`: Port number Elasticsearch is listening to. Default to `9200`.
* `index_prefix`: Elasticsearch index prefix the particular log will be indexed to. **Required**.
* `input_type`: Specifying the input type. Currently only type `ovirt` is supported. Default to `ovirt`.
* `retryfailures`: Specifying whether retries or not in case of failure. Allowed value is `true` or `false`. Default to `true`.
* `tls`: If true, encrypt the connection with TLS. You must provide key/certificates and triplets {`ca_cert`, `cert`, `private_key`} and/or {`ca_cert_src`, `cert_src`, `private_key_src`}. Default to `true`.
* `use_cert`: [DEPRECATED] If true, encrypt the connection with TLS. You must provide key/certificates and triplets {`ca_cert`, `cert`, `private_key`} and/or {`ca_cert_src`, `cert_src`, `private_key_src`}. Default to `true`. Option `use_cert` is deprecated in favor of `tls` and `use_cert` will be removed in the next minor release.
* `ca_cert`: Path to CA cert for Elasticsearch. Default to `/etc/pki/tls/certs/basename of ca_cert_src`.
* `cert`: Path to cert to connect to Elasticsearch. Default to `/etc/pki/tls/certs/basename of cert_src`.
* `private_key`: Path to key to connect to Elasticsearch. Default to `/etc/pki/tls/private/basename of private_key_src`.
* `ca_cert_src`: Local CA cert file path which is copied to the target host. If `ca_cert` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `cert_src`: Local cert file path which is copied to the target host. If `cert` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `private_key_src`: Local key file path which is copied to the target host. If `private_key` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `uid`: If basic HTTP authentication is deployed, the user name is specified with this key.

logging_elasticsearch_password: If basic HTTP authentication is deployed, the password is specified with this global variable. Please be careful that this `logging_elasticsearch_password` is a global variable to be placed at the same level as `logging_output`, `logging_input`, and `logging_flows` are. Another things to be aware of are this `logging_elasticsearch_password` is shared among all the elasticsearch outputs. That is, the elasticsearch servers should share one password if there are multiple of servers. Plus, the uid and password are configured if both of them are found in the playbook. For instance, if there are multiple elasticsearch outputs and one of them is missing the `uid` key, then the configured output does not have the uid and password.

#### logging_outputs files type

`files` output supports storing logs in the local files usually in /var/log.

Available options:

* `facility`: Facility in selector; default to `*`.
* `severity`: Severity in selector; default to `*`.
* `exclude`: Exclude list used in selector; default to none.
* `property`: Property in property-based filter; no default
* `property_op`: Operation in property-based filter; In case of not `!`, put the `property_op` value in quotes; default to `contains`
* `property_value`: Value in property-based filter; default to `error`
* `path`: Path to the output file.
* File/Directory properties - same as corresponding variables of the Ansible `file` module:
  * `mode` - sets the rsyslog `omfile` module `FileCreateMode` parameter
  * `owner` - sets the rsyslog `omfile` module `fileOwner` or `fileOwnerNum` parameter.  If the value
    is an integer, set `fileOwnerNum`, otherwise, set `fileOwner`.
  * `group` - sets the rsyslog `omfile` module `fileGroup` or `fileGroupNum` parameter.  If the value
    is an integer, set `fileGroupNum`, otherwise, set `fileGroup`.
  * `dir_mode` - sets the rsyslog `omfile` module `DirCreateMode` parameter
  * `dir_owner` - sets the rsyslog `omfile` module `dirOwner` or `dirOwnerNum` parameter.  If the value
    is an integer, set `dirOwnerNum`, otherwise, set `dirOwner`.
  * `dir_group` - sets the rsyslog `omfile` module `dirGroup` or `dirGroupNum` parameter.  If the value
    is an integer, set `dirGroupNum`, otherwise, set `dirGroup`.
* `template`: Template format for the particular files output. Allowed values
  are `traditional`, `syslog`, and `modern`, or one of the templates defined in
  `logging_custom_templates`.  Default to `modern`.

Global options:

`logging_files_template_format`: Set default template for the files output.
Allowed values are `traditional`, `syslog`, and `modern`, or one of the
  templates defined in `logging_custom_templates`.  Default to `modern`.

**Note:** Selector options and property-based filter options are exclusive. If Property-based filter options are defined, selector options will be ignored.

**Note:** Unless the above options are given, these local file outputs are configured.

```bash
  kern.*                                      /dev/console
  *.info;mail.none;authpriv.none;cron.none    /var/log/messages
  authpriv.*                                  /var/log/secure
  mail.*                                      -/var/log/maillog
  cron.*                                      -/var/log/cron
  *.emerg                                     :omusrmsg:*
  uucp,news.crit                              /var/log/spooler
  local7.*
```

#### logging_outputs forwards type

`forwards` output sends logs to the remote logging system over the network.

Available options:

* `facility`: Facility in selector; default to `*`.
* `severity`: Severity in selector; default to `*`.
* `exclude`: Exclude list used in selector; default to none.
* `property`: Property in property-based filter; no default
* `property_op`: Operation in property-based filter; In case of not `!`, put the `property_op` value in quotes; default to `contains`
* `property_value`: Value in property-based filter; default to `error`
* `target`: Target host (fqdn). **Required**.
* `udp_port`: UDP port number. Default to `514`.
* `tcp_port`: TCP port number. Default to `514`.
* `tls`: Set to `true` to encrypt the connection using the default TLS implementation used by the provider. Default to `false`.
* `pki_authmode`: Specifying the default network driver authentication mode. `x509/name`, `x509/fingerprint`, or `anon` is accepted. Default to `x509/name`.
* `permitted_server`: Hostname, IP address, fingerprint(sha1) or wildcard DNS domain of the server which this client will be allowed to connect and send logs over TLS. Default to `*.{{ logging_domain }}`
* `template`: Template format for the particular forwards output. Allowed values
  are `traditional`, `syslog`, and `modern`, or one of the templates defined in
  `logging_custom_templates`.  Default to `modern`.

Global options:

`logging_forwards_template_format`: Set default template for the forwards
output. Allowed values are `traditional`, `syslog`, and `modern`, or one of the
  templates defined in `logging_custom_templates`.  Default to `modern`.

**Note:** Selector options and property-based filter options are exclusive. If Property-based filter options are defined, selector options will be ignored.

#### logging_outputs relp type

`relp` output sends logs to the remote logging system over the network using relp.

Available options:

* `target`: Host name the remote logging system is running on. **Required**.
* `port`: Port number the remote logging system is listening to. Default to `20514`.
* `tls`: If true, encrypt the connection with TLS. You must provide key/certificates and triplets {`ca_cert`, `cert`, `private_key`} and/or {`ca_cert_src`, `cert_src`, `private_key_src`}. Default to `true`.
* `ca_cert`: Path to CA cert to configure Relp with tls. Default to `/etc/pki/tls/certs/basename of ca_cert_src`.
* `cert`: Path to cert to configure Relp with tls. Default to `/etc/pki/tls/certs/basename of cert_src`.
* `private_key`: Path to key to configure Relp with tls. Default to `/etc/pki/tls/private/basename of private_key_src`.
* `ca_cert_src`: Local CA cert file path which is copied to the target host. If `ca_cert` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `cert_src`: Local cert file path which is copied to the target host. If `cert` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `private_key_src`: Local key file path which is copied to the target host. If `private_key` is specified, it is copied to the location. Otherwise, to logging_config_dir.
* `pki_authmode`: Specifying the authentication mode. `name` or `fingerprint` is accepted. Default to `name`.
* `permitted_servers`: List of hostnames, IP addresses, fingerprints(sha1), and wildcard DNS domains which will be allowed by the `logging` client to connect and send logs over TLS. Default to `['*.{{ logging_domain }}']`

#### logging_outputs remote_files type

`remote_files` output stores logs to the local files per remote host and program name originated the logs.

Available options:

* `facility`: Facility in selector; default to `*`.
* `severity`: Severity in selector; default to `*`.
* `exclude`: Exclude list used in selector; default to none.
* `property`: Property in property-based filter; no default
* `property_op`: Operation in property-based filter; In case of not `!`, put the `property_op` value in quotes; default to `contains`
* `property_value`: Value in property-based filter; default to `error`
* `async_writing`: If set to `true`, the files are written asynchronously. Allowed value is `true` or `false`. Default to `false`.
* `client_count`: Count of client logging system supported this rsyslog server. Default to `10`.
* `io_buffer_size`: Buffer size used to write output data. Default to `65536` bytes.
* `remote_log_path`: Full path to store the filtered logs.
                       This is an example to support the per host output log files
                       `/path/to/output/dir/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log`
* `remote_sub_path`: Relative path to logging_system_log_dir to store the filtered logs.

**Note:** Selector options and property-based filter options are exclusive. If Property-based filter options are defined, selector options will be ignored.

**Note:** If both `remote_log_path` and `remote_sub_path` are _not_ specified, the remote_file output configured with the following settings.

```java
  template(
    name="RemoteMessage"
    type="string"
    string="/var/log/remote/msg/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log"
  )
  template(
    name="RemoteHostAuthLog"
    type="string"
    string="/var/log/remote/auth/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log"
  )
  template(
    name="RemoteHostCronLog"
    type="string"
    string="/var/log/remote/cron/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log"
  )
  template(
    name="RemoteHostMailLog"
    type="string"
    string="/var/log/remote/mail/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log"
  )
  ruleset(name="unique_remote_files_output_name") {
    authpriv.*   action(name="remote_authpriv_host_log" type="omfile" DynaFile="RemoteHostAuthLog")
    *.info;mail.none;authpriv.none;cron.none action(name="remote_message" type="omfile" DynaFile="RemoteMessage")
    cron.*       action(name="remote_cron_log" type="omfile" DynaFile="RemoteHostCronLog")
    mail.*       action(name="remote_mail_service_log" type="omfile" DynaFile="RemoteHostMailLog")
  }
```

### Logging_flows options

* `name`: Unique name of the flow.
* `inputs`: A list of inputs, from which processing log messages starts.
* `outputs`: A list of outputs. to which the log messages are sent.

### Security options

These variables are set in the same level of the `logging_inputs`, `logging_output`, and `logging_flows`.

#### logging_pki_files

Specifying either of the paths of the ca_cert, cert, and key on the control host or
the paths of theirs on the managed host or both of them.
When TLS connection is configured, `ca_cert_src` and/or `ca_cert` is required.
To configure the certificate of the logging system, `cert_src` and/or `cert` is required.
To configure the private key of the logging system, `private_key_src` and/or `private_key` is required.

* `ca_cert_src`: location of the ca_cert on the control host; if given, the file is copied to the managed host.
* `cert_src`: location of the cert on the control host; if given, the file is copied to the managed host.
* `private_key_src`: location of the key on the control host; if given, the file is copied to the managed host.
* `ca_cert`: path to be deployed on the managed host; the path is also used in the rsyslog config.
  default to /etc/pki/tls/certs/<ca_cert_src basename>
* `cert`: ditto - default to /etc/pki/tls/certs/<cert_src basename>
* `private_key`: ditto - default to /etc/pki/tls/private/<private_key_src basename>

#### logging_domain

The default DNS domain used to accept remote incoming logs from remote hosts. Default to "{{ ansible_domain if ansible_domain else ansible_hostname }}"

### Server performance optimization options

These variables are set in the same level of the `logging_inputs`, `logging_output`, and `logging_flows`.

* `logging_tcp_threads`: Input thread count listening on the plain tcp (ptcp) port. Default to `1`.
* `logging_udp_threads`: Input thread count listening on the udp port. Default to `1`.
* `logging_udp_system_time_requery`: Every `value` OS system calls, get the system time. Recommend not to set above 10. Default to `2` times.
* `logging_udp_batch_size`: Maximum number of udp messages per OS system call. Recommend not to set above 128. Default to `32`.
* `logging_server_queue_type`: Type of queue. `FixedArray` is available. Default to `LinkedList`.
* `logging_server_queue_size`: Maximum number of messages in the queue. Default to `50000`.
* `logging_server_threads`: Number of worker threads. Default to `logging_tcp_threads` + `logging_udp_threads`.

### Other options

These variables are set in the same level of the `logging_inputs`, `logging_output`, and `logging_flows`.

* `logging_enabled`: When 'true', logging role will deploy specified configuration file set. Default to 'true'.
* `logging_mark`: Mark message periodically by immark, if set to `true`. Default to `false`.
* `logging_mark_interval`: Interval for `logging_mark` in seconds. Default to `3600`.
* `logging_purge_confs`: `true` or `false`. If `logging_purge_confs` is set to
  `true`, remove files in rsyslog.d which do not belong to any rpm packages.
  That includes config files generated by the previous logging role run.   NOTE:
  If `/etc/rsyslog.conf` was modified, and you use `logging_purge_confs: true`,
  and you are not providing any `logging_inputs`, then the `rsyslog` package
  will be uninstalled and reinstalled in order to revert back to the original
  system default configuration.  On `ostree` systems, this does not work, so a minimal
  rsyslog.conf will be used, which is *not* the same as the default rsyslog.conf provided
  by the rpm package.  So please use caution if you use this option on `ostree` systems.
* `logging_system_log_dir`: Directory where the local log output files are placed. Default to `/var/log`.
* `logging_preserve_fqdn`: If set to `true`, rsyslog will use the fully qualified domain name (FQDN, ex. server1.example.com) instead of the short hostname (ex. server1). Default to `false`.
* `logging_manage_firewall`: If set to `true` and ports are found in the logging role
  parameters, configure the firewall for the ports using the firewall role.
  If set to `false`, the `logging role` does not manage the firewall.
  Default to `false`.
  NOTE: `logging_manage_firewall` is limited to _adding_ ports.
  It cannot be used for _removing_ ports.
  If you want to remove ports, you will need to use the firewall system
  role directly.
* `logging_manage_selinux`: If set to `true` and ports are found in the logging role
  parameters, configure the SELinux for the ports using the `selinux` role.
  If set to `false`, the `logging` role does not manage SELinux.
  Default to `false`.
  NOTE: `logging_manage_selinux` is limited to _adding_ policy.
  It cannot be used for _removing_ policy.
  If you want to remove policy, you will need to use the `selinux` role directly.
* `logging_custom_config_files`: A list of files to copy to the remote logging configuration
  directory.  For `rsyslog`, this will be `/etc/rsyslog.d/`.  This assumes the
  default logging configuration will load and process the configuration files in
  that directory.  For example, the default `rsyslog` configuration has a directive
  like `$IncludeConfig /etc/rsyslog.d/*.conf`.  WARNING: The use of
  `rsyslog_custom_config_files` or using `type: custom` is deprecated.
* `logging_certificates`: Information used to generate a private key and certificate.
  Default to `[]`.
  The value of `logging_certificates` is passed on to the `certificate_requests`
  in the `certificate` role and used to create the private key and certificate.
  For the supported parameters of `logging_certificates`,
  see the [`certificate_requests` role documentation section](https://github.com/linux-system-roles/certificate/#certificate_requests).
  Before running the `logging` role with `logging_certificates`, join the managed
  hosts to an IPA domain to make CA available to sign the certificate to be created.

  With this example, a private key `logging_cert.key` is generated in `/etc/pki/tls/private`
  and a certificate `logging_cert.crt` is in `/etc/pki/tls/certs` signed by
  the CA certificate managed by `ipa`.

```yaml
    logging_certificates:
      - name: logging_cert
        dns: ['localhost', 'www.example.com']
        ca: ipa
```

  The created private key and certificate are set with the ca certificate, e.g.,
  in `logging_pki_files` as follows:

```yaml
  logging_pki_files:
    - ca_cert: /etc/ipa/ca.crt
      cert: /etc/pki/tls/certs/logging_cert.crt
      private_key: /etc/pki/tls/private/logging_cert.key
```

  or in the `relp` parameters as follows:

```yaml
   logging_inputs:
     - name: relp_server
       type: relp
       tls: true
       ca_cert: /etc/ipa/ca.crt
       cert: /etc/pki/tls/certs/logging_cert.crt
       private_key: /etc/pki/tls/private/logging_cert.key
       [snip]
```

  NOTE: The `certificate` role, unless using IPA and joining the systems to an
  IPA domain, creates self-signed certificates, so you will need to explicitly
  configure trust, which is not currently supported by the system roles.

### Update and Delete

Due to the nature of ansible idempotency, if you run ansible-playbook multiple times without changing any variables and options, no changes are made from the second time. If some changes are made, only the rsyslog configuration files affected by the changes are recreated. To delete any existing rsyslog input or output config files generated by the previous ansible-playbook run, you need to add "state: absent" to the dictionary to be deleted (in this case, input_nameA and output_name0). And remove the flow dictionary related to the input and output as follows.

```yaml
logging_inputs:
  - name: input_nameA
    type: input_typeA
    state: absent
  - name: input_nameB
    type: input_typeB
logging_outputs:
  - name: output_name0
    type: output_type0
    state: absent
  - name: output_name1
    type: output_type1
logging_flows:
  - name: flow_nameY
    inputs: [input_nameB]
    outputs: [output_name1]
```

If you want to remove all the configuration files previously configured, in addition to setting `state: absent` to each logging_inputs and logging_outputs item, add `logging_enabled: false` to the configuration variables as follows. It will eliminate the global and common configuration files, as well.  Or, use `logging_purge_confs: true` to wipe out all previous configuration and replace it with your given configuration.

```yaml
logging_enabled: false
logging_inputs:
  - name: input_nameA
    type: input_typeA
    state: absent
  - name: input_nameB
    type: input_typeB
    state: absent
logging_outputs:
  - name: output_name0
    type: output_type0
    state: absent
  - name: output_name1
    type: output_type1
    state: absent
logging_flows:
  - name: flow_nameY
    inputs: [input_nameB]
    outputs: [output_name1]
```

## Configuration Examples

### Standalone configuration

Deploying `basics input` reading logs from systemd journal and implicit `files output`
to write to the local files. This also deploys two custom files to the
`/etc/rsyslog.d/` directory.

```yaml
---
- name: Deploying basics input and implicit files output
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_custom_config_files:
      - files/90-my-custom-file.conf
      - files/my-custom-file.rulebase
    logging_inputs:
      - name: system_input
        type: basics
```

The following playbook generates the same logging configuration files.

```yaml
---
- name: Deploying basics input and files output
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_custom_config_files:
      - files/90-my-custom-file.conf
      - files/my-custom-file.rulebase
    logging_inputs:
      - name: system_input
        type: basics
    logging_outputs:
      - name: files_output
        type: files
    logging_flows:
      - name: flow0
        inputs: [system_input]
        outputs: [files_output]
```

Deploying `basics input` reading logs from systemd unix socket and `files output` to write to the local files.

```yaml
---
- name: Deploying basics input using systemd unix socket and files output
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: system_input
        type: basics
        use_imuxsock: true
    logging_outputs:
      - name: files_output
        type: files
    logging_flows:
      - name: flow0
        inputs: [system_input]
        outputs: [files_output]
```

Deploying `basics input` reading logs from systemd journal and `files output` to write to the individually configured local files.
This also shows how to specify ownership/permission for log files/directories created by the logger.

```yaml
---
- name: Deploying basic input and configured files output
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: system_input
        type: basics
    logging_outputs:
      - name: files_output0
        type: files
        severity: info
        exclude:
          - authpriv.none
          - auth.none
          - cron.none
          - mail.none
        path: /var/log/messages
      - name: files_output1
        type: files
        facility: authpriv,auth
        path: /var/log/secure
      - name: files_output2
        type: files
        severity: info
        path: /var/log/myapp/my_app.log
        mode: "0600"
        owner: logowner
        group: loggroup
        dir_mode: "0700"
        dir_owner: logowner
        dir_group: loggroup
    logging_flows:
      - name: flow0
        inputs: [system_input]
        outputs: [files_output0, files_output1]
```

Deploying `files input` reading logs from local files and `files output` to write to the individually configured local files.

```yaml
---
- name: Deploying files input and configured files output
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: files_input0
        type: files
        input_log_path: /var/log/containerA/*.log
      - name: files_input1
        type: files
        input_log_path: /var/log/containerB/*.log
    logging_outputs:
      - name: files_output0
        type: files
        severity: info
        exclude:
          - authpriv.none
          - auth.none
          - cron.none
          - mail.none
        path: /var/log/messages
      - name: files_output1
        type: files
        facility: authpriv,auth
        path: /var/log/secure
    logging_flows:
      - name: flow0
        inputs: [files_input0, files_input1]
        outputs: [files_output0, files_output1]
```

Deploying `files input` reading logs from local files and `files output` to write to the local files based on the property-based filters.

```yaml
---
- name: Deploying files input and configured files output
  hosts: all
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: files_input0
        type: files
        input_log_path: /var/log/containerA/*.log
      - name: files_input1
        type: files
        input_log_path: /var/log/containerB/*.log
    logging_outputs:
      - name: files_output0
        type: files
        property: msg
        property_op: contains
        property_value: error
        path: /var/log/errors.log
      - name: files_output1
        type: files
        property: msg
        property_op: "!contains"
        property_value: error
        path: /var/log/others.log
    logging_flows:
      - name: flow0
        inputs: [files_input0, files_input1]
        outputs: [files_output0, files_output1]
```

### Client configuration

Deploying `basics input` reading logs from systemd journal and `forwards output` to forward the logs to the remote rsyslog.

```yaml
---
- name: Deploying basics input and forwards output
  hosts: clients
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: basic_input
        type: basics
    logging_outputs:
      - name: forward_output0
        type: forwards
        severity: info
        target: your_target_hostname
        udp_port: 514
      - name: forward_output1
        type: forwards
        facility: mail
        target: your_target_hostname
        tcp_port: 514
    logging_flows:
      - name: flows0
        inputs: [basic_input]
        outputs: [forward_output0, forward_output1]
```

Deploying `files input` reading logs from a local file and `forwards output` to forward the logs to the remote rsyslog over tls. Assuming the ca_cert, cert and key files are prepared at the specified paths on the control host. The files are deployed to the default location `/etc/pki/tls/certs/`, `/etc/pki/tls/certs/`, and `/etc/pki/tls/private`, respectively.

```yaml
---
- name: Deploying files input and forwards output with certs
  hosts: clients
  roles:
    - linux-system-roles.logging
  vars:
    logging_pki_files:
      - ca_cert_src: /local/path/to/ca_cert
        cert_src: /local/path/to/cert
        private_key_src: /local/path/to/key
    logging_inputs:
      - name: files_input
        type: files
        input_log_path: /var/log/containers/*.log
    logging_outputs:
      - name: forwards_output
        type: forwards
        target: your_target_host
        tcp_port: your_target_port
        pki_authmode: x509/name
        permitted_server: '*.example.com'
    logging_flows:
      - name: flows0
        inputs: [basic_input]
        outputs: [forwards-severity_and_facility]
```

### Server configuration

Deploying `remote input` reading logs from remote rsyslog and `remote_files output` to write the logs to the local files under the directory named by the remote host name.

```yaml
---
- name: Deploying remote input and remote_files output
  hosts: server
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: remote_udp_input
        type: remote
        udp_ports: [514, 1514]
      - name: remote_tcp_input
        type: remote
        tcp_ports: [514, 1514]
    logging_outputs:
      - name: remote_files_output
        type: remote_files
    logging_flows:
      - name: flow_0
        inputs: [remote_udp_input, remote_tcp_input]
        outputs: [remote_files_output]
```

Deploying `remote input` reading logs from remote rsyslog and `remote_files output` to write the logs to the configured local files with the tls setup supporting 20 clients. Assuming the ca_cert, cert and key files are prepared at the specified paths on the control host. The files are deployed to the default location `/etc/pki/tls/certs/`, `/etc/pki/tls/certs/`, and `/etc/pki/tls/private`, respectively.

```yaml
---
- name: Deploying remote input and remote_files output with certs
  hosts: server
  roles:
    - linux-system-roles.logging
  vars:
    logging_pki_files:
      - ca_cert_src: /local/path/to/ca_cert
        cert_src: /local/path/to/cert
        private_key_src: /local/path/to/key
    logging_inputs:
      - name: remote_tcp_input
        type: remote
        tcp_ports: [6514, 7514]
        permitted_clients: ['*.example.com', '*.test.com']
    logging_outputs:
      - name: remote_files_output0
        type: remote_files
        remote_log_path: /var/log/remote/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log
        async_writing: true
        client_count: 20
        io_buffer_size: 8192
      - name: remote_files_output1
        type: remote_files
        remote_sub_path: others/%FROMHOST%/%PROGRAMNAME:::secpath-replace%.log
    logging_flows:
      - name: flow_0
        inputs: [remote_udp_input, remote_tcp_input]
        outputs: [remote_files_output0, remote_files_output1]
```

### Client configuration with Relp

Deploying `basics input` reading logs from systemd journal and `relp output` to send the logs to the remote rsyslog over relp.

```yaml
---
- name: Deploying basics input and relp output
  hosts: clients
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: basic_input
        type: basics
    logging_outputs:
      - name: relp_client
        type: relp
        target: logging.server.com
        port: 20514
        tls: true
        ca_cert_src: /path/to/ca.pem
        cert_src: /path/to/client-cert.pem
        private_key_src: /path/to/client-key.pem
        pki_authmode: name
        permitted_servers:
          - '*.server.com'
    logging_flows:
      - name: flow
        inputs: [basic_input]
        outputs: [relp_client]
```

### Server configuration with Relp

Deploying `relp input` reading logs from remote rsyslog and `remote_files output` to write the logs to the local files under the directory named by the remote host name.

```yaml
---
- name: Deploying remote input and remote_files output
  hosts: server
  roles:
    - linux-system-roles.logging
  vars:
    logging_inputs:
      - name: relp_server
        type: relp
        port: 20514
        tls: true
        ca_cert_src: /path/to/ca.pem
        cert_src: /path/to/server-cert.pem
        private_key_src: /path/to/server-key.pem
        pki_authmode: name
        permitted_clients:
          - '*.client.com'
          - '*.example.com'
    logging_outputs:
      - name: remote_files_output
        type: remote_files
    logging_flows:
      - name: flow
        inputs: [relp_server]
        outputs: [remote_files_output]
```

### Port Managed by Firewall and SELinux Role

When a port is specified in the logging role configuration,
the firewall role is automatically included and the port
is managed by the firewalld.

The port is then configured by the `selinux` role and given
an appropriate syslog SELinux port type depending upon the
associated TLS value.

You can verify the changes by the following command-line.

For firewall,

```bash
firewall-cmd --list-port
```

For SELinux,

```bash
semanage port --list | grep "syslog"
```

The newly specified port will be added to this default set.

```bash
syslog_tls_port_t     tcp   6514, 10514
syslog_tls_port_t     udp   6514, 10514
syslogd_port_t        tcp   601, 20514
syslogd_port_t        udp   514, 601, 20514
```

## Providers

* Rsyslog

## Tests

tests/README.md - This documentation shows how to execute CI tests in the tests directory as well as how to debug when the test fails.

## rpm-ostree

See README-ostree.md
