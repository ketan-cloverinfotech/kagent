# Kagent Auto-Healing Pods



## kagent managed-healing demo pack

Use this pattern for every use case:

```text
1. Apply broken manifest
2. Confirm issue with kubectl
3. Open kagent UI
4. Ask k8s-agent / my-first-k8s-agent to diagnose
5. Ask it to heal only that namespace/resource
6. Verify with kubectl
```

kagent’s `k8s-agent` is designed for Kubernetes troubleshooting and has read tools like `GetResources`, `DescribeResource`, `GetEvents`, `GetPodLogs`, and write tools like `ApplyManifest`, `PatchResource`, and `DeleteResource`. Its own safety guidance says to start with read-only checks, explain changes before modifying, limit scope, and verify after the change. ([Kagent](https://kagent.dev/agents/k8s-agent))

---

# 0. Create demo namespace

```bash
# Create namespace for all kagent healing demos
kubectl create namespace kagent-heal-demo
```

```bash
# Check namespace
kubectl get ns kagent-heal-demo
```

---

# Use case summary

| # | Issue | Kubernetes default behavior | kagent managed-healing action |
|---:|---|---|---|
| 1 | ImagePullBackOff wrong tag | Keeps retrying | Detect bad image and rollback |
| 2 | ErrImagePull private registry | Keeps retrying | Check imagePullSecret / service account |
| 3 | CrashLoopBackOff bad env/config | Restarts pod only | Read logs and patch missing env |
| 4 | OOMKilled | Restarts container | Suggest or patch memory limit |
| 5 | Wrong liveness probe | Kills pod repeatedly | Patch liveness probe |
| 6 | Wrong readiness probe | Pod stays unready | Patch readiness probe |
| 7 | Pod Pending | Scheduler keeps trying | Check taint/selector/resources/PVC |
| 8 | ContainerCreating | Waits | Check mount/CNI/image issue |
| 9 | Init container failed | Main container never starts | Read init logs and patch init logic |
| 10 | ConfigMap missing | Pod fails | Create/patch ConfigMap |
| 11 | Secret missing | Pod fails | Create/patch Secret reference |

---

# 1. ImagePullBackOff wrong image tag

## What we will create

```text
Good Deployment: nginx:latest
Bad update: nginx:this-tag-does-not-exist-999
Expected issue: ImagePullBackOff
Healing: rollout undo
```

## Create working Deployment first

```bash
# Create a working deployment first so rollback history exists
kubectl create deployment imagepull-demo \
  --image=nginx:latest \
  -n kagent-heal-demo
```

```bash
# Wait until deployment is healthy
kubectl rollout status deployment/imagepull-demo -n kagent-heal-demo
```

## Break it with wrong image

```bash
# Update deployment with wrong image tag
kubectl set image deployment/imagepull-demo \
  nginx=nginx:this-tag-does-not-exist-999 \
  -n kagent-heal-demo
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo
```

Expected:

```text
ImagePullBackOff
```

## kagent healing prompt

Paste this in `k8s-agent`:

```text
In namespace kagent-heal-demo, deployment imagepull-demo is failing.

First diagnose only:
1. Check Deployment status
2. Check ReplicaSets
3. Check Pods
4. Check Events
5. Confirm current image
6. Check rollout history

If the current image is nginx:this-tag-does-not-exist-999 and there is a previous healthy revision, perform rollout undo for deployment/imagepull-demo.

After rollback, verify:
1. Deployment rollout status
2. Pod status
3. Current image

Do not modify any other resource.
```

## Verify

```bash
# Confirm deployment is healthy after rollback
kubectl rollout status deployment/imagepull-demo -n kagent-heal-demo
```

```bash
# Confirm image is back to nginx:latest
kubectl get deployment imagepull-demo -n kagent-heal-demo \
  -o jsonpath='{.spec.template.spec.containers[*].image}{"\n"}'
```

---

# 2. ErrImagePull private registry issue

## Manifest file

```bash
# Create private registry image pull issue
cat > 02-private-registry.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-registry-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-registry-demo
  template:
    metadata:
      labels:
        app: private-registry-demo
    spec:
      imagePullSecrets:
        - name: missing-registry-secret
      containers:
        - name: app
          image: ghcr.io/private-demo/not-allowed:1.0.0
          command: ["sh", "-c", "sleep 3600"]
EOF
```

```bash
# Apply broken private registry deployment
kubectl apply -f 02-private-registry.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=private-registry-demo
```

Expected:

```text
ErrImagePull
ImagePullBackOff
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment private-registry-demo is failing with image pull issue.

Diagnose:
1. Check Deployment YAML
2. Check Pod status
3. Check Pod events
4. Check imagePullSecrets
5. Check ServiceAccount used by the pod
6. Confirm whether secret missing-registry-secret exists

This is a private registry demo. Do not guess real credentials.

If the secret is missing, explain the exact kubectl command format needed to create the imagePullSecret.

Do not create any fake production secret.
Only suggest the fix and verify the root cause.
```

## Example manual fix format

Use this only when you have real registry credentials:

```bash
# Create image pull secret with real registry credentials
kubectl create secret docker-registry missing-registry-secret \
  --docker-server=ghcr.io \
  --docker-username='<USERNAME>' \
  --docker-password='<TOKEN>' \
  --docker-email='<EMAIL>' \
  -n kagent-heal-demo
```

Why not auto-heal directly here?

```text
kagent cannot know your real private registry username/token.
For production, it should only identify the missing/invalid secret and suggest the exact fix.
```

---

# 3. CrashLoopBackOff due to bad env/config

## Manifest file

```bash
# Create CrashLoopBackOff caused by missing DB_HOST env
cat > 03-crashloop-missing-env.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashloop-env-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crashloop-env-demo
  template:
    metadata:
      labels:
        app: crashloop-env-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Starting app..."
              echo "DB_HOST=$DB_HOST"
              if [ -z "$DB_HOST" ]; then
                echo "ERROR: DB_HOST is missing"
                exit 1
              fi
              echo "App started successfully"
              sleep 3600
EOF
```

```bash
# Apply CrashLoopBackOff demo
kubectl apply -f 03-crashloop-missing-env.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=crashloop-env-demo
```

Expected:

```text
CrashLoopBackOff
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment crashloop-env-demo is in CrashLoopBackOff.

Diagnose first:
1. Check pod status
2. Read container logs
3. Check events
4. Check Deployment env configuration

If logs clearly show "DB_HOST is missing", patch deployment crashloop-env-demo by adding this env variable:

DB_HOST=demo-db.kagent-heal-demo.svc.cluster.local

After patching, verify:
1. Rollout status
2. Pod status
3. New pod logs

Only modify deployment crashloop-env-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Add missing DB_HOST env variable
kubectl set env deployment/crashloop-env-demo \
  DB_HOST=demo-db.kagent-heal-demo.svc.cluster.local \
  -n kagent-heal-demo
```

---

# 4. OOMKilled due to low memory limit

## Manifest file

```bash
# Create OOMKilled demo with very low memory limit
cat > 04-oomkilled.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oom-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oom-demo
  template:
    metadata:
      labels:
        app: oom-demo
    spec:
      containers:
        - name: app
          image: python:3.12-alpine
          command:
            - sh
            - -c
            - |
              python - <<'PY'
              data = []
              while True:
                  data.append("x" * 1024 * 1024)
              PY
          resources:
            requests:
              memory: "16Mi"
              cpu: "50m"
            limits:
              memory: "32Mi"
              cpu: "200m"
EOF
```

```bash
# Apply OOMKilled demo
kubectl apply -f 04-oomkilled.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=oom-demo
```

```bash
# Check previous termination reason
kubectl describe pod -n kagent-heal-demo -l app=oom-demo
```

Expected:

```text
Reason: OOMKilled
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment oom-demo is restarting.

Diagnose:
1. Check pod status
2. Check previous container termination reason
3. Check events
4. Check memory request and limit

If the container was OOMKilled and current memory limit is 32Mi, patch deployment oom-demo to:

requests.memory = 128Mi
limits.memory = 256Mi

After patching, verify:
1. Rollout status
2. Pod status
3. Restart count

Only modify deployment oom-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Increase memory request and limit
kubectl set resources deployment/oom-demo \
  -n kagent-heal-demo \
  --requests=memory=128Mi,cpu=50m \
  --limits=memory=256Mi,cpu=200m
```

Production note:

```text
For real apps, do not blindly increase memory.
Check metrics first, then set request/limit based on real usage.
```

---

# 5. Wrong liveness probe

## Manifest file

```bash
# Create liveness probe failure demo
cat > 05-wrong-liveness.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveness-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: liveness-demo
  template:
    metadata:
      labels:
        app: liveness-demo
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          livenessProbe:
            httpGet:
              path: /wrong-health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
EOF
```

```bash
# Apply wrong liveness probe demo
kubectl apply -f 05-wrong-liveness.yaml
```

```bash
# Watch restarts
kubectl get pods -n kagent-heal-demo -l app=liveness-demo -w
```

Expected:

```text
RESTARTS increasing
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment liveness-demo has pod restarts.

Diagnose:
1. Check pod status
2. Check restart count
3. Check pod events
4. Check livenessProbe config
5. Confirm whether nginx serves "/" but probe uses /wrong-health

If livenessProbe path is /wrong-health, patch deployment liveness-demo livenessProbe path to "/".

After patching, verify:
1. Rollout status
2. Pod restart count is not increasing
3. Pod is Running

Only modify deployment liveness-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Patch liveness probe path from /wrong-health to /
kubectl patch deployment liveness-demo -n kagent-heal-demo --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/livenessProbe/httpGet/path",
    "value": "/"
  }
]'
```

---

# 6. Wrong readiness probe

## Manifest file

```bash
# Create readiness probe failure demo
cat > 06-wrong-readiness.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /wrong-ready
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2
EOF
```

```bash
# Apply wrong readiness probe demo
kubectl apply -f 06-wrong-readiness.yaml
```

```bash
# Check pod readiness
kubectl get pods -n kagent-heal-demo -l app=readiness-demo
```

Expected:

```text
READY 0/1
STATUS Running
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment readiness-demo pod is Running but not Ready.

Diagnose:
1. Check pod status
2. Check events
3. Check readinessProbe config
4. Confirm nginx is listening on port 80
5. Confirm readiness path is /wrong-ready

If readinessProbe path is /wrong-ready, patch it to "/".

After patching, verify:
1. Rollout status
2. Pod READY becomes 1/1
3. No new readiness probe failure events

Only modify deployment readiness-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Patch readiness probe path from /wrong-ready to /
kubectl patch deployment readiness-demo -n kagent-heal-demo --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/readinessProbe/httpGet/path",
    "value": "/"
  }
]'
```

---

# 7. Pod stuck Pending

## Manifest file

This creates a pod that cannot schedule because node selector does not match any node.

```bash
# Create Pending pod due to impossible nodeSelector
cat > 07-pending-nodeselector.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pending-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pending-demo
  template:
    metadata:
      labels:
        app: pending-demo
    spec:
      nodeSelector:
        demo-node: does-not-exist
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "sleep 3600"]
EOF
```

```bash
# Apply Pending pod demo
kubectl apply -f 07-pending-nodeselector.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=pending-demo
```

Expected:

```text
Pending
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment pending-demo pod is stuck Pending.

Diagnose:
1. Check pod status
2. Check events
3. Check nodeSelector
4. Check available node labels
5. Check if scheduling failed due to unmatched node selector

If the only problem is nodeSelector demo-node=does-not-exist, patch deployment pending-demo to remove nodeSelector.

After patching, verify:
1. Rollout status
2. Pod status becomes Running
3. Pod is scheduled on a node

Only modify deployment pending-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Remove wrong nodeSelector
kubectl patch deployment pending-demo -n kagent-heal-demo --type='json' -p='[
  {
    "op": "remove",
    "path": "/spec/template/spec/nodeSelector"
  }
]'
```

---

# 8. Pod stuck ContainerCreating

## Manifest file

This creates a volume mount failure because the ConfigMap volume does not exist.

```bash
# Create ContainerCreating demo due to missing ConfigMap volume
cat > 08-containercreating-missing-volume.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: containercreating-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: containercreating-demo
  template:
    metadata:
      labels:
        app: containercreating-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "cat /etc/demo/config.txt && sleep 3600"]
          volumeMounts:
            - name: demo-config
              mountPath: /etc/demo
      volumes:
        - name: demo-config
          configMap:
            name: missing-volume-config
EOF
```

```bash
# Apply ContainerCreating demo
kubectl apply -f 08-containercreating-missing-volume.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=containercreating-demo
```

Expected:

```text
ContainerCreating
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment containercreating-demo pod is stuck in ContainerCreating.

Diagnose:
1. Check pod status
2. Check pod events
3. Check volume mounts
4. Check whether ConfigMap missing-volume-config exists
5. Confirm if the issue is FailedMount due to missing ConfigMap

If ConfigMap missing-volume-config is missing, create it with this key:

config.txt = "demo config loaded successfully"

After creating it, verify:
1. Pod status becomes Running
2. Container logs show "demo config loaded successfully"

Only modify resources related to containercreating-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Create missing ConfigMap required by volume mount
kubectl create configmap missing-volume-config \
  --from-literal=config.txt="demo config loaded successfully" \
  -n kagent-heal-demo
```

---

# 9. Init container failed

## Manifest file

```bash
# Create init container failure demo
cat > 09-init-container-failed.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: init-fail-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: init-fail-demo
  template:
    metadata:
      labels:
        app: init-fail-demo
    spec:
      initContainers:
        - name: check-before-start
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              echo "Init check failed: dependency not ready"
              exit 1
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "echo main app started && sleep 3600"]
EOF
```

```bash
# Apply init container failure demo
kubectl apply -f 09-init-container-failed.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=init-fail-demo
```

Expected:

```text
Init:CrashLoopBackOff
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment init-fail-demo is not starting.

Diagnose:
1. Check pod status
2. Check init container status
3. Read init container logs
4. Check pod events
5. Confirm main container never starts because init container fails

This is a demo. If init container command exits with code 1, patch the init container command to:

sh -c "echo Init check passed && exit 0"

After patching, verify:
1. Rollout status
2. Pod status becomes Running
3. Main container logs show "main app started"

Only modify deployment init-fail-demo in namespace kagent-heal-demo.
```

## Manual fix command

```bash
# Patch init container command to pass
kubectl patch deployment init-fail-demo -n kagent-heal-demo --type='json' -p='[
  {
    "op": "replace",
    "path": "/spec/template/spec/initContainers/0/command",
    "value": ["sh", "-c", "echo Init check passed && exit 0"]
  }
]'
```

---

# 10. ConfigMap missing

## Manifest file

This creates `CreateContainerConfigError` because the app expects a ConfigMap that does not exist.

```bash
# Create missing ConfigMap demo
cat > 10-missing-configmap.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: missing-configmap-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: missing-configmap-demo
  template:
    metadata:
      labels:
        app: missing-configmap-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "echo APP_MODE=$APP_MODE && sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-config-missing
EOF
```

```bash
# Apply missing ConfigMap demo
kubectl apply -f 10-missing-configmap.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=missing-configmap-demo
```

Expected:

```text
CreateContainerConfigError
```

## kagent healing prompt

```text
In namespace kagent-heal-demo, deployment missing-configmap-demo is failing.

Diagnose:
1. Check pod status
2. Check events
3. Check Deployment envFrom configuration
4. Confirm whether ConfigMap app-config-missing exists

If ConfigMap app-config-missing is missing, create it with:

APP_MODE=demo

After creating it, verify:
1. Pod status becomes Running
2. Container logs show APP_MODE=demo

Only create the missing ConfigMap in namespace kagent-heal-demo.
Do not modify other workloads.
```

## Manual fix command

```bash
# Create missing ConfigMap
kubectl create configmap app-config-missing \
  --from-literal=APP_MODE=demo \
  -n kagent-heal-demo
```

---

# 11. Secret missing

## Manifest file

This creates `CreateContainerConfigError` because the app expects a Secret that does not exist.

```bash
# Create missing Secret demo
cat > 11-missing-secret.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: missing-secret-demo
  namespace: kagent-heal-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: missing-secret-demo
  template:
    metadata:
      labels:
        app: missing-secret-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "echo DB_PASSWORD is configured && sleep 3600"]
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret-missing
                  key: DB_PASSWORD
EOF
```

```bash
# Apply missing Secret demo
kubectl apply -f 11-missing-secret.yaml
```

```bash
# Check pod status
kubectl get pods -n kagent-heal-demo -l app=missing-secret-demo
```

Expected:

```text
CreateContainerConfigError
```

## kagent healing prompt

For demo only:

```text
In namespace kagent-heal-demo, deployment missing-secret-demo is failing.

Diagnose:
1. Check pod status
2. Check events
3. Check secretKeyRef in Deployment
4. Confirm whether Secret app-secret-missing exists
5. Confirm required key name is DB_PASSWORD

This is a demo namespace only.

If Secret app-secret-missing is missing, create it with:
DB_PASSWORD=demo-password

After creating it, verify:
1. Pod status becomes Running
2. Deployment is healthy

Only create the missing Secret in namespace kagent-heal-demo.
Do not expose the secret value in final summary.
```

## Manual fix command

```bash
# Create missing Secret for demo only
kubectl create secret generic app-secret-missing \
  --from-literal=DB_PASSWORD=demo-password \
  -n kagent-heal-demo
```

Production note:

```text
In production, kagent should not invent secret values.
It should only say which Secret/key is missing.
A human or external secret manager should provide the real value.
```

---

# Useful check commands for all demos

```bash
# Show all demo pods
kubectl get pods -n kagent-heal-demo
```

```bash
# Show warning events
kubectl get events -n kagent-heal-demo --sort-by=.lastTimestamp
```

```bash
# Show all deployments
kubectl get deploy -n kagent-heal-demo
```

```bash
# Describe one broken pod
kubectl describe pod <POD_NAME> -n kagent-heal-demo
```

```bash
# Check logs of one pod
kubectl logs <POD_NAME> -n kagent-heal-demo
```

```bash
# Check previous logs after restart
kubectl logs <POD_NAME> -n kagent-heal-demo --previous
```

---

# Open kagent UI

You can use the dashboard or port-forward method. The kagent quick start says the UI lets you select an agent such as `my-first-k8s-agent`, chat with it, and inspect tool arguments/results from the conversation. ([Kagent](https://kagent.dev/docs/kagent/getting-started/quickstart))

```bash
# Port-forward kagent UI
kubectl port-forward -n kagent svc/kagent-ui 8080:8080
```

Open:

```text
http://localhost:8080
```

Use one of these agents:

```text
k8s-agent
my-first-k8s-agent
```

---

# Best common kagent prompt format

Use this format for any healing demo:

```text
In namespace kagent-heal-demo, deployment <DEPLOYMENT_NAME> is unhealthy.

Follow this flow:
1. Start with read-only checks.
2. Check Deployment, ReplicaSet, Pod status, Events, and logs where applicable.
3. Identify exact root cause.
4. Explain the safest fix.
5. If the fix is low-risk and limited to this demo Deployment, perform the fix.
6. Do not modify any other namespace or resource.
7. After the fix, verify rollout status and pod status.

Target:
namespace: kagent-heal-demo
deployment: <DEPLOYMENT_NAME>
```

---

# Cleanup after all demos

```bash
# Delete complete demo namespace
kubectl delete namespace kagent-heal-demo
```

---

## My recommended demo order

Run these first because they are easiest to explain:

```text
1. ImagePullBackOff wrong tag
2. CrashLoopBackOff missing env
3. Wrong readiness probe
4. Wrong liveness probe
5. OOMKilled
6. Pending pod
7. ConfigMap missing
8. Secret missing
```

For production, keep this rule:

```text
Auto-heal allowed:
- rollback demo deployment
- patch probe in demo namespace
- increase memory in demo namespace
- create demo ConfigMap

Human approval required:
- private registry credentials
- production secrets
- PVC/PV/storage fixes
- namespace deletion
- security/network policy changes
```

---

**Sources:**

- [GitHub](https://kagent.dev/agents/k8s-agent)



