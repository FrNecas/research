## Current state

As of beginning of 2022 we [use certbot manually](https://github.com/packit/deployment#obtaining-a-lets-encrypt-cert-using-certbot)
every 3 months to generate a new multi-domain wildcard certificate for all our
microservices. The whole process takes like half an hour, but still would be
nice to have it fully automated.

## Copr

[Copr uses certbot](https://pagure.io/fedora-infra/ansible/blob/main/f/roles/copr/certbot/tasks/letsencrypt.yml)
as well, but with automated renewals, so can we do it in a similar way?

Well, I'm not sure. They run certbot and the renewal service & timer
(systemd equivalent of cron job) on the same machine(s) as web server(s).
But we run our httpd/nginx in containers and probably don't want
to have more processes (httpd/nginx + crond) running in each of the
containers. For nginx, it'd also mean a separate image with crond & certbot
because we use official nginx image.
But [some people do that](https://medium.com/rahasak/setup-lets-encrypt-certificate-with-nginx-certbot-and-docker-b13010a12994).

Or maybe running two containers in one pod? Does anyone has experience with that?

With RWO volumes it's also not possible to run certbot+crond in another
pod with the certificate on a shared volume (because the RWO volume
can't be mounted to more containers).

## Other options

Both, copr and us use `certbot certonly`, they just use [`--standalone`](https://eff-certbot.readthedocs.io/en/stable/using.html#standalone)
while we use [`--manual`](https://eff-certbot.readthedocs.io/en/stable/using.html#manual).
But there are more plugins, so can some of them be or any
help?

### Certbot plugins

[There are](https://eff-certbot.readthedocs.io/en/stable/using.html#getting-certificates-and-choosing-plugins)
special plugins for [apache](https://eff-certbot.readthedocs.io/en/stable/using.html#apache)
and [nginx](https://eff-certbot.readthedocs.io/en/stable/using.html#nginx)
which might help a lot with automation, but the problem with running
certbot/crond+httpd/nginx in the same container still persists.

There are also [DNS plugins](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins)
which would probably allow us to create/renew the certificate(s) in a different
container, but there's no plugin for Google Domains, just [Google Cloud DNS](https://certbot-dns-google.readthedocs.io/en/stable/)
which is a different service.

### OCP way

Wait, there must be some Openshift/k8s native way to this.

#### [Openshift-acme](https://github.com/tnozicka/openshift-acme/tree/master/deploy#single-namespace)

is what we used to utilize as a first version of our cert management, but then
we switched to DNS challenge, [removed it](https://github.com/packit/deployment/commit/2d87c7b2c6711271671b54a994202fa5e65b0c4a)
and probably don't want to return.

#### [Cert-manager](https://github.com/jetstack/cert-manager)

is a Kubernetes add-on, which [can be installed in OCP as operator](https://www.redhat.com/sysadmin/cert-manager-operator-openshift)
but requires cluster-admin privileges to do so. After installation, one also
needs to [configure a specific ACME issuer](https://cert-manager.io/docs/configuration/acme)
with solver being either:

- [DNS01](https://cert-manager.io/docs/configuration/acme/dns01/) - again,
  [there's no provider](https://cert-manager.io/docs/configuration/acme/dns01/google/)
  for Google Domains, just Google CloudDNS
- [HTTP01](https://cert-manager.io/docs/configuration/acme/http01/) - looks
  like lots of experimentation

## Output

There's no straightforward way to automate the certificate generation/renewal
completely. All options look like they'd still need a lot of experimentation
and/or some compromises.
