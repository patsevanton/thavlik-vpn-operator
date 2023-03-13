# vpn-operator
Kubernetes operator for VPN sidecars written in pure [rust](https://www.rust-lang.org/).

## Motivation
This operator simplifies the process of hiding your pods behind one or more VPN services. Instead of assigning the same exact VPN sidecar to every pod you want cloaked, you use the included `Provider` and `Mask` Custom Resources to automate credentials distribution across any number of VPN services.

## Installation
1. Apply the Custom Resource Definitions:
```bash
kubectl apply -f crds/
```
2. Install the vpn-operator [helm chart](https://helm.sh/):
```bash
# Create your chart configuration file.
cat <<EOF | echo "$(</dev/stdin)" > values.yaml
# In this example, we're exposing prometheus metrics
# for the controllers but disabling PodMonitor creation.
# This is what you would want to do if your cluster
# has a custom method for scraping the pods' metrics.
# PodMonitor is a Custom Resource from kube-prometheus:
# https://github.com/prometheus-operator/kube-prometheus
prometheus:
  expose: true
  podMonitors: false
EOF

# Install the chart into the `vpn` namespace.
RELEASE_NAME=vpn
CHART_PATH=chart/
helm install \
  --namespace=vpn \
  --create-namespace \
  $RELEASE_NAME \
  $CHART_PATH \
  -f values.yaml
```

## Usage
1. Create a `Provider` resource and credentials `Secret` with your VPN credentials:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-vpn-credentials
  namespace: default
spec:
  stringData:
  # Environment variables for gluetun, or however you
  # choose to connect to your VPN. Set spec.verify.skip=true
  # in the Mask resource to disable verification with gluetun.
    VPN_NAME: "my-vpn-name"
    VPN_USERNAME: "myusername"
    VPN_PASSWORD: "mypassword"
---
apiVersion: vpn.beebs.dev/v1
kind: Provider
metadata:
  name: my-vpn
  namespace: default
  labels:
  # You can optionally specify a label so that Masks
  # have the ability to select this service at the
  # exclusion of others.
    vpn.beebs.dev/provider: my-example-vpn-label
spec:
  # In this example, the contractual terms with the
  # VPN service only allows five devices to be active
  # simultaneously. This field is mandatory. You
  # shouldn't attempt to create an unlimited number
  # of connections. Always set it to sane value for
  # your purposes. 
  maxSlots: 5

  # Corresponds to the above Secret resource.
  secret: my-vpn-credentials

  # The controller will attempt to verify that the
  # VPN credentials are correct and the service works.
  # It will do this by injecting the Secret's data as
  # environment variables into a gluetun container and
  # probing an IP service until it returns something
  # different from the initial/unmasked IP address.
  # Note: all of these fields are optional.
  verify:
    # Set to true to bypass credentials verification.
    skip: false

    # Amount of time that can elapse before verification is failed.
    timeout: 30s

    # You can configure periodic verification here:
    interval: 1h

    # The following enables customization of the verification Pod
    # resource. All of these values are optional, and they are merged
    # onto the default generated.
    overrides:
      pod: # Overrides for the Pod resource.
        metadata:
          labels:
            mylabel: myvalue
        spec:
          # Use case: your verify pod need access to your cluster
          # in order to check if your custom VPN is working
          serviceAccountName: my-service-account
      # Overrides for the containers are specified separately.
      # This way you can omit them from the pod override.
      containers:
      # Overrides for the init Container. This container fetches
      # the unmasked IP address from an external service and writes
      # it to /shared/ip for the other containers.
        init:
          image: curlimages/curl:7.88.1
      # Overrides for the VPN Container. This container connects
      # to the VPN service using environment variables from the
      # Provider's credentials Secret. As all containers in a pod
      # share the same network, it will connect all containers
      # to the VPN.
        vpn:
          image: qmcgaw/gluetun:latest
          imagePullPolicy: Always
      # Overrides for the probe Container. This container is
      # responsible for probing the IP service and exiting with
      # code zero when it differs from the initial IP.
        probe:
          image: curlimages/curl:7.88.1
```

2. Make sure the `Provider` enters the `Active` phase:
```bash
kubectl get provider -Aw
```

3. Create `Mask` resources to reserve slots with the `Provider`:
```yaml
apiVersion: vpn.beebs.dev/v1
kind: Mask
metadata:
  name: my-mask
  namespace: default
spec:
  # You can optionally require the Mask be assigned
  # specific Providers. These value will correspond
  # to Providers' metadata.labels["vpn.beebs.dev/provider"]
  #providers: [my-example-vpn-label]
```

4. Wait for the `Mask`'s phase to be `Active` before using it:
```bash
kubectl get mask -Aw
```

5. The `Mask`'s status object contains a reference to the VPN credentials Secret created for it at `status.provider.secret`. Plug these values into your sidecar containers (e.g. [gluetun](https://github.com/qdm12/gluetun)).

## Scaling
While the controller code is fully capable of concurrent reconciliations, scaling is not as simple as increasing the number of replicas in the deployments. I have ideas for how to scale horizontally, so please open an issue if you encounter problems scaling vertically. This can done by adjusting your `values.yaml` file:
```yaml
controllers:
  masks:
    resources:
      limits:
        memory: 192Mi
        cpu: 400m
  providers:
    resources:
      limits:
        memory: 192Mi
        cpu: 400m
```

## Notes
### Custom Resource Definitions
The CRDs are generated by [kube-rs](https://github.com/kube-rs/kube) and include their comments from the [surrounding code](types/src/types.rs). You can view the field descriptions with `kubectl`:
```bash
kubectl get crd providers.vpn.beebs.dev -o yaml
kubectl get crd masks.vpn.beebs.dev -o yaml
```

### Waiting phase
If no slots are available, the phase of the `Mask` will be `Waiting`. If you are writing a controller that creates `Mask`s for your custom resources, you may consider adding an equivalent `Waiting` phase to your custom resource to show it's waiting on a VPN slot to become available.

### Garbage collection
Your application is responsible for monitoring the status of your `Mask` and killing the pod if the provider is changed or unassigned. Failing to do so may result in creating more connections than afforded by a `Provider`'s `spec.maxSlots`.

Note: Some VPN services like SurfShark reserve the right to ban you for abusing their generous "unlimited devices" policies. In such cases, it's recommended to use a relatively low `spec.maxSlots` for the `Provider`.

### Development
Notes on the operator code itself can be found in [operator/README.md](operator/README.md).

## License
All code in this repository is released under [MIT](LICENSE-MIT) / [Apache 2.0](LICENSE-Apache) dual license, which is extremely permissive. Please open an issue if somehow these terms are insufficient.
