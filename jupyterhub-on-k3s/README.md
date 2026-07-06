# DigitAfrica EDGE-AI BP

---
Deploying notebook on a given K3s cluster. OIDC is enabled for the jupyterhub instance

## What this deploys (high level)

Current implementation supports the following deployments:
* Tier - 1: Jupyterhub on a k3s instance. Once it is instantiated, users can login with their accounts, and deploy their notebooks.

Current code has been tested using a three-node cluster, based on the Raspberry-Pi 5 platform.

---

## Configuration for the BP

Hosts need to be declared in the ```inventories/prod/hosts.ini``` file.

```
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

### Key variables in `inventories/prod/group_vars/all.yml`

| Variable | Default | Description |
|---|---|---|
| `k3s_disable_traefik` | `false` | `false` enables Traefik Ingress. `true` disables it (services fall back to NodePort). |
| `k3s_disable_servicelb` | auto-derived | Automatically set to `true` when Traefik is enabled. Do not set manually. |
| `tier0_expose_mode` | `ingress` | `ingress` routes Tier-0 services through Traefik. `nodeport` exposes them on raw ports (backward compatible). |
| `jupyterhub_public_url` | `http://<host>/jupyter` | **Must match the server IP or DNS name.** Used to generate OIDC callback URLs and the final access URL. |

> **Important:** Change `jupyterhub_public_url` to match your actual server address before deploying. For example:
> ```yaml
> jupyterhub_public_url: "http://10.64.45.176/jupyter"
> ```
> If you use a DNS name instead of an IP, also add the hostname to `tls_domains`.

---

## Networking — Traefik Ingress

All services are exposed through the **Traefik Ingress controller** that ships with k3s. There are no NodePort assignments to manage.

| Service | Path | Notes |
|---|---|---|
| JupyterHub (Tier-0 k3s) | `http://<host>/jupyter/` | Single-node k3s |
| JupyterHub (Tier-1) | `http://<host>/jupyter/` | Multi-node k3s |
| Model cache (Tier-0) | `http://<host>/models/` | Static file server |

Visiting the path without a trailing slash (e.g. `/jupyter`) redirects automatically to `/jupyter/`.


### Falling back to NodePort

If you want to skip Traefik and use plain NodePort (e.g. for debugging or constrained environments), set these in `group_vars/all.yml`:

```yaml
k3s_disable_traefik: true
tier0_expose_mode: "nodeport"
```

Services will then be reachable at `http://<host>:30888` (JupyterHub) and `http://<host>:30080` (model cache).


### Environment Configuration

The jupyter hub is configured via the following variables (default values provided)

General settings
```yaml
digitafrica_namespace: "digitafrica"                            # the namespace where to deploy the jupyter hub
```

Jupiter hub specific parameters
```yaml
# Tier1
tier1:
  # Jupyter hub
  jhub:
    http_nodeport: 30888                                        # nodeport for HTTP access to the hub
    password: "digitafrica"                                     # hub password if OIDC is not used
    # OIDC related
    oidc:
      enabled: true                                             # activate OIDC authentication to the hub
      issuer_url: "https://keycloak:8443/realms/digitafrica"    # OIDC identity provider issuer URL
      client_id: "jupyterhub-client"                            # client ID at the identity provider
      client_secret: "REDACTED"                                 # client secret at the identity provider
      scope:                                                    # OIDC scope required
        - "openid"
        - "profile"
        - "email"
      username_claim: "preferred_username"                      # claim to be used to determine the username
      tls_verify: True                                          # wether or not TLS certificate should be verified
      landing_page: "http://jupyter.digitafrica.eu:30888"       # public URL of the hub page
```

In Tier1, when OIDC is used, it is important for the issuer URL to be reachable from within the jupyter hub pod itself. If the digitafrica user portal is deployed and that the user portal and the hub are deployed in the same namespace (as it is by default), then the user portal is directly reachable via the name `keycloak` and listens on port `8443`.

OIDC authentication requires a redirect URL. This URL corresponds to landing page of the jupyter hub and it must be be reachable by the browser of the users. In this example we reach the landing page via the kubernetes service of the jupyterhub, which is exposed with the node port 30888. The service IP address is reachable with the name `jupyter.digitafrica.eu` in our case. Adapt to your environement.

Depending on the type of the deployment (Tier-0/1) only the respective configs need to be present.

#### Creation of a client in the user portal
If you use the user portal, you must create a client. Connect to the user-portal and add a client to your realm of interest.

* Client type: *OpenID Connect*
* Client ID: `jupyterhub-client`
* Client authentication: On
* Valid redirect URIs: `http://jupyter.digitafrica.eu:30888/hub/oauth_callback`

Assuming that your landing page is `http://jupyter.digitafrica.eu:30888`

Make sure as well that you have users in the realm. 

You will be able to connect to the hub with the users provided by the user portal.

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


# Reference

For related architecture and identity integration design, refer to:

DigitAfrica User Portal:
https://gitlab.inria.fr/digitafrica/blueprints/services/user-portal
