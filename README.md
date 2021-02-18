# external-metrics-multiplexer

[External Metrics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/external-metrics-api.md) is the blessed Kubernetes way of retrieving metrics and exposing them to the Kubernetes API to provide data for autoscaling decisions.

However, by design, _only one_ provider of external metrics can be used at any given time. There is an open proposal to fix this ([kubernetes-sigs/custom-metrics-apiserver#70](https://github.com/kubernetes-sigs/custom-metrics-apiserver/issues/70)), but in the meantime... Please meet the **external-metrics-multiplexer**!

**external-metrics-multiplexer** allows running several External Metrics providers in Kubernetes at once, and reverse-proxies to providers based on metric prefixes.

## Disclaimer

This is a bit of a hackish proof-of-concept, and while it works, it's not particularly secure (see _Design_) or well-tested. If you plan to use this in prod, I'd advise rewriting it to address the security implications, or use a more fully-fledged project like [Keda](https://github.com/kedacore/keda/).

## Design

Essentially, this chart is just a souped-up nginx that:
- Matches metrics to a metric provider based on their prefix
- Optionally, strips the prefix
- Reverse-proxies the metric request to its metric provider

Regarding authentication/authorization: in order to accept HTTPS requests from the APIServer, we generate a self-signed certificates at start-up time and set nginx to use it.

⚠️  This means we use `insecureSkipTLSVerify: true`, which isn't recommended for security.

Then, to talk to the metrics providers, we use a ServiceAccount toke with RBAC permissions to read metrics (see [templates/rbac.yaml](./templates/rbac.yaml)).

⚠️  This hasn't been thoroughly reviewed for security and may expose metrics without proper RBAC!

## Configure & install

Configure the providers you want to use in [values.yaml](./values.yaml), install that Helm chart, and you're be good to go!

Once all installed, you'll be able to use several metrics providers in HPAs, like:

```yaml
---
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-prometheus
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker-prometheus
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: "prometheus-a-metric-called-foo" # Will use "a-metric-called-foo" from the Prometheus provider
        selector:
          matchLabels:
            yeah: "labels-should-work-too"
      target:
        type: AverageValue
        value: 1

---
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: worker-cloudwatch
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: worker-cloudwatch
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metric:
        name: "cloudwatch-bar" # Will use metric "bar" from the Cloudwatch provider
      target:
        type: AverageValue
        value: 1
```

## Supported providers

**external-metrics-multiplexer** has been tested with the following providers:
- [prometheus-adapter](https://github.com/kubernetes-sigs/prometheus-adapter)
- [k8s-cloudwatch-adapter](https://github.com/awslabs/k8s-cloudwatch-adapter)

Other compliant external-metrics providers should work as well.

## Known issues

**external-metrics-multiplexer** currently doesn't work with [Datadog](https://docs.datadoghq.com/agent/cluster_agent/external_metrics/).

As far as I understand, this is due to Datadog-specific optimisations. Datadog lists all the HPAs, then creates one huge Datadog query by concatenating all the various metrics, and sends this to the Datadog API in a single request.

This is all very clever and optimised, but unfortunately it'll break if any query is not datadog-compatible, which is pretty much always the case when using multiple providers.
