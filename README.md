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
```

## Ports

`siot_nats_port` is the one port a deployment usually names, since Simple IoT
places its other listeners relative to it: since v0.28 the web UI is one above
it and NATS monitoring two above it, with monitoring and the NATS WebSocket on
localhost. Left `""`, `siot_http_port`, `siot_nats_http_port`, and
`siot_nats_ws_port` are not passed to Simple IoT, so its own defaults apply. Set
`siot_http_port` to move the web UI somewhere else, and tell the reverse proxy
in front of it the same number.

Simple IoT through v0.27 put the web UI on 8118, NATS monitoring on 8222, and
the NATS WebSocket on 9222, which is what this role used to pass explicitly, so
a deployment on one of those releases lands where it always did. On v0.28 the
settings behind `siot_nats_http_port` and `siot_nats_ws_port` are gone, and
`siot_nats_monitor_port` moves monitoring instead.

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

## Hardening

The service is installed the way `siot install` sets one up, so a deployment
made with this role and one made with the binary's own installer hold the same
ground:

- The auth token is written to `siot_env_file` (`siot.env` in the data
  directory) with mode `0600` and read through `EnvironmentFile=`, rather than
  into the unit file, which every user on the machine can read.
- The data directory, which holds the store, the device key, and the token, is
  created mode `0700`.
- The unit carries `NoNewPrivileges=true` and, with `siot_sandbox` left on, the
  systemd sandboxing directives: the service sees the rest of the system
  read-only and writes only `siot_read_write_paths`, which is the data directory
  by default. `systemd-analyze security siot` shows what the unit still allows.

Two settings cover the cases where a deployment needs more than that.
`siot_read_write_paths` takes further directories, for example `siot_bin_dir`
for a deployment that runs `siot update`. `siot_protect_home` is `read-only`
rather than the installer's `true`, since sharing Caddy's certificates reads
them out of Caddy's home directory and `siot_data_dir` can live under `/home`.

A client that needs hardware -- a serial port, GPIO, IIO, CAN -- is allowed it
in a drop-in:

```sh
sudo systemctl edit siot
```

```ini
[Service]
SupplementaryGroups=dialout
DeviceAllow=/dev/ttyUSB0 rw
```

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

Setting `siot_provisioning_template: true` runs the files through Jinja on the
way to the server, so one can carry a value the playbook holds rather than
repeating it -- the hash of an enrollment token kept in a vault, for example.
The server sees the rendered file, so `siot provision -check` and Simple IoT
itself read the same thing. Leave it false for files that carry Jinja syntax of
their own.

Files apply in lexical order, so the usual `10-`, `20-` prefixes express which
one goes first. Nodes are matched by description, which is what makes applying a
file repeatedly do what applying it once did. Renaming a description, in the
file or in the UI, detaches the two and creates a second node, so give nodes
descriptions that are meant to last.
