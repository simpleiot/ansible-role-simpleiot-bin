# SimpleIoT Ansible Role

## Example use in a playbook

```
    - name: SIOT service
      role: siot
      tags: siot
      vars:
        siot_binary: siot
        siot_domain: iot.xyz.com
        siot_user: caddy2
        siot_influx_pass: <influxdb password>
        siot_twilio_sid: <twilio sid>
        siot_twilio_auth_token: <twilio auth token>
        siot_twilio_from: <twilio phone #>

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
