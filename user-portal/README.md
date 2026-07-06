# DigitAfrica user-portal

Deploying Keycloak on a given K3s cluster.

## What this deploys (high level)

Current implementation supports the following deployments:
* Tier - 1: Keycloak on a k3s instance. Once it is instantiated, other services can use the instance for identification and authentication.

Current code has been tested using a three-node cluster, based on the AMD64 virtual machines.

## Configuration for the BP

Hosts need to be declared in the ```inventories/prod/hosts.ini``` file.

```ini
### TIER 1 NODES ###
[tier1_server]
digitafrica-edge-node1 ansible_host=10.64.45.176 ansible_ssh_pass=REDACTED ansible_become_pass=REDACTED

[tier1_agents]
digitafrica-edge-node2 ansible_host=10.64.45.179 ansible_ssh_pass=REDACTED ansible_become_pass=REDACTED
digitafrica-edge-node3 ansible_host=10.64.45.175 ansible_ssh_pass=REDACTED ansible_become_pass=REDACTED

[tier1:children]
tier1_server
tier1_agents

[all:vars]
ansible_user=ubuntu
ansible_become=true
#ansible_ssh_common_args='-o ProxyJump=proxy@bastion1.theblueprintfactory.org'
```

### Environment Configuration

The user portal is configured via the followin variables (default values provided)

General settings
```yaml
digitafrica_namespace: "digitafrica"  # the namespace where to deploy the user portal
```

User portal specific parameters
```yaml
# Tier1
tier1:
  # User-portal
  user_portal:
    http_nodeport: 30080              # nodeport for HTTP access to the user portal
    https_nodeport: 30443             # nodeport for HTTPS access to the user portal
    admin_user: "admin"               # Administrator username
    admin_password: "admin"           # Administrator password 
    postgres_db: keycloak             # PostgreSQL database name
    postgres_user: keycloak           # PostgreSQL user name
    postgres_password: keycloakpass   # PostgreSQL password
```

TLS is required to ensure secure usage of the user portal.

If `tls_mode` is set to `selfsigned`, the deployement  will automatically generate
the self-signed certificate used to connect to the user portal. The generated
certificate CN is the very first domain provided in the `tls_domains` list and
the certificate Subject Alternative Name contains all the domaines provided in
this list.

If the mode is `selfsigned`, the certificate is not generated, instead it is
assumed to be already installed on the `tier1_server` servers in the directory
`tls_workdir`. The certificate must be in the file `keycloak.crt` and the key
in the file `keycloak.key`.

```yaml
tls_mode: "selfsigned"                 # or "provided"
tls_domains:                           # list of domains supported by the generated certificate 
  - "user-portal.digitafrica.eu"
tls_workdir: "/opt/digitafrica/tls"    # location where to retrieve TLS key and certificate
tls_selfsigned_days: 3650              # validity duration of the certificate in days
```

Depending on the type of the deployment (Tier-0/1) only the respective configs need to be present.

## Deploying the BP

To install on the nodes declared at the hosts.ini file, ensure that the deploying machine has ```ansible``` and ```ssh-pass``` installed, and ssh access to all the machines.

```bash
ansible-galaxy collection install -r requirements.yml
```

To install the BP, use the following command:

```bash
ansible-playbook -i inventories/prod/hosts.ini playbooks/site.yml
```

You can uninstall the current version of the BP using the following command:

```bash 
ansible-playbook -i inventories/prod/hosts.ini playbooks/site.yml -e digitafrica_uninstall=true
```

# OIDC Authentication Implementation

## Overview

In the Tier-1 deployment, authentication has been implemented using **OpenID Connect (OIDC)**.

The user portal is used to be the Identity Provider (IdP) for DigitAfrica. In this setup, the IdP is **Keycloak**.

## Keycloak admin console

You can reach the Keycloak admin console on port `30443` (`{https_nodeport}`) of
any cluster worker node. The username/password correspond to the one provided in
the varaiables `admin_user`/`admin_password`.


## Using SLICES as Identity Provider (optional)

This setup can use SLICES as an external Identity Provider via Keycloak.

If you want to implement a service with SLICES OAuth, follow the official documentation:  
[https://doc.slices-ri.eu/dev/website_oauth.html](https://doc.slices-ri.eu/dev/website_oauth.html)

To integrate SLICES-RI identity provider, go to **Identity Providers** and then:

1. Add OpenID Connect v1.0 Identity provider
2. Define the alias of your choice, this will determine the redirect URL to be used. It is of the form `https://{user_portal_fqdn}/realms/{realm}/broker/{alias}/endpoint` where `user_portal_fqdn` is the `host:port` to be used to reach your user portal from the outside, `realm` is the realm in which you add the identity provider, and `alias` is the alias you chose for this provider.
3. Use a discovery endpoint and specify: `https://portal.slices-ri.eu/.well-known/openid-configuration`
4. Set the client ID and client secret that you received from your request to SLICES (see above).

Once it is created, go and configure the provider and in the advanced pane, and set the scopes to be `openid userinfo projects`.

Then create two mappers:

1. Mapper type=*Attribute Importer*, claim=`first_name`, User attribute Name=`firstName`
2. Mapper type=*Attribute Importer*, claim=`last_name`, User attribute Name=`lastName`

The mappers are required to seamlessly create SLICES users in Keycloak. If not provided, then users will be asked to provide their first and last names.