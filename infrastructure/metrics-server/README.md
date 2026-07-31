# metrics-server — one-time setup (not managed by Flux/Git)

## Before applying
You installed this manually earlier via
`kubectl apply -f .../components.yaml` — that Deployment isn't owned by
Helm, and creating a new Helm-managed release with the same name
(`metrics-server`, namespace `kube-system`) risks a real ownership
conflict. Remove the manual install first:
```
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Verify after applying
```
kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server
kubectl top nodes
```
`kubectl top nodes` returning real numbers (not an error) is the actual
end-to-end confirmation — that's the same API `kubectl top pods` and any
HPA in the cluster relies on.

If it's still failing after this, check the actual container args landed
correctly — this is the one place a chart-version field-name mismatch
(`args` vs `defaultArgs`, see the comment in `release.yaml`) would show up:
```
kubectl get deployment metrics-server -n kube-system -o jsonpath='{.spec.template.spec.containers[0].args}'
```
Should show `["--kubelet-insecure-tls"]` or similar. If it's empty, that's
the field name that needs correcting.
