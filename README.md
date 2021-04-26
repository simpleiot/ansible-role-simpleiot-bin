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
