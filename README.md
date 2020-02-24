# leb-hackathon12-knative
This is an experimental integration between Knative Serving and Decco to achieve Scale-to-Zero DU services, which should significantly reduce the costs of running PMKFT management planes.

- Google Slides presentation: https://docs.google.com/presentation/d/1OPSh9shffZpj0iNn82CFH31aJecxjYtYR5zjZdq3mLg/edit?usp=sharing

## Technical overview of Proof of Concept

- Ambassador service: had to make a small change to the `ambassador` service
  resource to allow incoming traffic, otherwise got disconnects. I think the
  necessary change was to eliminate the `externalTrafficPolicy: Local` and
  `healthCheckNodePort: 30996` fields, but not sure why this was necessary
  and this needs to be re-tested and verified.
- Rename of qbert ingress & service resources to avoid conflict with similar resources managed by Ambassador and Knative.
- Had to hack Ambassador to increase request envoy route timeout from 3 secs to 60 secs to
  tolerate pod startup time: https://github.com/datawire/ambassador/blob/master/python/ambassador/envoy/v2/v2route.py#L150
  (changed 3000 to 60000). TODO: find if there's a way to do this via configuration change instead.
  Compare these included files:
  - [ambassador-svc-orig.yaml](ambassador-svc-orig.yaml) (original resource)
  - [ambassador-svc.yaml](ambassador-svc.yaml) (modified version)
- Details on how Ambassador container was hacked without rebuilding it from source:
  - Pull released ambassador image (in this case `quay.io/datawire/aes:1.1.1`)
  - Run temp container and edit in place using vi: `docker run -ti --entrypoint /bin/bash quay.io/datawire/aes:1.1.1`
  - Exit container and note its ID (in this example, `af5d992f6497`)
  - Commit container and tag it with new repo/tag:
    `docker commit -c 'ENTRYPOINT ["bash","/buildroot/ambassador/python/entrypoint.sh"]' af5d992f6497 platform9/ambassador-edge-stack:0.1.3`
- To work around Knative's required service URL scheme 
(https://servicename.namespace.lb-fqdn) and reconcile with DU scheme 
(https://du-fqdn/servicename), and also address TLS Cert and CORS issues, 
had to insert a new nginx-based proxy to forward to Ambassador-managed 
ingress endpoint. See included file: [nginx-qbert.conf](nginx-qbert.conf)
- Made changes to qbert container to work around Knative pod limitations:
  - all logs must go in /var/log (subdirectories not allowed)
  - no sidecars! This required creating new "cleartext" versions of these services:
    `keystone-internal`, `resmgr-internal`, `bbmaster`.
  - no special volume/device privileges (breaks mysqlfs)

