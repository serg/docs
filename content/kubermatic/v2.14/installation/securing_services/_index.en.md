+++
title = "Securing System Services"
date = 2018-04-28T12:07:15+02:00
weight = 60

+++

Access to Prometheus, Grafana and all other system services included with Kubermatic Kubernetes Platform (KKP) is secured by running them behind
[Keycloak-Gatekeeper](https://github.com/keycloak/keycloak-gatekeeper) and using [Dex](https://github.com/dexidp/dex)
as the authentication provider.

{{% notice note %}}
It is still possible to access the system services by using `kubectl port-forward` and thereby circumventing the
proxy's authentication entirely.
{{% /notice %}}

Dex can then be configured to use external authentication sources like GitHub's or Google's OAuth endpoint, LDAP or
OpenID Connect. For this to work you have to configure both Dex (the `oauth` Helm chart) and Keycloak-Gatekeeper
(called "IAP", Identity-Aware Proxy) in your Helm `values.yaml`.

### Dex

For each service that is supposed to use Dex as an authentication provider, configure a `client`. The callback URL is
called after authentication has been completed and must point to `https://<domain>/oauth/callback`. Remember that this
will point to Keycloak-Gatekeeper and is therefore independent of the actual underlying application (Gatekeeper will
intercept the requesta and not forward it to the upstream service). Generate a secure random secret for each client,
for example by doing `cat /dev/urandom | tr -dc A-Za-z0-9 | head -c32`.

A sample configuration for Prometheus and Alertmanager could look like this:

```yaml
dex:
  ingress:
    host: kubermatic.example.com

  clients:
   # keep the KKP client for the login to the KKP dashboard
  - id: kubermatic
    # ...

  # new client used for authenticating Prometheus
  - id: prometheus # a unique identifier
    name: Prometheus
    secret: <generate random secret key here>
    RedirectURIs:
    - 'https://prometheus.kubermatic.example.com/oauth/callback'

  # new client used for authenticating Alertmanager
  - id: alertmanager # a unique identifier
    name: Alertmanager
    secret: <generate another random secret key here>
    RedirectURIs:
    - 'https://alertmanager.kubermatic.example.com/oauth/callback'
```

Each service should have its own credentials (i.e. a different `secret` for every client). Re-deploying the `oauth` chart
with Helm will now prepare Dex to act as an authentication provider, but there is no Keycloak-Gatekeeper yet to make use of
this:

```bash
cd kubermatic-installer
helm upgrade --tiller-namespace kubermatic --install --values YOUR_VALUES_YAML_PATH --namespace oauth oauth charts/oauth/
```

### Keycloak-Gatekeeper (IAP)

Now that you have setup Dex, you need to configure Keycloak-Gatekeeper to sit in front of the system services and use it
for authentication. The configuration for this happens in the `iap` Helm chart. For each client that we configured in Dex,
add a `deployment` to the IAP configuration. Use the client's secret (from Dex) as the `client_secret` and generate
another random, secure encryption key to encrypt the client state with (which is then stored as a cookie in the user's
browser).

Extend your `values.yaml` with a section for the IAP:

```yaml
iap:
  deployments:
    prometheus:
      # will be used to create kubernetes Deployment object
      name: prometheus

      ingress:
        host: prometheus.kubermatic.example.com

      # the Kubernetes service and port the IAP should point to
      upstream_service: prometheus.monitoring.svc.cluster.local
      upstream_port: 9090

      # OAuth configuration from Dex
      # client_id is the "id" from the Dex config
      client_id: prometheus
      # client_secret is the "secret" from the Dex config
      client_secret: <copy value from Dex>

      # generate a fresh secret key here
      encryption_key: <generate random secret key here>

      # see https://www.keycloak.org/docs/latest/securing_apps/#configuration-options
      # this example configures that only users belong to a special GitHub
      # organization can access the service behind the proxy (Prometheus in this case)
      config:
        scopes:
        - "groups"
        resources:
        - uri: "/*"
          groups:
          - "mygithuborg:mygroup"

    alertmanager:
      name: alertmanager
      ingress:
        host: alertmanager.kubermatic.example.com
      upstream_service: alertmanager.monitoring.svc.cluster.local
      upstream_port: 9093
      client_id: alertmanager
      client_secret: <copy value from Dex>
      encryption_key: <generate another random secret key here>
      config:
        scopes:
        - "groups"
        resources:
        - uri: "/*"
          groups:
          - "mygithuborg:mygroup"
```

With all this configured, it's now time to install/upgrade the `iap` Helm chart:

```bash
cd kubermatic-installer
helm upgrade --tiller-namespace kubermatic --install --values YOUR_VALUES_YAML_PATH --namespace iap charts/iap/
```

This will create one Ingress per deployment you configured. If all your Ingress hosts are subdomains of your
primary domain, the wildcard DNS record we already set up earlier will be enough. Otherwise you will need to
update your DNS accordingly.

In addition, this will also create a TLS certificate for each IAP deployment. Once you setup the required DNS
records (see next steps) you can check their progress like so:

```bash
watch kubectl -n iap get certificates
#NAME           READY   SECRET             AGE
#prometheus     True    prometheus-tls     1h
#alertmanager   True    alertmanager-tls   1h
```

### DNS Records

To allow incoming traffic and to acquire a TLS certificate, DNS records must be in place. This can be either
a single wildcard entry for all IAP deployments or individual records. Refer to the
[installation instructions]({{< ref "../install_kubermatic#identity-aware-proxy" >}}) for more information on
what records to create.

### Alternative Authentication Provider

It's possible to use a different authentication provider than Dex. Please refer to the
[OIDC provider]({{< ref "../../advanced/oidc_config" >}}) chapter for more information on how to configure
KKP and Keycloak-Proxy accordingly.

### Security Considerations

The IAP does not protect services against access from within the cluster. Sensitive services should therefore
be configured to require further authentication. Grafana, the [monitoring stack]({{< ref "../monitoring_stack" >}})'s
dashboard UI, requires proper authentication by default, for example.
