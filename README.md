# ansible-role-portainer
=======================
Portainer - the coolest UI for Docker http://portainer.io/

This role installs Portainer using a Docker container and optionally configures endpoints, users, settings, registries, and application stacks.

## Tasks in Role
- Ensure docker-py is present using pip
- Remove existing container [if `remove_existing_container: true`]
- Remove persistent data [if `remove_persistent_data: true`]
- Deploy Portainer container to host [define `persistent_data_path`]
- Configure Admin user password
- Generate authentication token
- Define endpoints [DICT | list]
- Deploy application stacks via the Portainer API [see `portainer_stacks`]
- Configure Portainer settings [Jinja2 template]
- Configure registry [Jinja2 template]

## Requirements

- `curl`
- `docker` (service + python package)

## Role Vars

| name | description | default |
|------|-------------|---------|
| `configure_settings`  | override default Portainer settings with template  | false |
| `configure_registry` | configure a Docker registry for Portainer to use   | false |
| `remove_persistent_data` | remove the persistent data directory on the host | false |
| `remove_existing_container` | remove an existing container named 'portainer' | false |
| `persistent_data_path` | path that will be used to store persistent data | /opt/portainer:/data |
| `auth_method` | use LDAP or standalone [2 for ldap, 1 for standalone] | 1 |
| `registry_type` | 1 (Quay.io), 2 (Azure container registry) or 3 (custom registry) | 3 |
| `version` | Portainer version to use | latest |

*See `defaults/main.yml` for a complete list*

## Deploying Stacks via `portainer_stacks`

You can define Docker stacks to deploy using `portainer_stacks`. Each stack will be deployed using the Portainer API, and supports environment variable injection and optional force-redeployment.

### Example Playbook

```yaml
- hosts: myhosts
  become: true
  vars:
    endpoints:
      - name: local
        url: unix:///var/run/docker.sock
    portainer_stacks:
      - name: http-echo
        compose_src: files/http-echo-compose.yml.j2
        env:
          echo_text: "Hello from Portainer!"
        environment_name: local
        force_redeploy: false
  roles:
    - geerlingguy.docker
    - portainer
