# kagent

The Anthropic API key Secret block is stripped from the rendered manifest
and managed out-of-band as `Secret/kagent-anthropic` in the `kagent`
namespace, key `ANTHROPIC_API_KEY`.

To re-render:

```sh
VERSION=0.9.1
helm template kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
    --version $VERSION \
    --namespace kagent \
    --create-namespace > kagent-crds-stack.yaml

helm template kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
    --version $VERSION \
    --namespace kagent \
    --set providers.default=anthropic \
    --set providers.anthropic.apiKey=PLACEHOLDER \
    --set providers.anthropic.model=claude-sonnet-4-5 \
    --set argo-rollouts-agent.enabled=false \
    --set cilium-debug-agent.enabled=false \
    --set cilium-manager-agent.enabled=false \
    --set cilium-policy-agent.enabled=false \
    --set helm-agent.enabled=false \
    --set istio-agent.enabled=false \
    --set kgateway-agent.enabled=false \
    --set observability-agent.enabled=false \
    --set promql-agent.enabled=false > kagent-stack.yaml

```
