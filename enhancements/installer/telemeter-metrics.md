---
title: installer-telemeter-metrics
authors:
  - "@patrickdillon"
  - "@rna-afk"
reviewers:
  - "@abhinavdahiya"
  - "@brancz"
  - "@crawford"
  - "@sdodson"
approvers:
  - "@abhinavdahiya"
  - "@brancz"
  - "@crawford"
  - "@sdodson"
creation-date: 2020-02-07
last-updated: 2020-02-07
status: provisional
see-also:
replaces:
superseded-by:
---

# Installer Telemeter Metrics

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Open Questions

 > 1. This requires creating an [aggregation pushgateway](https://github.com/weaveworks/prom-aggregation-gateway) where the installer would push metrics. How would this be handled and by which team?
 > 2. If #1 is possible, how would client authentication be administered to prevent [malicious pushing of metrics](risks-and-mitigations)? Based on what--a pull secret? 
 > 3. How to filter to collect only real user usage and prevent OpenShift developer/Red Hat internal users from skewing metrics? Perhaps filter by version?
 > 4. Is it possible to determine whether a user modified an ignition config or manifest asset by comparing the asset the installer would generate to the one found on disk?
 > 5. Are [packaged-scoped variables](internal-installer-design) acceptable in the proposed installer `metrics` package? 

## Summary
Telemetry collects metrics from running clusters, but no metrics can be obtained from the installer binary.
This enhancement proposes that the installer would push metrics that show installer user workflows, statistics from installations, and feature usage. The installer would be instrumented to build Prometheus labels and samples and then push those metrics to an aggregation pushgateway.  

## Motivation

Obtaining metrics from the installer would allow engineers and product managers to understand different workflows users have with the installer, installation experiences including difficulties encountered during installation, and which features are used. By pushing these metrics to the Telemeter, OpenShift members would be able to create a variety of queries related to these aspects of installer usage. 

### Goals

1. Instrument the installer to collect metrics
2. Allow the installer to push those metrics to an aggregation pushgateway which is scraped by the Telemetry Prometheus instance

### Non-Goals

1. The metrics collected will be aggregate metrics so metrics will not be relevant to individual installer invocations and will not display detailed error messages

## Proposal

### User Stories


#### Cluster Admin/Installer User Stories
As an admin about to install an OpenShift cluster I want to be able to disable metrics collection so that I can opt out.
As an admin installing an OpenShift cluster I want my installation result to be unaffected by the success or failure of collecting metrics. 

#### Telemeter User
Some examples:
As a telemeter user I would like to create queries which help me determine how long the installation process takes in total and broken down by different stages of installation so that we can better understand user experience and expose any abnormalities.

As a telemeter user I would like to create queries which would show me aggregate counts of how often failures occur during a certain stage of installation so that we can better understand where users are experiencing problems.

As a telemeter user I would like to see how often users modify any ignition configs or manifests so that I can better understand how users are interacting with the installer. (see open question #4)

As a telemeter user I would like to see how often the installer is being run from MAC OS so I can better understand our user needs.

### Implementation Details/Notes/Constraints
The installer does not fit a typical Prometheus use case in that:
* metrics will be pushed rather than scraped
* meaning will come from metric labels as much as values
* the aggregation push gateway will be used to form the metrics into a useful time series

Initially, three proposed metrics could provide value:

#### cluster_installation_invocation
The primary metric `cluster_installation_invocation` would be pushed when users invoke either the `create`, `destroy`, `gather`, or `wait-for` command. The values for the labels would be collected throughout the execution of the command and built when the metric is ready to be pushed.  The `duration` of the command will be bounded in the installer to preset buckets of a certain step-size, 2.5 in the example shown here:
```
# HELP cluster_installation_invocation represents a single command run by the OpenShift installer and related outcomes.
# TYPE cluster_installation_invocation counter
cluster_installation_invocation{command="create", target="cluster",os="linux",platform="AWS",result="ClusterComplete",return="0",version="4.2",duration="25"} 1
cluster_installation_invocation{command="create", target="cluster",os="linux",platform="AWS",result="ClusterComplete",return="0",version="4.2",duration="22.5"} 1
cluster_installation_invocation{command="create", target="cluster",os="linux",platform="AWS",result="APIFailure",return="1",version="4.2",duration="30"} 1
cluster_installation_invocation{command="destroy", target="cluster",os="linux",platform="AWS",result="Success",return="1",version="4.2",duration="10"} 1
```
The `result` label denotes a command outcome:
* `ProvisioningFailed` - for `create` the installer failed to provision resources
* `BootstrapFailed` - for `create` the bootstrap node failed
* `APIFailed` - for `create` and `wait-for` the control plane API never came up
* `OperatorsDegraded` - for `create` and `wait-for` command not all operators became available
* `Success` - for `create` and `wait-for` the cluster completed successfully and other commands completed successfully

TODO: determine relevant results for other commands, perhaps inability to connect to SSH with `destroy` or `gather` 

#### cluster_installation_duration
The cluster_installation_duration metric is pushed after execution of the `create` and `wait-for` commands. The duration of each stage of installation is represented by a bucket incremented steps set by the installer, in this example, 2.5 minutes.

```
# HELP cluster_installation_duration keeps track of the count of the duration of individual stages of the command execution in buckets with the result of the command.
# TYPE cluster_installation_duration counter
cluster_installation_duration{provisioning="5", infrastructure="5", bootstrap="5", api="5", operators="5" success="true"} 1
```
One sample would be pushed per command invocation.

#### cluster_installation_modification
The `cluster_installation_modification` metric represents whether a user modified an installer created asset prior to running the `create` command.

Feedback would be appreciated on two potential designs. One which aggregates the values into a single sample and one that pushes multiple samples:

```
# HELP cluster_installation_modification lists all the assets that were changed from the original value by the user before the create command was called.
# TYPE cluster_installation_modification counter
cluster_installation_modification{asset="cluster-config.yaml,cluster-scheduler-02-config.yml"} 1
```
In this case, the values would be sorted in order to reduce cardinality. The benefit of this approach is that it creates a correlation between the modified resources in a particular run. It is less natural than the following approach and could make querying more difficult (feedback on querying, please).

```
# HELP cluster_installation_modification lists all the assets that were changed from the original value by the user before the create command was called.
# TYPE cluster_installation_modification counter
cluster_installation_modification{asset="cluster-config.yaml"} 1
cluster_installation_modification{asset="cluster-scheduler-02-config.yml"} 1
```

Either of these proposed metrics would be pushed only upon successful completion of the `create cluster` command.

#### Metric Labels
Collecting labels into a single metric sample and pushing once means that each sample represents a single installer invocation. When a sample is pushed to the aggregation pushgateway metrics with matching labels will be incremented:

```
cluster_installation_invocation{command="create", target="cluster",os="linux",platform="AWS",result="ClusterComplete",return="0",version="4.2",duration="25"} 100
cluster_installation_invocation{command="create", target="cluster",os="linux",platform="GCP",result="APIFailure",return="1",version="4.3",duration="30"} 25
cluster_installation_invocation{command="destroy", target="cluster",os="linux",platform="AWS",result="Success",return="1",version="4.2",duration="10"} 5
```
If the above example represents the state of our Prometheus database, it would indicate that there have been 100 successful cluster installations on AWS that took approximately 25 minutes.

The values for these labels may only become available at different times during an installation or command execution. Therefore the installer instrumentation must handle building labels by collecting key value pairs as they become available.

#### Internal Installer Design
A new metrics package would be created to handle the collection of metrics, building labels, and pushing to the aggregation pushgateway. 

As required label values are gathered throughout many installer packages,  the `metrics` package would maintain the state of metrics in package-scoped variables (feedback please: package-scoped variables may be generally frowned upon but is this a valid use case?).

##### API
The package acts as a lightweight wrapper around the Prometheus SDK. The `metrics` package would expose the following functions to be used by other packages within the installer:
```
// AddLabelValue adds a label key/value pair to the label being built for metric.
AddLabelValue(metric, key, value string)

// Push takes in a metric job name and pushes the values to the gateway.
Push(metric string)

// SetValue sets the value to the metric. The value would always be 1 for the present design.
SetValue(metric, value string)
```

The metric `cluster_installation_modification` would require a separate API function based on the decision listed above. In the second case, where each modified asset is sent in a separate sample, the API function would simply wrap the Prometheus SDK call to add to a metrics vector:

```
// AddToMetric adds a sample to a metrics vector.
AddToMetric(metric string, labels map[string]string, value string)
```

### Risks and Mitigations

The pushgateway would be an a public endpoint reachable on the internet. Therefore some authentication or security should be in place in order to prevent malicious pushing of metrics.

Users and customers may be wary of collecting metrics from the installer.


## Design Details

### Test Plan

**TODO**

**Note:** *Section not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).

### Graduation Criteria

**TODO**

**Note:** *Section not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, or as something else. Initial proposal
should keep this high-level with a focus on what signals will be looked at to
determine graduation.

Consider the following in developing the graduation criteria for this
enhancement:
- Maturity levels - `Dev Preview`, `Tech Preview`, `GA`
- Deprecation

Clearly define what graduation means.

#### Examples

These are generalized examples to consider, in addition to the aforementioned
[maturity levels][maturity-levels].

##### Dev Preview -> Tech Preview

- Ability to utilize the enhancement end to end
- End user documentation, relative API stability
- Sufficient test coverage
- Gather feedback from users rather than just developers

##### Tech Preview -> GA 

- More testing (upgrade, downgrade, scale)
- Sufficient time for feedback
- Available by default

**For non-optional features moving to GA, the graduation criteria must include
end to end tests.**

##### Removing a deprecated feature

- Announce deprecation and support policy of the existing feature
- Deprecate the feature

### Upgrade / Downgrade Strategy

Not applicable.

### Version Skew Strategy

Not applicable. 

## Implementation History

Major milestones in the life cycle of a proposal should be tracked in `Implementation
History`.

## Drawbacks

By using the aggregation pushgateway, we are losing specificity. We will not be able to tell whether one user heavily skews results.

 We want to understand user events in how they interact with the installer and get specific error messages and logs, but we cannot do that with Prometheus as a backing service. 

## Alternatives


A standard pushgateway could be used to collect individually identified metrics from installer invocations, which could then be summed up, getting us similar results as the aggregation gateway but with finer granularity. We do not pursue this option due to cardinality concerns.

## Infrastructure Needed

An aggregation pushgateway would need to be created and administered, perhaps by a service delivery team.
