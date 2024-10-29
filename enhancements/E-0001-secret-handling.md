# E-0001: Secret Handling

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. 

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement Proposal. Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a Enhancement Proposal within the wider community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

- Users deploying a package want to be able to pull package images from private registries.
- Package authors want the ability to declare secrets used within the package.
- Secret references are used to not duplicate sensitive information through PKO APIs.
- Users deploying a package want to be able to pass secret references into the package templating engine.
- Users deploying a cluster-package don't want to worry about copying secrets into newly-created namespaces (and updating/deleting them, too).

### Goals

<!--
List the specific goals of the enhancement proposal.
What is it trying to achieve?
How will we know that this has succeeded?
-->
- Users can deploy package images from private registries.
- Users can provide existing secret references to a package.
- Package maintainer don't have to hard-code secret names.
- Package maintainers can sync referenced-secrets to namespaces created within a ClusterPackage.
- Package maintainers can reference the pull secret of the templated image in case the package deploys another package (which is probably a component in from the same package image).

### Non-Goals

<!--
What is out of scope for this enhancement proposal?
Listing non-goals helps to focus discussion and make progress.
-->
- Providing facilities to automagically inject (pull) secret references into managed objects. (For example PodSpecs or (Cluster)Packages.)
- Syncing secrets across the MC/HC barried in the HyperShift architecture.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

