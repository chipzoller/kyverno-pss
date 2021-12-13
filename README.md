# About

This repo contains the latest Kubernetes [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) written for Kyverno, which now number 18. They represent a "refresh" of the existing policies stored [here](https://github.com/kyverno/policies/tree/main/pod-security).

The PSS policies are arguably the most important ones to Kyverno and so we need to ensure these, above all else, are extremely polished and accurate.

## Notes

### Ephemeral Containers

Ephemeral containers are on by default in Kubernetes 1.23, but prior to that a feature gate is needed. Many of these folders contain either/or a YAML and JSON manifest of ephemeral containers used to test the policies. See the KEP note [here](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/277-ephemeral-containers/README.md) for technical details. The command was used to test:

```sh
k replace --raw /api/v1/namespaces/default/pods/testpod/ephemeralcontainers -f ./ephemeralcontainer.json
```

This will do a `patch` op on the `<pod_name>/ephemeralcontainers` subresource. A `kubectl debug` command cannot be used to attach an ephemeral container with customized fields.

Kyverno needs to add `pods/ephemeralcontainers` as a subresource to the validatingwebhookconfiguration in order for AdmissionReview requests to hit Kyverno. When testing, each time a policy is deleted and another created, this change must be made each type due to the dynamic configuration of the webhooks.

### Version Support

Kyverno needs the `AnyNotIn` operator to support some of these policies, specifically the baseline/04-disallow-capabilities, baseline/06-disallow-host-ports (commented rule), and restricted/07-disallow-capabilities-strict policies. Also in the baseline/06-disallow-host-ports (commented rule), if/when this is employed by users, it's highly likely they're going to want to set a range of allowed ports rather than a static list. For that, an additional capability in Kyverno that's needed is the range operator (PR [here](https://github.com/kyverno/kyverno/pull/2788)).

### Optional Policies and Rules

There are now some "optional" policies that we need to somehow account for, especially when installed with Helm. Policies like baseline/06-disallow-host-ports have a commented-out rule that is for users who want to support a list of allowed hostPorts rather than just categorically deny everything. The restricted/05-require-non-root-groups is now listed as "optional" on the PSS page so we'd need to agree whether this always gets installed or is behind some sort of switch.

### CLI Support

Because some of these policies use a JMESPath expression to group all ephemeralContainers, initContainers, and containers into a single, flattened array, the Kyverno CLI is not going to be able to test against these policies presently. See [this issue](https://github.com/kyverno/kyverno/issues/2442) for reference.

### Tests

Pending item is to re-write the test resources and greatly expand upon them to include higher-level Pod controller manifests like Deployments and Jobs. We shouldn't just include testing against Pod resources and call it quits.

### Enforce Mode

All these policies are set to `enforce` for ease of testing. They should be changed back to `audit` before being released.

### Additional Annotations

Need to start employing the new annotations that describe the version(s) of Kyverno where these were tested and the same for Kubernetes. These are:

```
kyverno.io/kyverno-version: 1.6.0
kyverno.io/kubernetes-version: "1.22"
```

Cases where the `AnyNotIn` operator is used have been tagged, initially, for 1.6.0 because that's the corresponding milestone. It can obviously be changed. And all these were built and tested on Kubernetes 1.22 hence that value. We need to be careful not to change these values unless _the version(s) being added were actually and successfully tested_. There shouldn't be any guessing on our part as to the operability of these policies on earlier or later versions of either Kyverno or Kubernetes; if it wasn't explicitly tested and verified, it shouldn't be listed.
