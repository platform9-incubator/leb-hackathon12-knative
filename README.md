# leb-hackathon12-knative
This is an experimental integration between Knative Serving and Decco to achieve Scale-to-Zero DU services, which should significantly reduce the costs of running PMKFT management planes.

- Google Slides presentation: https://docs.google.com/presentation/d/1OPSh9shffZpj0iNn82CFH31aJecxjYtYR5zjZdq3mLg/edit?usp=sharing

## Technical overview of Proof of Concept

- qbert service
- Ambassador service change (had to make a small change to allow incoming traffic, otherwise got disconnects)
- rename of qbert ingress & service resources to avoid conflict with similar resources managed by Ambassador and Knative.
- had to hack Ambassador to increase request timeout from 3 secs to 60 secs to tolerate pod startup time.
- to work around Knative's required service URL scheme (https://servicename.namespace.lb-fqdn) and reconcile with DU scheme (https://du-fqdn/servicename), and also address TLS Cert and CORS issues, had to insert a new nginx-based proxy to forward to Ambassador-managed ingress endpoint.
- made changes to qbert container to work around Knative pod limitations:
  - all logs must go in /var/log (subdirectories not allowed)
  - no sidecars!
  - no special volume/device privileges (breaks mysqlfs)
