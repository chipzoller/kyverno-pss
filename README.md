# About

**THIS REPO IS DEPRECATED AND WILL BE REMOVED. ALL CONTENT HAS BEEN MERGED INTO THE KYVERNO POLICIES REPO [HERE](https://github.com/kyverno/policies/tree/main/pod-security).**

This repo contains the latest Kubernetes [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) written for Kyverno, which now number 18. They represent a "refresh" of the existing policies stored [here](https://github.com/kyverno/policies/tree/main/pod-security). Many are new and most have either been entirely rewritten or fixed/optimized. In addition to making the policies current in alignment with the PSS, the other large focus of this refresh has been to create extensive tests to cover them.

The PSS policies are arguably the most important ones to Kyverno and so we need to ensure these, above all else, are extremely polished and accurate.

**This repo is a WIP and is subject to periodic change. It should be considered experimental until finalized. Policies found within are recommended NOT to be used in production at this time.**

Table of Contents

- [General Notes](#general-notes)
  - [Ephemeral Containers](#ephemeral-containers)
  - [Version Support](#version-support)
  - [Optional Policies and Rules](#optional-policies-and-rules)
  - [CLI Support](#cli-support)
  - [Tests](#tests)
  - [Enforce Mode](#enforce-mode)
  - [Additional Annotations](#additional-annotations)
  - [Life Expectancy](#life-expectancy)
- [Testing Notes](#testing-notes)
  - [K3d](#k3d)
  - [Kyverno CLI `test` command](#kyverno-cli-test-command)
- [Questions/To-Do](#questionsto-do)
  - [Non-Root Groups](#non-root-groups)
  - [Rename test files](#rename-test-files)
  - [Helm chart alignment](#helm-chart-alignment)

## General Notes

### Ephemeral Containers

Ephemeral containers are on by default in Kubernetes 1.23, but prior to that a feature gate is needed. Many of these folders contain a YAML and/or JSON manifest of ephemeral containers used to test the policies. See the KEP note [here](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/277-ephemeral-containers/README.md) for technical details. The command was used to test:

```sh
kubectl replace --raw /api/v1/namespaces/default/pods/testpod/ephemeralcontainers -f ./ephemeralcontainer.json
```

This will do a `patch` op on the `testpod/ephemeralcontainers` subresource. `testpod` must first exist. A `kubectl debug` command cannot be used to attach an ephemeral container with customized fields, nor can ephemeral containers be created upon initial pod creation.

Kyverno needs to add `pods/ephemeralcontainers` as a subresource to the `validatingwebhookconfiguration` in order for AdmissionReview requests to hit Kyverno. When testing, each time a policy is deleted and another created, this change must be made to the webhook. EDIT: Support for ephemeral containers was added to Kyverno v1.5.3.

### Version Support

Kyverno needs the `AnyNotIn` and `AnyIn` operators to support some of these policies, specifically the baseline/04-disallow-capabilities, baseline/06-disallow-host-ports (commented rule), restricted/01-restrict-volume-types, and restricted/07-disallow-capabilities-strict policies. Also in the baseline/06-disallow-host-ports (commented rule), if/when this is employed by users, it's highly likely they're going to want to set a range of allowed ports rather than a static list. For that, an additional capability in Kyverno that's needed is the range operator (PR [here](https://github.com/kyverno/kyverno/pull/2788)). Combined together, Kyverno needs these enhancements to fully support all these policies:

- [Support in the CLI for more JMESPath expression strings](https://github.com/kyverno/kyverno/issues/2442) (Closed)
- [Improve/provide translations for nil JMESPath resolution to empty string](https://github.com/kyverno/kyverno/issues/2665) (Closed with workaround; implemented here)
- [Bug fix in the CLI for test coverage of the SELinux rules](https://github.com/kyverno/kyverno/issues/2877) (Closed)
- [anyPattern not working for ephemeralContainers](https://github.com/kyverno/kyverno/issues/2957) (Acknowledged bug in Kubernetes 1.23; fixed in 1.23.3)
- [Support for ephemeral containers out-of-the-box](https://github.com/kyverno/kyverno/issues/2821) (Closed)
- [Support for the `AnyNotIn` and `AnyIn` operators](https://github.com/kyverno/kyverno/issues/1837) (Closed)
- [Support for the range operator on integers](https://github.com/kyverno/kyverno/issues/2734) (Closed)
- [Bug fix in the CLI for tests of large sets of resources](https://github.com/kyverno/kyverno/issues/2878) (Closed; User error)

**Optional**

- [Support mutation of variables in `validate.deny` statements when matching on only Pod](https://github.com/kyverno/kyverno/issues/1933) (Closed)

### Optional Policies and Rules

There are now some "optional" policies that we need to somehow account for, especially when installed with Helm. Policies like baseline/06-disallow-host-ports have a commented-out rule that is for users who want to support a list of allowed hostPorts rather than just categorically deny everything. The restricted/05-require-non-root-groups is now listed as "optional" on the PSS page so we'd need to agree whether this always gets installed or is behind some sort of switch.

**Initial thoughts**: The restricted/05-require-non-root-groups policy should be a default installation as part of the PSS suite (Helm included). The baseline/06-disallow-host-ports control should default to the most restrictive which would be the present policy of "if hostPort is present it must be set to zero." This is rule named `host-ports-none`. This is the one that should be installed with Helm. An alternate version of this policy would be to allow for a range of hostPort defined in a rule the base name of which is `host-port-range`. This would not be installed with Helm but would exist in kyverno/policies and tagged with the annotation `policies.kyverno.io/category: Pod Security Standards (Baseline)` so that it is locatable along with the rest of the PSS policies. In this way, the default stance is more secure, and if users want a more permissable policy, they can find it on either kyverno/policies or the web frontend to that at kyverno.io/policies.

### CLI Support

Because some of these policies use a JMESPath expression to group all ephemeralContainers, initContainers, and containers into a single, flattened array, the Kyverno CLI is not going to be able to test against these policies presently. See [this issue](https://github.com/kyverno/kyverno/issues/2442) for reference. There are also some bugs that need to be addressed. See the list containing them under [Version Support](#version-support).

### Tests

Tests are being given a significant amount of attention to ensure most permutations are covered as well as hitting some of the key high-level Pod controllers with one from each category: Deployment and CronJob.

Three central premises being used to write tests: 1) Pods and at least two other higher-level controllers should be tested; 2) the resources written as tests should be valid in the absence of any Kyverno policies (i.e., the resource is accepted by the Kubernetes API server and not invalid); and 3) enough tests should be written to cover all rules, all portions of each rule (as possible), and most permutations of each rule so as to make a reasonable assumption as to the quality of the policy and not allow obvious avenues for circumvention or exploitation.

### Enforce Mode

All these policies are set to `enforce` for ease of testing. They should be changed back to `audit` before being released.

### Additional Annotations

Need to start employing the new annotations that describe the version(s) of Kyverno where these were tested and the same for Kubernetes. These are:

```sh
kyverno.io/kyverno-version: 1.6.0
kyverno.io/kubernetes-version: "1.22"
```

Cases where the `AnyNotIn` or `AnyIn` operators are used have been tagged, initially, for 1.6.0 because that's the corresponding milestone. It can obviously be changed. And all these were built and tested on Kubernetes 1.22 hence that value. We need to be careful not to change these values unless _the version(s) being added were actually and successfully tested_. There shouldn't be any guessing on our part as to the operability of these policies on earlier or later versions of either Kyverno or Kubernetes; if it wasn't explicitly tested and verified, it shouldn't be listed.

### Life Expectancy

This repo will most likely be deleted when the redesigned PSS policies are merged into kyverno/policies. Fair warning.

## Testing Notes

### K3d

I'm using mostly K3d to rapidly build Kubernetes clusters of different versions, configurations, and Kyverno versions. The reduced resource consumption means you can run more concurrently on fewer resources and the OCI images are far smaller than KinD.

The `EphemeralContainers` feature gate can be enabled with either a CLI flag or a config manifest like that shown below. There is currently no K3s image built for Kubernetes v1.23. This was what I used when also building the Windows hostProcess policy as the API server will reject any windowsOptions fields unless the `WindowsHostProcessContainers` feature gate is enabled. It matters not that the node image is Linux since this is for testing purposes.

```yaml
apiVersion: k3d.io/v1alpha3
kind: Simple
name: win-ephemeral
image: rancher/k3s:v1.22.4-k3s1
options:
  k3s:
    extraArgs:
    - arg: --kube-apiserver-arg=feature-gates=EphemeralContainers=true,WindowsHostProcessContainers=true
      nodeFilters:
        - server:*
    - arg: --kube-scheduler-arg=feature-gates=EphemeralContainers=true
      nodeFilters:
        - server:*
    - arg: --kubelet-arg=feature-gates=EphemeralContainers=true
      nodeFilters:
        - agent:*
```

### Kyverno CLI `test` command

I'm trying to start from the ground up and determine the most reasonably complete suite of tests that can be performed on each policy and have begun to work through these starting with the first Baseline policy. The tentative idea is to test for pass and fail scenarios in three main resource types: Pods, one of Deployment|DaemonSet|StatefulSet|Job, and CronJob as these cover the auto-gen capabilities well. Within each of these, most permutations of denied and allowed Pods should be tested. This approach may seem a bit overkill as it will involve a large number of resources, but it provides very strong coverage for future Kyverno development as all holes are plugged.

## Questions/To-Do

### Non-Root Groups

The "non-root groups" control is listed as optional without an explanation on the [Restricted page](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted). Would like to understand this rationale.

Also, on this same control, the present policy on the `runAsGroup` control, if read correctly, suggests a non-zero value is mandatory which I think represents a change. This has been reflected in the current version of the policy.

Although the control here doesn't list a note similar to those occurring in restricted/03-require-run-as-nonroot and restricted/06-restrict-seccomp-strict, it was assumed that the `.spec.*containers[].` level fields are permitted to be unset if the `.spec.` level field is set and vice versa. The `anyPattern` method was therefore employed for all three of these policies.

From the restricted/03-require-run-as-nonroot control in the [PSS Restricted](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) column:

> The container fields may be undefined/`nil` if the pod-level `spec.securityContext.runAsNonRoot` is set to `true`.

From the restricted/06-restrict-seccomp-strict control in the [PSS Restricted](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted) column:

> The container fields may be undefined/`nil` if the pod-level `spec.securityContext.seccompProfile.type` field is set appropriately. Conversely, the pod-level field may be undefined/`nil` if _all_ container-level fields are set.

### Rename test files

Test manifest files should be renamed to `kyverno-test.yaml` in accordance with https://github.com/kyverno/kyverno/issues/2322.

### Helm chart alignment

Need to align these with the kyverno-policies Helm chart and figure out how to deal with optional and/or alternate policies. An alternate would be in baseline/06-disallow-hostports. Host ports are either denied (set to `0`) or allowed with only a permitted array/range. The commented out rule represents the more permissive alternate rule. An optional policy (although unexplained) would be restricted/05-require-non-root-groups. See [Optional Policies and Rules](#optional-policies-and-rules) above for a solution.
