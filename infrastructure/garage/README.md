# Garage — one-time setup (not managed by Flux/Git)

**Status: currently unused.** This was built as Velero's backup target;
Velero was removed by request and nothing has replaced it as a consumer.
Left running in case you want it for something later — the steps below
are still accurate if you do, just know there's no reason to run through
them right now.

Worth knowing up front: as of early-mid 2026, MinIO's community edition was
archived by its maintainers (no more releases, no more images) — it's the
reason this uses Garage instead. If you'd already read a MinIO-based guide
elsewhere, that's why this repo doesn't match it.

## 1. Generate the RPC secret and admin token, create the Secret
```
kubectl create namespace garage
kubectl create secret generic garage-secrets \
  -n garage \
  --from-literal=rpc_secret=$(openssl rand -hex 32) \
  --from-literal=admin_token=$(openssl rand -base64 32)
```

## 2. Once the pod is running, bootstrap the cluster layout
Even single-node Garage needs an explicit layout assignment before it'll
accept writes:
```
kubectl exec -n garage deploy/garage -- /garage status
# note the Node ID from the output, then:
kubectl exec -n garage deploy/garage -- \
  /garage layout assign -z homelab -c 90G <node-id>
kubectl exec -n garage deploy/garage -- \
  /garage layout apply --version 1
```

## 3. Create the bucket and an access key for Velero
```
kubectl exec -n garage deploy/garage -- /garage bucket create velero
kubectl exec -n garage deploy/garage -- /garage key create velero-key
kubectl exec -n garage deploy/garage -- \
  /garage bucket allow velero --read --write --owner --key velero-key
```
The key-create output shows a Key ID and Secret Key — you need both for
the next step. They're shown once; if you lose them, create a new key.

## 4. Create the credentials secret Velero reads
Velero expects AWS-format ini content, not separate literals:
```
cat <<EOF > /tmp/garage-credentials
[default]
aws_access_key_id=<key-id-from-step-3>
aws_secret_access_key=<secret-key-from-step-3>
EOF

kubectl create secret generic velero-garage-credentials \
  -n velero \
  --from-file=cloud=/tmp/garage-credentials

rm /tmp/garage-credentials
```

## 5. Verify
```
kubectl exec -n garage deploy/garage -- /garage bucket list
```
Should show `velero`. If the pod won't start at all, check for the
world-readable-secret error first (`kubectl logs -n garage
deploy/garage`) — see the note in `deployment.yaml`.
