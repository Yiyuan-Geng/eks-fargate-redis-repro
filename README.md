# EKS Fargate - DD_TAGS Reproduction (ZD #2544171)

Reproduction environment for Zendesk ticket #2544171: `DD_TAGS` configured via
`datadog.tags` in Helm values are **not propagated** to the admission controller
injected sidecar agent on EKS Fargate.

## Issue Summary

Customer configures `datadog.tags` in their Helm values (via Terraform), expecting
tags like `owner:prog-devops-ce@symplr.com` to appear in the Datadog UI. However,
the admission controller's sidecar injection does not inject `DD_TAGS` or
`DD_EXTRA_TAGS` into the sidecar agent container, so the tags never reach the UI.

## Files

| File | Description |
|---|---|
| `cluster.yaml` | eksctl cluster config with Fargate profile for `fargate` namespace |
| `rbac.yaml` | ClusterRole, ClusterRoleBinding, ServiceAccount for sidecar agent |
| `values.yaml` | Helm values for Datadog chart with admission controller sidecar injection + `datadog.tags` |
| `redis-deployment.yaml` | Simple Redis deployment with `agent.datadoghq.com/sidecar: fargate` label |

## Reproduction Steps

```bash
# 1. Create the EKS Fargate cluster (~15-20 min)
aws-vault exec sso-tse-sandbox-account-admin -- eksctl create cluster -f cluster.yaml

# 2. Create the fargate namespace
aws-vault exec sso-tse-sandbox-account-admin -- kubectl create namespace fargate

# 3. Apply RBAC
aws-vault exec sso-tse-sandbox-account-admin -- kubectl apply -f rbac.yaml

# 4. Create the datadog-secret (replace <YOUR_API_KEY>)
aws-vault exec sso-tse-sandbox-account-admin -- kubectl create secret generic datadog-secret \
  --from-literal api-key=<YOUR_API_KEY> \
  --from-literal token=abcdabcdabcdabcdabcdabcdabcdabcd -n fargate

# 5. Deploy Cluster Agent via Helm (pinned to chart 3.60.0 to match customer)
aws-vault exec sso-tse-sandbox-account-admin -- helm install datadog -f values.yaml datadog/datadog \
  --version 3.60.0 -n fargate

# 6. Wait for cluster agent to be ready, then deploy Redis
aws-vault exec sso-tse-sandbox-account-admin -- kubectl get pods -n fargate
aws-vault exec sso-tse-sandbox-account-admin -- kubectl apply -f redis-deployment.yaml
```

## Verifying the Issue

Once the Redis pod is running with 2/2 containers (redis + datadog-agent-injected),
check the env vars on the injected sidecar:

```bash
aws-vault exec sso-tse-sandbox-account-admin -- kubectl get pod <REDIS_POD> -n fargate \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{range .env[*]}  {.name}={.value}{"\n"}{end}{"\n"}{end}'
```

**Result:** The `datadog-agent-injected` container does NOT have `DD_TAGS` or
`DD_EXTRA_TAGS` set, confirming that `datadog.tags` from Helm values are not
propagated to the injected sidecar.

## Cleanup

```bash
aws-vault exec sso-tse-sandbox-account-admin -- kubectl delete namespace fargate
aws-vault exec sso-tse-sandbox-account-admin -- eksctl delete cluster -f cluster.yaml
```
