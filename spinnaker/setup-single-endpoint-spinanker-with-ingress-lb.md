# Setup single endpoint spinnaker with ingress loadbalancer

tags: #spinnaker #infra #gcp #kubernate #gke

## Why this setup
1. Why single endpoint? We need single endpoint to avoid 3rd party cookie sharing issue on latest chrome browser. When doing Oauth with spinnaker using two endpoints for deck and gate, it will have that issue of cookie is blocked by the browser.
2. Why using a [[loadbalancer]] for single endpoint?
	1. Deck can expose `/gate` endpoint that goes to gate service.
		- references: [spinnaker community discussion](https://community.spinnaker.io/t/spinnaker-authentication-using-iap/1110/6), [code pointer to set the env var](https://github.com/helm/charts/commit/5a95accf98734a2c301ba40045184281f0ddb5b2#diff-1f4ada09744f95decb1cb95f5c0f43bb)
		- basically you can set the `API_HOST` to sth like `http://gate.spinnaker:8084` (within k cluster)
	2. But there is an issue/bug on `Oauth` redirect. It will try to build a wrong url that endup missing "gate" in the path.
		- https://github.com/spinnaker/spinnaker/issues/996
		- https://github.com/spinnaker/spinnaker/issues/1112
	3. Using the loadbalancer as the single endpoint somehow would just work :)

## Infra
- using [[GKE]] on top of [[GCP]]
- everything for spinnaker is in same cluster
- using [[kleat]]

## Goal

1.  using single ingress loadbalancer to act as the endpoint for both `deck` and `gate`.
2. `/api/v1/*` goes to `gate` while others goes to `deck`
3. SSL terminates at loadbalancer


```
[ ingress load balancer ] -------- > [ deck ]
                          |
						  +------- > [ gate ]

```

### Tweaks required for the setup
#### GKE loadbalancer would use `/` as healthcheck endpoint instead of `/health` for gate.

The setup I have to have it working is to patch the `readinessProbe` of the `gate` deployment.

Following is my patch yaml file for the gate. Add this file under `patchesStrategicMerge` inside your `kustomization.yml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gate
spec:
  template:
    spec:
      containers:
        - name: gate
          readinessProbe:
            $patch: replace
            httpGet:
              path: /api/v1/health
              port: traffic-port
```

(The `kustomization-base` [README](https://github.com/spinnaker/kustomization-base#replace-gates-readiness-probe) brings you to use `exec` which would let the L7 loadbalancer not auto pick up the HTTP healthcheck endpoint. Patch with the above file instead)

#### gate service should know all of its endpoints should be prefixed with `/api/v1`

Add the following to your `gate-local.yml` file. This will ensure the gate service know it should prefix with `/api/v1`

```yaml
server:
  servlet:
    context-path: /api/v1
```

Make sure the `gate-local.yml` file is merged to gate-config inside `kustomization.yml`

```yaml
- behavior: merge
  files:
  - kleat/gate.yml
  - local/gate-local.yml # <-- add this line
  name: gate-config
```

## References
- [[armory]] single hostname for deck and gate: https://docs.armory.io/docs/armory-admin/hostname-deck-gate-configure/
- `useExecHealthCheck` [[halyard]] config: https://spinnaker.io/reference/halyard/custom/#useexechealthcheck
- How `useExecHealthCheck` is updating readiness probe in halyard: [code](https://github.com/spinnaker/halyard/pull/1171/files)
- loadbalancer terminated SSL setup (JP): https://tech.buysell-technologies.com/entry/2019/06/11/112647