- A new object of kind `SecretSync` will be introduced to the in-cluster API of PKO
  - allowing a specified `Secret` source object to be synced to multiple specified namespace/name combinations
  - implementing status reporting so that it can be used as a building block in automations - for example: PKO phases waiting for successful synchronization before continuing with a package rollout.
  - usage can happen
  -   freely (without using PKO's other API functionalities), maybe solving other problems of users not taken into further consideration in this proposal other than thinking about the performance implications on PKO and the apiserver this might have
  - within packages
- The PackageManifest API has to be changed so that `Secret` references can be specified and aliased. References specify:
  - an alias name to be used with newly introduced resolving functions in the package templates
  - when package is installable cluster-scoped: a namespaced name of the source secret
  - when package is installable namespace-scoped: a name of the source secret which is expected in the Package's target namespace
  - if the reference is optional or not.

- The `(Cluster)Package` API has to be changed to
  - allow the specification of an `ImagePullSecret` for pulling the package image itself from a private registry.
  - allow an end-user to supply a list of secret references and bind them to the alias names described in the `PackageManifest` by the packaging person.


### User Stories

<!--
Detail the things that people will be able to do if this enhancement proposal is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story: User

As a user I want to deploy package images from private registries.

As a user I want to be able to get a list of secrets required for deploying a package from a package image's manifest.

As a user I want to provide existing secret references to a package, declaring several secrets as required. If needed, I want to rely on package-operator to sync the secret to namespaces created by ClusterPackages.

#### Story: Package Maintainer

As a package maintainer I want to declare secret references, that later have to be provided by the package's end-user at run-time, in my PackageManifest and alias them for reference in my templates.

As a package maintainer I want to be able to request synchronization of a provided secret into Namespaces managed by a ClusterPackage.

### Notes/Constraints/Caveats

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

- Mutating secrets in place will create problems with workloads not properly watching the filesystem.
  - These workloads usually use workarounds that annotate their pods with a hash built from the secret object so that its pods are recycled when the secret changes.
  - Should we take this into consideration?
  - I was always hoping that we could leverage kustomize's ideas of generating new secrets with new names and referencing the changed secret name instead. This would have given us free workload pod rotation. But I don't think this idea is possible within PKO's model of including plaintext objects in `(Cluster)ObjectDeployments` and `(Cluster)ObjectSets`.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->
- Risk of copying the wrong secrets to the wrong places, leaking secrets to an unintended audience.
  - Although this is a real risk,
    1. we're not really more afraid of mixing up `Secrets` than mixing up other `Object` types.
    2. we'll be adding extensive integration tests to the PKO codebase in an attempt to catch any potential bugs which would result in such a leaky scenario

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Changes

#### PackageManifest API changes

The PackageManifest API needs to be changed to include an optional list `.spec.secretRefAliases`, which allows the person writing the object manifests/templates inside the package image to
  - communicate required secret references to the person deploying the package.
  - refer to secrets in their templates using the specified alias.
This effectively decouples package templates and the runtime cluster context and allows the packager to forgo hardcoding magic secret references into the package templates.


```yaml
apiVersion: manifests.package-operator.run/v1alpha1
kind: PackageManifest
metadata:
  name: my-package
spec:
  # Optional: specify a list of aliases for referecing secrets in templates.
  secrets:
  - name: tls-keypair-and-certs
    desc: |-
      Reference to a secret of type `kubernetes.io/tls`.
      Used for tls termination that is configured by the
      ingress object included in this package.
```

#### (Cluster)Package API changes

The (Cluster)Package APIs need to be changed to include these new fields under `.spec`:
1. an optional list of `secretRefs`, which can be refered to in the package's templates. For more information, refer to the sections about the PackageManifest and templating changes.
2. an optional `imagePullSecret` field which refers to an existing secret alias from `.spec.secretRefs`. If specified, the secret is resolved and used to authenticate against the registry when pulling the package image.


```yaml
kind: ClusterPackage
spec:
  # Optional: if specified, the secret reference with matching alias in `.spec.secretRefs` is used to pull the package image from its registry.
  imagePullSecret: image-pull-credentials

  # Optional: specify a list of secret references and the alias they fulfill.
  secretRefs:
  - alias: tls-keypair-and-cert
    ref:
      name: my-example-app-cert
      namespace: tls-certificates
  - alias: image-pull-credentials
    ref:
      name: image-pull-quay-io
      namespace: pull-secrets
```

```yaml
kind: Package
spec:
  # Optional: if specified, the secret reference with matching alias in `.spec.secretRefs` is used to pull the package image from its registry.
  imagePullSecret: image-pull-credentials

  # Optional: specify a list of secret references and the alias they fulfill.
  secretRefs:
  - alias: tls-keypair-and-cert
    ref:
      name: my-example-app-cert
  - alias: image-pull-credentials
    ref:
      name: image-pull-quay-io
```

Note: PKO should implement pre-flight checks to validate that all aliased secret refs requested by the PackageManifest are actually supplied in the Package object.
Note: PKO must include the secretRefs values into the hashing algo that is used to determine if the package controller should re-template the Package's ObjectDeployment to ensure re-templating when the valued change.
(This is not about watching the secrets themselves. Only the string values in package.spec.secretRefs.*.ref)

#### Package Templating changes

The templating engine needs to be changed to include functionality that resolved and exposes the aliased secret references.
Note: In this context resolving means: translate from alias to specified reference.

- `secret "alias"` -> looks up the aliased secret reference and returns the full reference object
- `secretName "alias"` -> looks up the aliased secret reference and returns the name of the referenced secret
- `secretNamespace "alias"` -> looks up the aliased secret reference and returns the namespace of the referenced secret

Note: Failure to resolve an alias should result in an error that aborts the template execution to avoid silencing a propably unwanted situation.

#### Introduction of SecretSync API

A new kind `SecretSync` has to be introduced to enable cross-namespace synchronization of secrets that already pre-exist in the cluster.

This api must support
- Specifying one synchronization strategy out of:
  - `watch`
    - PKO uses Server-Side-Apply to add the cache label to the source secret which makes it visible to the dynamic cache.
    - PKO instructs the dynamicCache facility to watch the Secret GVK and requeue SecretSync reconciliation when the source secret changes.
    - Note: PKO must establish the cache finalizer on the SecretSync object so it can ensure that it can remove the cache label from the source secret and cache resources are freed properly.
  - `poll`
    - PKO does not watch the source secret, but it will periodically re-run the full reconciliation procedure for the SecretSync object in a user specified interval.
    - The user must be able to specify the poll interval in the object manifest.
- Specifying a required reference (namespaced name) to a source secret which serves as synchronization input.
- Specifying a required list of destination secret references (namespaced names) which will serve as synchronization output.
- An optional pause field which allows a user to (temporarily) disable synchronization for debugging purposes.

Example that watches the input secret:
```yaml
apiVersion: package-operator.run/v1alpha1
kind: SecretSync
metadata:
  name: bootstrap-token
spec:
  paused: false
  strategy:
    watch: {}
  src:
    namespace: kube-system
    name: bootstrap-token-abcdef
  dest:
  - namespace: default
    name: synced-bootstrap-token
```

Example that polls the input secret:
```yaml
apiVersion: package-operator.run/v1alpha1
kind: SecretSync
metadata:
  name: bootstrap-token
spec:
  paused: false
  strategy:
    poll:
      interval: 15m
  src:
    namespace: kube-system
    name: bootstrap-token-abcdef
  dest:
  - namespace: default
    name: synced-bootstrap-token
```

The api must:
- Default `.spec.strategy` to `{watch: {}}` (default to the watch strategy if unspecified).
- Enforce that the `.spec.strategy` is immutable and cannot be changed on a live SecretSync object to avoid cache/finalizer confusions.
- Enforce that only `.spec.stategy.watch` or `.spec.stategy.poll` are set and not both.
- Enforce that, if the `poll` strategy is in use, then `.spec.stategy.poll.interval` must be set.
- Enforce a lower bound on `.spec.stategy.poll.interval` to prevent DoSing the pko-manager.
- Enforce an upper bound on the length of `.spec.dest[]` which must not exceed 32 items to prevent DoSing the pko-manager.
- Enforce unique items in `.spec.dest[]` + `.spec.src`.
- Report a Paused status condition.
- Report a Sync status condition.
- Report a Phase, like our other APIs do, too.
- Provide the same collision prevention semantics and settings as the core PKO APIs do.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->
- Upgrade strategy to maintain existing behaviour: None. The features will only be enabled when users opt-into them via the API.

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and nodes?
- How does an n-3 kubelet or kube-proxy without this feature available behave when this feature is used?
- How does an n-1 kube-controller-manager or kube-scheduler without this feature available behave when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

- No other components will be involved with introduction of this new feature at the moment.

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR enhancement proposal at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the enhancement proposal to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The enhancement proposal must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [ ] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name:
  - Components depending on the feature gate:
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.
-->

###### What happens if we reenable the feature if it was previously rolled back?

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason: 
- [ ] API .status
  - Condition name: 
  - Other field: 
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this enhancement proposal?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

- As is already the case: PKO stops working without a functional apiserver. PKO continues reconciliation when the apiserver becomes healthy again.

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->


###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a enhancement proposal should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the enhancement proposal was available
- the version of Kubernetes where the enhancement proposal graduated to general availability
- when the enhancement proposal was retired or superseded
-->

- [@Rob](https://github.com/robshelly) already created [SD-DDR-0011: Package Operator Private Registry Support](https://docs.google.com/document/d/1QwbTRQHrvopkZNxuh7HS7dfOsG6upe_QdVQQ8TXR444/) and came up with a design to handle image pull secrets.
- [@Josh](https://github.com/erdii) already created [PR-514](https://github.com/package-operator/package-operator/pull/514) and [PR-579](https://github.com/package-operator/package-operator/pull/579) to explore implementation details of SD-DDR-0011.
- [@Josh](https://github.com/erdii) has explored a draft reference implementation of the SecretSync API in [PR-1406](https://github.com/package-operator/package-operator/pull/1406)

## Drawbacks

<!--
Why should this enhancement proposal _not_ be implemented?
-->
- Building a feature to sync secrets between namespaces managed by ClusterPackage objects will probably prompt users to ask for generic object-syncing to copy other common objects around (an example being an internal CA certificate stored in a ConfigMap object). We should probably implement the sync feature for both ConfigMaps and Secrets but cannot provide facilities for every possible GroupVersionKind in the cluster or our caches would grow enormously which gives me serious scaling concerns.

## Alternatives

- Except for package image pull-secret support, all secret handling functionality could be replaced by adding objects to packages that request secrets from other sources, for example using the [Vault Secrets Operator](https://github.com/hashicorp/vault-secrets-operator) or using an operator that can copy objects from other places in the cluster.

- secret aliasing for package templates could also be a more generic obejct aliasing framework instead (but we probably don't want to open that can of worms caching wise):
  ```yaml
  kind: Package
  spec:
    objectRefs:
    - alias: my-cm
      ref:
        api: ""
        kind: ConfigMap
        name: an-existing-cm
    - alias: my-secret
      ref:
        api: ""
        kind: Secret
        name: an-existing-secret
    - alias: my-service
      ref:
        api: ""
        kind: Service
        name: an-existing-service
    - alias: my-plumbus
      ref:
        api: ""
        kind: Plumbus
        name: an-existing-plumbus
  ```
