# Health Probes in Ambient Mesh

Kubernetes health check probes present a problem and create a special case for Kubernetes traffic policy in general. They originate from the kubelet running as a process on the node, and not some other pod in the cluster. They are plaintext and unsecured. Neither the kubelet or the Kubernetes node typically have their own cryptographic identity, so access control isn’t possible. It’s not enough to simply allow all traffic through on the health probe port, as malicious traffic could use that port just as easily as the kubelet could. In addition, many apps use the same port for health probes and legitimate application traffic, so simple port-based allows are unacceptable.

Various CNI implementations solve this in different ways and seek to either work around the problem by silently excluding kubelet health probes from normal policy enforcement, or configuring policy exceptions for them.

## How it works with Ambient Mesh?

In ambient mesh, this problem is solved by using a combination of iptables rules and source network address translation (SNAT) to rewrite only packets that provably originate from the local node with a fixed link-local IP, so that they can be explicitly ignored by Istio policy enforcement as unsecured health probe traffic. A link-local IP was chosen as the default since they are typically ignored for ingress-egress controls, and by IETF standard are not routable outside of the local subnetwork.

This behavior is transparently enabled when you add pods to the ambient mesh, and by default ambient uses the link-local address `169.254.7.127` to identify and correctly allow kubelet health probe packets.

## Problem still exist in some platforms

- EKS won't allow traffic flow throught the local-link IP when the SGG (security groups for Pods) is in Strict Mode.
- Openshift and OVN-Kubernetes CNI also doesn't allow the local-link to be routed properly.
- Clusters using Cilium's BPF masquerading has issues with Ambient and the use of link-local IPs as well.

For all those cases, we would like to have a routable IP that probes can continue to flow from the kubelet to the pod.

## Change the HOST_PROBE_SNAT_IP rule in istio-cni

To accomplish getting a routable IP in the node due to the rules written by the istio-cni, we will need to change the `HOST_PROBE_SNAT_IP`.

That will require a patch in the current `istio-cni-node` installation of the DaemonSet as follows:

```yaml
spec:
  template:
    spec:
      containers:
      - name: install-cni
        env:
        - name: HOST_PROBE_SNAT_IP
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: status.hostIP
```

To run the patch above:

```bash
kubectl patch ds istio-cni-node -n istio-system --patch-file istio-cni-patch.yaml
```

The above adds a set of iptable rules in `ISTIO_POSTRT` to use the local-node IP Address to route the probes.

If Istio CNI is already installed, all the nodes must be recreated after upgrading to a fixed version of the CNI. The host iptable ordering is not changed when upgrading only on initial creating of the iptable chains.

*NOTE*
> The [fix](https://github.com/istio/istio/pull/56984/files) for above can be found in the 1.27 release to INSERT and not APPEND the rules.

## Verify the addition of the host IP

When the patch is applied, use a debug pod into a node to check the rules applied by the istio-cni-node with `iptables-save`.

In the  `ISTIO_POSTRT` rule, you will find the change of the local-link address `169.254.7.127` to the hostIP of your node.

Before:

```
-A ISTIO_POSTRT -m owner --socket-exists -p tcp -m set --match-set istio-inpod-probes-v4 dst -j SNAT --to-source 169.254.7.127
```

After:

```
-A ISTIO_POSTRT -m owner --socket-exists -p tcp -m set --match-set istio-inpod-probes-v4 dst -j SNAT --to-source 10.0.1.8
```