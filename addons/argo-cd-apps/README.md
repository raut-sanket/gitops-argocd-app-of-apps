# ArgoCD Apps

Helm Chart for the ArgoCD declarative setup. Applications defined here are automatically added to ArgoCD (After the first manual deployment of this Chart).

## Example Options

```yaml
apps:
  - name: argocd
    namespace: argocd
    # Override branch
    targetRevision: <branch>
    # Enable Autosync
    autosync: true
    # Override default valueFiles
    valueFiles:
        - values.yaml
        - values.override.yaml

    # For plain manifests only, default is Helm
    source:
        directory: true

# Same options as above.
addons: []
```
