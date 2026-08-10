# SimpleIoT Ansible Role

## Example use in a playbook

```
    - name: SIOT service
      role: simpleiot-bin
      tags: siot
      vars:
        siot_binary: siot
        siot_auth_token: 816add8d-887b-498a-9558-f2ad85890390
        siot_domain: portal.xyz.com
        siot_http_port: 8080
        siot_user: caddy2
        siot_nats_port: 4222
        siot_nats_url: nats://{{siot_domain}}:{{ siot_nats_port }}
        siot_nats_http_port: 8222
```

## Other recommended services

Simple IoT works will with Caddy2, InfluxDb and Grafana. See the following roles
for installing them:

- https://github.com/cbrake/ansible-role-caddy2
- https://github.com/cbrake/ansible-role-grafana
- https://github.com/cbrake/ansible-role-influxdb

This role is configured to work with the above. Other scenarios will likely
require some customization -- pull requests are welcome.

## Using Caddy2 TLS certs with SimpleIoT

This role can be configured to use caddy2 TLS certs for NATS by setting the user
to `caddy2`.

## Versions

`siot_version` decides what is installed. A release publishes a bare executable
rather than a tarball, so the role downloads the binary under its release name
and installs it to `siot_bin_dir`. Simple IoT can also update itself with
`siot update`, which this role does not use: a self-update would be reverted by
the next playbook run, so bump `siot_version` here instead.

## Provisioning

An instance can be configured from
[provisioning files](https://docs.simpleiot.org/docs/user/configuration.html)
rather than by hand through the UI. Point `siot_provisioning_src` at a directory
in your playbook repo holding the YAML files that describe what the tree should
contain:

```
    - name: SIOT service
      role: simpleiot-bin
      tags: siot
      vars:
        siot_provisioning_src: provisioning/myserver
```

The role copies those files to `siot_provisioning_dir`, removes any file on the
server the repo no longer has, and runs `siot provision -check` so a file that
does not parse fails the play rather than being reported later in the instance's
`provisioning` node. SIOT applies the files at start-up and again whenever one
of them changes, so a configuration change is a commit followed by a playbook
run, with no restart involved.

Files apply in lexical order, so the usual `10-`, `20-` prefixes express which
one goes first. Nodes are matched by description, which is what makes applying a
file repeatedly do what applying it once did. Renaming a description, in the
file or in the UI, detaches the two and creates a second node, so give nodes
descriptions that are meant to last.
