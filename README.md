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

Setting `siot_user: caddy2` runs Simple IoT as the user Caddy runs as, which
gives it read access to the certificates Caddy already obtains, and points the
NATS listener at the certificate for `siot_domain`:

```yaml
siot_user: caddy2
siot_domain: portal.xyz.com
siot_nats_url: nats://{{ siot_domain }}:{{ siot_nats_port }}
```

`siot_nats_url` is how Simple IoT's own clients reach the server, so with TLS in
play it has to be the name on the certificate rather than `localhost`, which is
what the certificate would not match.

Two things follow from sharing the certificate this way. Simple IoT reads the
files when it starts, so it keeps serving the previous certificate after Caddy
renews, until the service restarts; a periodic restart, or a deploy, brings the
new one into use. And `siot_user` becomes part of the deployment's state: the
role settles ownership of the data directory on each run so that a change of
user carries the existing store with it.

A `siot_domain` that Caddy does not serve leaves Simple IoT with no certificate
to read and the NATS listener fails to start, so give it a name that appears in
the Caddyfile.

The alternative is to leave the user alone and reverse proxy the NATS WebSocket
port (`siot_nats_ws_port`) through Caddy under a name of its own, which keeps
TLS in one place and needs no certificate sharing.

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
