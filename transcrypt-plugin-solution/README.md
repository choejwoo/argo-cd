# Transcrypt Plugin Solution for ArgoCD

This directory contains the solution for the transcrypt plugin issue in ArgoCD.

## Files

- `transcrypt-plugin-final.yaml` - Final plugin configuration (for Kubernetes)
- `test-transcrypt-application-final.yaml` - Example Application manifest
- `github-issue-response-final.md` - GitHub issue response

## Solution

The issue was caused by missing volume mount configuration. The repo-server and sidecar containers need to share the same volume for the plugin socket directory.

### Key Fix

Add this to your Helm values:

```yaml
repoServer:
  volumeMounts:
    - mountPath: /home/argocd/cmp-server/plugins
      name: plugins
  
  volumes:
    - name: plugins
      emptyDir: {}
```

### Additional Note

ArgoCD adds `ARGOCD_ENV_` prefix to plugin environment variables. Update your plugin config to handle this:

```yaml
init:
  args:
    - |
      PASSWORD="${ARGOCD_ENV_TRANSCRYPT_PASSWORD:-$TRANSCRYPT_PASSWORD}"
      if [ -z "$PASSWORD" ]; then
        exit 1
      fi
      /usr/local/bin/transcrypt -c aes-256-cbc -p "$PASSWORD" -y
```

## Local Testing Setup (goreman)

To test this locally using goreman, follow these steps:

### 1. Install Required Tools

```bash
# Install transcrypt
mkdir -p ~/bin
cd /tmp
git clone https://github.com/elasticdog/transcrypt.git
cp transcrypt/transcrypt ~/bin/
chmod +x ~/bin/transcrypt

# Install kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
mv kustomize ~/bin/
chmod +x ~/bin/kustomize

# Add to PATH
export PATH="$HOME/bin:$PATH"
```

### 2. Configure Plugin

Copy the plugin configuration to the ArgoCD test directory:

```bash
cd /home/${USER}/argo-cd
cp transcrypt-plugin-solution/transcrypt-plugin-final.yaml test/cmp/plugin.yaml
```

**Note**: Update the paths in `plugin.yaml` for your local environment:
- `/home/${USER}/bin/transcrypt` → your transcrypt path
- `/home/${USER}/bin/kustomize` → your kustomize path

### 3. Create Test Repository

```bash
# Create test repo with transcrypt
cd /tmp
mkdir test-transcrypt-repo && cd test-transcrypt-repo
git init
echo "Initial commit" > README.md
git add README.md && git commit -m "Initial commit"

# Setup transcrypt
export TRANSCRYPT_PASSWORD="test-password-123"
export PATH="$HOME/bin:$PATH"
transcrypt -c aes-256-cbc -p "$TRANSCRYPT_PASSWORD" -y

# Create encrypted secret
mkdir -p k8s/base/configs
cat > k8s/base/configs/secrets.secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: test-secret
type: Opaque
stringData:
  password: secret123
EOF

# Create kustomization files
cat > k8s/base/configs/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - secrets.secret.yaml
EOF

mkdir -p k8s/staging
cat > k8s/staging/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../base/configs
EOF

git add . && git commit -m "Add encrypted secrets"
```

### 4. Start ArgoCD with goreman

```bash
cd /home/${USER}/argo-cd
make start-local
```

### 5. Create Application

```bash
kubectl apply -f transcrypt-plugin-solution/test-transcrypt-application-final.yaml
```

Or create via ArgoCD UI:
- Repository URL: `/tmp/test-transcrypt-repo`
- Path: `k8s/staging`
- Plugin: `transcrypt`
- Environment Variables: `TRANSCRYPT_PASSWORD=test-password-123`

### 6. Verify

```bash
# Check application status
kubectl get application test-transcrypt -n default

# Check created resources
kubectl get secret test-secret -n default
```

## Test Results

Tested successfully in local environment (goreman). The volume mount fix should work the same in Kubernetes.

