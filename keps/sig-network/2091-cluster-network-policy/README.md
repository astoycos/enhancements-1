# KEP-2091: Add support for ClusterNetworkPolicy resources

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [ClusterNetworkPolicy resource](#clusternetworkpolicy-resource)
  - [ClusterDefaultNetworkPolicy resource](#defaultnetworkpolicy-resource)
  - [Precedence model](#precedence-model)
  - [User Stories](#user-stories)
    - [Story 1: Deny traffic from certain sources](#story-1-deny-traffic-from-certain-sources)
    - [Story 2: Funnel traffic through ingress/egress gateways](#story-2-funnel-traffic-through-ingressegress-gateways)
    - [Story 3: Isolate multiple tenants in a cluster](#story-3-isolate-multiple-tenants-in-a-cluster)
    - [Story 4: Enforce network/security best practices](#story-4-enforce-networksecurity-best-practices)
    - [Story 5: Restrict egress to well known destinations](#story-5-restrict-egress-to-well-known-destinations)
  - [RBAC](#rbac)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [Future Work](#future-work)
- [Design Details](#design-details)
  - [ClusterNetworkPolicy API Design](#clusternetworkpolicy-api-design)
  - [ClusterDefaultNetworkPolicy API Design](#defaultnetworkpolicy-api-design)
  - [Shared API Design](#shared-api-design)
    - [AppliedTo](#appliedto)
    - [Namespaces](#namespaces)
    - [IPBlock](#ipblock)
  - [Sample Specs for User Stories](#sample-specs-for-user-stories)
    - [Story 1: Deny traffic from certain sources](#story-1-deny-traffic-from-certain-sources-1)
    - [Story 2: Funnel traffic through ingress/egress gateways](#story-2-funnel-traffic-through-ingressegress-gateways-1)
    - [Story 3: Isolate multiple tenants in a cluster](#story-3-isolate-multiple-tenants-in-a-cluster-1)
    - [Story 4: Enforce network/security best practices](#story-4-enforce-networksecurity-best-practices-1)
    - [Story 5: Restrict egress to well known destinations](#story-5-restrict-egress-to-well-known-destinations-1)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha to Beta Graduation](#alpha-to-beta-graduation)
    - [Beta to GA Graduation](#beta-to-ga-graduation)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Upgrade considerations](#upgrade-considerations)
    - [Downgrade considerations](#downgrade-considerations)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Scalability](#scalability)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [NetworkPolicy v2](#networkpolicy-v2)
  - [Single CRD with DefaultRules field](#single-crd-with-defaultrules-field)
  - [Single CRD with IsOverrideable field](#single-crd-with-isoverrideable-field)
  - [Single CRD with BaselineAllow as Action](#single-crd-with-baselineallow-as-action)
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] (R) Graduation criteria is in place
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Introduce new set of APIs to express an administrator's intent in securing
their K8s cluster. This doc proposes two new set of resources,
ClusterNetworkPolicy API and the ClusterDefaultNetworkPolicy API to complement
the developer focused NetworkPolicy API in Kubernetes.

## Motivation

Kubernetes provides NetworkPolicy resources to control traffic within a
cluster. NetworkPolicies focus on expressing a developers intent to secure
their applications. Thus, in order to satisfy the needs of a security admin,
we propose to introduce new set of APIs that capture the administrators intent.

### Goals

The goals for this KEP are to satisfy two key user stories:
1. As a cluster administrator, I want to enforce irrevocable guardrails that
   all workloads must adhere to in order to guarantee the safety of my clusters.
2. As a cluster administrator, I want to deploy a default set of policies to all
   workloads that may be overridden by the developers if needed.

There are several unique properties that we need to add in order accomplish the
user stories above.
1. Deny rules and, therefore, hierarchical enforcement of policy
2. Semantics for a cluster-scoped policy object that may include
   namespaces/workloads that have not been created yet.
3. Backwards compatibility with existing Kubernetes Network Policy API

### Non-Goals

Our mission is to solve the most common use cases that cluster admins have.
That is, we don't want to solve for every possible policy permutation a user
can think of. Instead, we want to design an API that addresses 90-95% use cases
while keeping the mental model easy to understand and use.

Additionally, this proposal is squarely focused on solving the needs of the
Cluster Administrator. It is not intended to solve:
1. Logging / error reporting for Network Policy
2. Kubernetes Node policies
3. New policy selectors (services, service accounts, etc.)
4. Tools/CLI to debug/explain NetworkPolicies, cluster-scoped policies and their
   impact on workloads.

## Proposal

In order to achieve the two primary broad use cases for a cluster admin to
secure K8s clusters, we propose to introduce the following two new resources
under `networking.k8s.io` API group:
- ClusterNetworkPolicy
- ClusterDefaultNetworkPolicy

### ClusterNetworkPolicy resource

A ClusterNetworkPolicy resource will help the administrators set strict
security rules for the cluster, i.e. a developer CANNOT override these rules
by creating NetworkPolicies that applies to the same workloads as the
ClusterNetworkPolicy does.

Unlike the NetworkPolicy resource in which each rule represents an allowed
traffic, ClusterNetworkPolicy will enable administrators to set `Authorize`,
`Deny` or `Allow` as the action of each rule. ClusterNetworkPolicy rules should
be read as-is, i.e. there will not be any implicit isolation effects for the Pods
selected by the ClusterNetworkPolicy, as opposed to what NetworkPolicy rules imply.

In terms of precedence, the aggregated `Authorize` rules (all ClusterNetworkPolicy
rules with action `Authorize` in the cluster combined) should be evaluated before
aggregated ClusterNetworkPolicy `Deny` rules, followed by aggregated ClusterNetworkPolicy
`Allow` rules, followed by NetworkPolicy rules in all Namespaces. As such, the
`Authorize` rules have the highest precedence, which in most cases shall only be
used to provide exceptions to deny-all rules and authorize traffic from/to well-known
entities.

ClusterNetworkPolicy `Deny` rules are useful for administrators to explicitly
block traffic from malicious clients, or workloads that poses security risks.
Those traffic restrictions can only be lifted once the `Deny` rules are deleted
or modified by the admin. In clusters where the admin requires total control over
security postures of all workloads, the `Deny` rules can also be used to deny all
incoming/outgoing traffic in the cluster, with few exceptions that's listed out
by `Authorize` rules. On the other hand, the `Allow` rules can be used to call out
traffic in the cluster that needs to be allowed for certain components to work
as expected (egress to CoreDNS for example). Those traffic should not be blocked
when developers apply NetworkPolicy to their Namespaces which isolates the workloads.

### ClusterDefaultNetworkPolicy resource

A ClusterDefaultNetworkPolicy resource will help the administrators set baseline
security rules for the cluster, i.e. a developer CAN override these rules by creating
NetworkPolicies that applies to the same workloads as the ClusterDefaultNetworkPolicy
does.

ClusterDefaultNetworkPolicy works just like NetworkPolicy except that it is cluster-scoped.
When workloads are selected by a ClusterDefaultNetworkPolicy, they are isolated except
for the ingress/egress rules specified. ClusterDefaultNetworkPolicy rules will not have
actions associated -- each rule will be an 'allow' rule.

Aggregated NetworkPolicy rules will be evaluated before aggregated ClusterDefaultNetworkPolicy
rules. If a Pod is selected by both, a ClusterDefaultNetworkPolicy and a NetworkPolicy,
then the ClusterDefaultNetworkPolicy's effect on that Pod becomes obsolete.
In this case, the traffic allowed will be solely determined by the NetworkPolicy.

### Precedence model

```
                    Yes -------> [ Allow ]             
                     |                                                             
                     |                                                           
         +-----------------------+      
  ---->  |    traffic matches    |   
         | a CNP Authorize rule? |                 
         +-----------------------+             
                 |                                                                
                 No                               Yes ------> [ Allow ]           Yes ------> [ Allow ]
                 |                                 |                               |
                 V                                 |                               |
         +------------------+             +-------------------+              +------------------+
         | traffic matches  | --- No -->  | traffic matches   |  --- No -->  | traffic matches  |
         | a CNP Deny rule? |             | a CNP Allow rule? |              | a NetworkPolicy  |
         +------------------+             +-------------------+              | rule?            |
                 |                                                           +------------------+
                 |                                                                 |
                Yes -------> [ Drop ]                                              No
                                                                                   |
                                                                                   V
         +------------------+             +-------------------+              +------------------+
  <----  | traffic matches  | <--- No --- | traffic matches   |  <--- No --- | traffic matches  |
         | DNP default      |             | a DNP Allow rule? |              | NP default       |
         | isolation(*)?    |             +-------------------+              | isolation(*)?    |
         +------------------+                      |                         +------------------+
                |                                  |                                |
                |                                 Yes -------> [ Allow ]            |
               Yes ------> [ Drop ]                                                Yes ------> [ Drop ]


CNP = ClusterNetworkPolicy   DNP = ClusterDefaultNetworkPolicy   NP = NetworkPolicy
(*) If a Pod has a ingress NetworkPolicy applied, then any ingress traffic to the Pod that does
    not match the NetworkPolicy's ingress rules, matches NetworkPolicy default isolation
    (the Pod is isolated for ingress). Same applies for egress. Same applies for DNP.

```

The diagram above explains the rule evaluation precedence between ClusterNetworkPolicy,
NetworkPolicy and ClusterDefaultNetworkPolicy.

Consider the following scenario:

- Pod `server` exists in Namespace x. Each Namespace [a, b, c, d] has a Pod named `client`.
The following policy resources also exist in the cluster:
- (1) A ClusterNetworkPolicy `Authorize` rule selects Namespace x and allows ingress traffic from Namespace a.
- (2) A ClusterNetworkPolicy `Deny` rule selects Namespace x and denies all ingress traffic from Namespace a and b.
- (3) A ClusterNetworkPolicy `Allow` rule selects Namespace x and allows all ingress traffic Namespace b and c.
- (4) A NetworkPolicy rule isolates [x/server], only allows ingress traffic from its own Namespace.
- (5) A ClusterDefaultNetworkPolicy rule isolates [x/server], only allows ingress traffic from Namespace d.

Now suppose the client in each Namespace initiates traffic towards x/server.
- a/client -> x/server is affected by rule (1), (2), (4) and (5). Since rule (1) has highest precedence, the request should be allowed.
- b/client -> x/server is affected by rule (2), (3), (4) and (5). Since rule (2) has highest precedence, the request should be denied.
- c/client -> x/server is affected by rule (3), (4) and (5). Since rule (3) has highest precedence, the request should be allowed.
- d/client -> x/server is affected by rule (4) and (5). Since rule (4) has higher precedence, the request should be denied.

### User Stories

![Alt text](user_story_diagram.png?raw=true "User Story Diagram")

#### Story 1: Deny traffic from certain sources

As a cluster admin, I want to explicitly deny traffic from certain source IPs
that I know to be bad.

Many admins maintain lists of IPs that are known to be bad actors, especially
to curb DoS attacks. A cluster admin could use ClusterNetworkPolicy to codify
all the source IPs that should be denied in order to prevent that traffic from
accidentally reaching workloads. Note that the inverse of this (allow traffic
from well known source IPs) is also a valid use case.

#### Story 2: Funnel traffic through ingress/egress gateways

As a cluster admin, I want to ensure that all traffic coming into (going out of)
my cluster always goes through my ingress (egress) gateway.

It is common practice in enterprises to setup checkpoints in their clusters at
ingress/egress. These checkpoints usually perform advanced checks such as
firewalling, authentication, packet/connection logging, etc.
This is a big request for compliance reasons, and ClusterNetworkPolicy can ensure
that all the traffic is forced to go through ingress/egress gateways.
It is worth noting that the Cluster-scoped NetworkPolicy APIs will not redirect
traffic, rather it can ensure that no traffic is allowed in/out except traffic
via the gateways.

#### Story 3: Isolate multiple tenants in a cluster

As a cluster admin, I want to isolate all the tenants (modeled as Namespaces)
on my cluster from each other by default. Tenancy may be modeled as 1:1, where
1 tenant is mapped to a single Namespace, or 1:n, where a single tenant may
own more than 1 Namespace.

Many enterprises are creating shared Kubernetes clusters that are managed by a
centralized platform team. Each internal team that wants to run their workloads
gets assigned a Namespace on the shared clusters. Naturally, the platform team
will want to make sure that, by default, all intra-namespace traffic management
is authorized by the Namespace owners and all inter-namespace traffic is denied.

#### Story 4: Enforce network/security best practices

As a cluster admin, I want all workloads to start with a network/security
model that meets the needs of my company.

In order to follow best practices, an admin may want to begin the cluster
lifecycle with a default zero-trust security model, where in the default policy
of the cluster is to deny traffic. Only traffic that is essential to the cluster
will be opened up with stricter cluster level policies. Namespace owners are therefore
forced to use NetworkPolicies to explicitly allow only known traffic. This follows
a model which is familiar to many security administrators, where in you deny by default
and then poke holes in the cluster by adding explicit allow rules.

#### Story 5: Restrict egress to well known destinations

As a cluster admin, I want to explicitly limit which workloads can connect to
well known destinations outside the cluster.

This user story is particularly relevant in hybrid environments where customers
have highly restricted databases running behind static IPs in their networks
and want to ensure that only a given set of workloads is allowed to connect to
the database for PII/privacy reasons. Using ClusterNetworkPolicy, a user can
write a policy to guarantee that only the selected pods can connect to the
database IP.

### RBAC

Cluster-scoped NetworkPolicy resources are meant for cluster administrators.
Thus, access to manage these resources must be granted to subjects which have
the authority to outline the security policies for the cluster. Therefore, by
default, the `admin` and `edit` ClusterRoles will be granted the permissions
to edit the cluster-scoped NetworkPolicy resources.

### Notes/Constraints/Caveats

It is important to note that the controller implementation for cluster-scoped
policy APIs will not be provided as part of this KEP. Such controllers which
realize the intent of these APIs will be provided by individual CNI providers,
as is the case with the NetworkPolicy API.

### Risks and Mitigations

A potential risk of the ClusterNetworkPolicy resource is, when it's stacked on
top of existing NetworkPolicies in the cluster, some existing allowed traffic
patterns (which were regulated by those NetworkPolicies) may become blocked by
ClusterNetworkPolicy `Deny` rules, while some isolated workloads may become
accessible instead because of ClusterNetworkPolicy `Allow` or `Authorize` rules.

Developers could face some difficulties figuring out why the NetworkPolicies
did not take effect, even if they know to look for ClusterNetworkPolicy rules
that can potentially override these policies:
To understand why traffic between a pair of Pods is allowed/denied, a list of
NetworkPolicy resources in both Pods' Namespace used to be sufficient
(considering no other CRDs in the cluster tries to alter traffic behavior).
The same Pods, on the other hand, can appear as an AppliedTo, or an ingress/egress
peer in any ClusterNetworkPolicy. This makes looking up policies that affect a
particular Pod more challenging than when there's only NetworkPolicy resources.

In addition, in an extreme case where a ClusterNetworkPolicy `Authorize` rule,
ClusterNetworkPolicy `Deny` rule, ClusterNetworkPolicy `Allow` rule,
NetworkPolicy rule and ClusterDefaultNetworkPolicy rule applies to an overlapping
set of Pods, users will need to refer to the precedence model mentioned in the
[previous section](precedence-model) to determine which rule would take effect.
As shown in that section, figuring out how stacked policies affect traffic
between workloads might not be very straightfoward.

To mitigate this risk and improve UX, a tool which reversely looks up affecting
policies for a given Pod and prints out relative precedence of those rules
can be quite useful. The [cyclonus](https://github.com/mattfenwick/cyclonus)
project for example, could be extended to support ClusterNetworkPolicy and
ClusterDefaultNetworkPolicy. This is an orthogonal effort and will not be addressed
by this KEP in particular.

### Future Work

Although the scope of the cluster-scoped policies is wide, the above proposal
intends to only solve the use cases documented in this KEP. However, we would
also like to consider the following set of proposals as future work items:
- **Logging**: Very often cluster administrators want to log every connection
  that is either denied or allowed by a firewall rule and send the details to
  an IDS or any custom tool for further processing of that information.
  With the introduction of `deny` rules, it may make sense to incorporate the
  cluster-scoped policy resources with a new field, say `loggingPolicy`, to
  determine whether a connection matching a particular rule/policy must be
  logged or not.
- **Rule identifier**: In order to collect traffic statistics corresponding to
  a rule, it is necessary to identify the rule which allows/denies that traffic.
  This helps administrators figure the impact of the rules written in a
  cluster-scoped policy resource. Thus, the ability to uniquely identify a rule
  within a cluster-scoped policy resource becomes very important.
  This can be addressed by introducing a field, say `name`, per `ClusterNetworkPolicy`
  and `ClusterDefaultNetworkPolicy` ingress/egress rule.
- **Node Selector**: Cluster administrators and developers want to write
  policies that apply to cluster nodes or host network pods. This can be
  addressed by introducing nodeSelector field under `appliedTo` field of the
  `ClusterNetworkPolicy` and `ClusterDefaultNetworkPolicy` spec.
  `ClusterDefaultNetworkPolicy` is a better candidate compared to K8s `NetworkPolicy`
  for introducing this field as nodes are cluster level resources.

## Design Details

### ClusterNetworkPolicy API Design
The following new `ClusterNetworkPolicy` API will be added to the `netpol.networking.k8s.io` API group.

**Note**: Much of the behavior of certain fields proposed below is intentionally aligned with K8s NetworkPolicy
resource, wherever possible. For eg. the behavior of empty or missing fields matches the behavior specified in
the NetworkPolicySpec.

```golang
type ClusterNetworkPolicy struct {
	metav1.TypeMeta
	metav1.ObjectMeta

    // Specification of the desired behavior of ClusterNetworkPolicy.
	Spec ClusterNetworkPolicySpec
}

type ClusterNetworkPolicySpec struct {
	// No implicit isolation of AppliedTo Pods/Namespaces.
	// Required field.
	AppliedTo    AppliedTo
	Ingress      []ClusterNetworkPolicyIngressRule
	Egress       []ClusterNetworkPolicyEgressRule
}

type ClusterNetworkPolicyIngress/EgressRule struct {
	// Action specifies whether this rule must allow traffic or deny traffic. Deny rules take
	// precedence over allow rules. Any exception to a deny rule must be written as an Authorize
	// rule which takes highest precedence. i.e. Authorize > Deny > Allow
	// Required field for any rule.
	Action       RuleAction
	// List of ports for incoming/outgoing traffic.
	// Each item in this list is combined using a logical OR. If this field is
	// empty or missing, this rule matches all ports (traffic not restricted by port).
	// If this field is present and contains at least one item, then this rule allows/denies
	// traffic only if the traffic matches at least one port in the list.
	// +optional
	Ports        []networkingv1.NetworkPolicyPort
	// List of sources/dest which should be able to access the pods selected for this rule.
	// Items in this list are combined using a logical OR operation. If this field is
	// empty or missing, this rule matches all sources/dest (traffic not restricted by
	// source/dest). If this field is present and contains at least one item, this rule
	// allows/denies traffic only if the traffic matches at least one item in the from/to list.
	// +optional
	From/To      []ClusterNetworkPolicyPeer
}

type ClusterNetworkPolicyPeer struct {
	PodSelector  *metav1.LabelSelector
	// required if a PodSelector is specified
	Namespaces   *Namespaces
	IPBlock      *networkingv1.IPBlock
}

const (
	// RuleActionAuthorize is the highest priority allow rules which enables admins to provide exceptions to deny rules.
	RuleActionAuthorize RuleAction = "Authorize"
	// RuleActionDeny enables admins to deny specific traffic. Any exception to this deny rule must be overridden by
	// creating a RuleActionAuthorize rule.
	RuleActionDeny      RuleAction = "Deny"
	// RuleActionAllow enables admins to specifically allow certain traffic. These rules will be enforced after
	// Authorize and Deny rules.
	RuleActionAllow     RuleAction = "Allow"
)
```

For the ClusterNetworkPolicy ingress/egress rule, the `Action` field dictates whether
traffic should be allowed or denied or always allowed from/to the ClusterNetworkPolicyPeer.
This will be a required field.

### ClusterDefaultNetworkPolicy API Design
The following new `ClusterDefaultNetworkPolicy` API will be added to the `netpol.networking.k8s.io` API group:

```golang
type ClusterDefaultNetworkPolicy struct {
	metav1.TypeMeta
	metav1.ObjectMeta
	// Specification of the desired behavior of ClusterDefaultNetworkPolicy.
	Spec ClusterDefaultNetworkPolicySpec
}

type ClusterDefaultNetworkPolicySpec struct {
	// Implicit isolation of AppliedTo Pods.
	AppliedTo   AppliedTo
	Ingress     []ClusterDefaultNetworkPolicyIngressRule
	Egress      []ClusterDefaultNetworkPolicyEgressRule
}

type ClusterDefaultNetworkPolicyIngress/EgressRule struct {
	// List of ports for incoming/outgoing traffic.
	// Each item in this list is combined using a logical OR. If this field is
	// empty or missing, this rule matches all ports (traffic not restricted by port).
	// If this field is present and contains at least one item, then this rule allows
	// traffic only if the traffic matches at least one port in the list.
	// +optional
	Ports       []networkingv1.NetworkPolicyPort
	// List of sources/dest which should be able to access the pods selected for this rule.
	// Items in this list are combined using a logical OR operation. If this field is
	// empty or missing, this rule matches all sources/dest (traffic not restricted by
	// source/dest). If this field is present and contains at least one item, this rule
	// allows traffic only if the traffic matches at least one item in the from/to list.
	// +optional
	From/To     []ClusterDefaultNetworkPolicyPeer
}

type ClusterDefaultNetworkPolicyPeer struct {
	PodSelector       *metav1.LabelSelector
	// one of NamespaceSelector or Namespaces is required, if a PodSelector is specified
	NamespaceSelector *metav1.LabelSelector
	Namespaces        *Namespaces
	IPBlock           *networkingv1.IPBlock
}
```

### Shared API Design
The following structs will be added to the `netpol.networking.k8s.io` API group and
shared between `ClusterNetworkPolicy` and `ClusterDefaultNetworkPolicy`:

```golang
type AppliedTo struct {
	// required if a PodSelector is specified
	NamespaceSelector   *metav1.LabelSelector
	// optional
	PodSelector         *metav1.LabelSelector
}

// Namespaces define a way to select Namespaces in the cluster.
type Namespaces struct {
	Scope       NamespaceMatchType
	// Labels are set only when scope is "SameLabels".
	Labels      []string
	// Selector is only set when scope is "Selector".
	Selector    *metav1.LabelSelector
}

// NamespaceMatchType describes Namespace matching strategy.
type NamespaceMatchType string

// This list can/might get expanded in the future (i.e. NotSelf etc.)
const (
	NamespaceMatchSelf          NamespaceMatchType = "Self"
	NamespaceMatchSelector      NamespaceMatchType = "Selector"
	NamespaceMatchSameLabels    NamespaceMatchType = "SameLabels"
)

```

#### AppliedTo
The `AppliedTo` field in Cluster scoped network policies is what `Spec.PodSelector` field is to K8s NetworkPolicy spec,
as means to specify the target Pods that this cluster-scoped policy (either `ClusterNetworkPolicy` or
`ClusterDefaultNetworkPolicy`) applies to.
Since the policy is cluster-scoped, the `NamespaceSelector` field is required.
An empty `NamespaceSelector` (namespaceSelector: {}) selects all Namespaces in the Cluster.

#### Namespaces
The `Namespaces` field replaces `NamespaceSelector` in NetworkPolicyPeer, as
means to specify the Namespaces of ingress/egress peers for cluster-scoped policies.
The scope of the Namespaces to be selected is specified by the matching strategy chosen.
For selecting Pods from specific Namespaces, the `Selector` scope works exactly as `NamespaceSelector`.
The `Self` scope is added to satisfy the specific needs for cluster-scoped policies:

__Self:__
This is a special strategy to indicate that the rule only applies to the Namespace for
which the ingress/egress rule is currently being evaluated upon. Since the Pods
selected by the ClusterNetworkPolicy appliedTo could be from multiple Namespaces,
the scope of ingress/egress rules whose `namespace.scope=self` will be the Pod's
own Namespace for each selected Pod.
Consider the following example:

- Pods [a1, b1] exist in Namespace x, which has labels `app=a` and `app=b` respectively.
- Pods [a2, b2] exist in Namespace y, which also has labels `app=a` and `app=b` respectively.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterDefaultNetworkPolicy
spec:
  appliedTo:
    namespaceSelector: {}
  ingress:
    - onlyFrom:
      - namespaces:
          scope: self
        podSelector:
          matchLabels:
            app: b
```

The above ClusterDefaultNetworkPolicy should be interpreted as: for each Namespace in
the cluster, all Pods in that Namespace should only allow traffic from Pods in
the _same Namespace_ who has label app=b. Hence, the policy above allows
x/b1 -> x/a1 and y/b2 -> y/a2, but denies y/b2 -> x/a1 and x/b1 -> y/a2.

__SameLabels:__
This is a special strategy to indicate that the rule only applies to the Namespaces
which share the same label value. Since the Pods selected by the ClusterNetworkPolicy appliedTo
could be from multiple Namespaces, the scope of ingress/egress rules whose `namespace.scope=samelabels; labels: [tenant]`
will be all the Pods from the Namespaces who have the same label value for the "tenant" key.
Consider the following example:

- Pods [a1, b1] exist in Namespace coke-1, which has label `tenant=coke`.
- Pods [a2, b2] exist in Namespace coke-2, which has label `tenant=coke`.
- Pods [a3, b3] exist in Namespace pepsi-1, which has label `tenant=pepsi`.
- Pods [a4, b4] exist in Namespace pepsi-2, which has label `tenant=pepsi`.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterDefaultNetworkPolicy
spec:
  appliedTo:
    namespaceSelector: {}
  ingress:
    - onlyFrom:
      - namespaces:
          scope: samelabels
          labels:
            - tenant
```

The above ClusterDefaultNetworkPolicy should be interpreted as: for each Namespace in
the cluster, all Pods in that Namespace should only allow traffic from all Pods in
the Namespaces who has the same label value for key `tenant`. Hence, the policy above allows
all Pods in Namespaces labeled `tenant=coke` i.e. coke-1 and coke-2, to reach each other,
similarly allow all Pods in Namespaces labeled `tenant=pepsi` i.e. pepsi-1 and pepsi-2, are allowed
to talk to each other, however it does not allow any Pod in coke-1 or coke-2 to reach Pods in
pepsi-1 or pepsi-2.

#### IPBlock

The `ClusterNetworkPolicyPeer` and `ClusterDefaultNetworkPolicyPeer` both allow the
ability to set an `IPBlock` as a peer. The usage of this field is similar to
how it is used in the NetworkPolicyPeer. However, we should also explicitly
note that the IPBlock set in this field could belong to the cluster locally, or
could be cluster external. For example, the peer could be set with a subnet
or an IP which maps to a Pod existing in the cluster. This means that a Pod
in a cluster-scoped NetworkPolicy could be identified by either the labels
applied on the Pod, or its IP address. In case of multiple conflicting rules
targeting the same Pod, but identified in different ways, such as via labelSelector
or via PodIP set in IPBlock, the net effect of the rules will be determined
by the `action` associated with the rule and/or the resource in which the
rule is set in, i.e. ClusterNetworkPolicy rule takes precedence over a
ClusterDefaultNetworkPolicy rule.

### Sample Specs for User Stories

![Alt text](user_story_diagram.png?raw=true "User Story Diagram")

#### Story 1: Deny traffic from certain sources
As a cluster admin, I want to explicitly deny traffic from certain source IPs
that I know to be bad.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterNetworkPolicy
metadata:
  name: deny-bad-ip
spec:
  appliedTo:
    # if there's an ingress gateway in the cluster, applying the policy to
    # gateway namespace will be sufficient
    namespaceSelector: {}
  ingress:
    - action: Deny
      from:
      - ipBlock:
          cidr: 192.0.2.0/24  # banned addresses
```

#### Story 2: Funnel traffic through ingress/egress gateways
As a cluster admin, I want to ensure that all traffic coming into (going out of)
my cluster always goes through my ingress (egress) gateway.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterNetworkPolicy
metadata:
  name: ingress-egress-gateway
spec:
  appliedTo:
    namespaceSelector:
      matchLabels:
        type: tenant  # assuming all tenant namespaces will be created with this label
  ingress:
    - action: Authorize
      from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: dmz # ingress gateway
      - namespaces:
          scope: self
    - action: Deny
      from:
      - ipBlock:
          cidr: 0.0.0.0/0
  egress:
    - action: Authorize
      from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: istio-egress # egress gateway
      - namespaces:
          scope: self
    - action: Deny
      to:
      - ipBlock:
          cidr: 0.0.0.0/0
```

__Note:__ The above policy is very restrictive, i.e. it rejects ingress/egress
traffic between tenant Namespaces and `kube-system`. For `coredns` etc. to work,
`kube-system` Namespace or at least the `app=kube-dns` pods needs to be added into
the `Allow` rule list.

#### Story 3: Isolate multiple tenants in a cluster

As a cluster admin, I want to isolate all the tenants (modeled as Namespaces)
on my cluster from each other by default.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterDefaultNetworkPolicy
metadata:
  name: namespace-isolation
spec:
  appliedTo:
    namespaceSelector:
      matchLabels:
        type: tenant  # assuming all tenant namespaces will be created with this label
  ingress:
    - onlyFrom:
      - namespaces:
          self: true
```

__Note:__ The above policy will take no effect if applied together with
`ingress-egress-gateway`, since both policies apply to the same Namespaces, and
ClusterNetworkPolicy rules have higher precedence than ClusterDefaultNetworkPolicy
rules.
In this policy, tenant isolation (Namespace being the boundary) is not enforced,
but rather provided as a cluster default for initial security postures in each
tenant. Tenants can completely overwrite this based on the need of its namespaces,
using NetworkPolicy. They cannot, however, overwrite Allow, Deny and Authorize
rules which cluster admins listed out as guardrails (dns must be allowed, egress
to some IPs must be denied etc.) If strict tenant isolation is desired, cluster
admins should write a ClusterNetworkPolicy instead. See Story 4 below.

#### Story 4: Enforce network/security best practices

As a cluster admin, I want all workloads to start with a network/security
model that meets the needs of my company.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterNetworkPolicy
spec:
  appliedTo:
    namespaceSelector:
      matchLabels:
        type: tenant  # assuming all tenant namespaces will be created with this label
  ingress:
    - action: Allow
      from:
      - namespaceSelector:
          matchLabels:
              app: system  # which can include kube-system and logging/monitoring namespaces
```

__Note:__ The above policy only ensures that traffic from `app=system` Namespaces
will not be blocked, if developers create NetworkPolicy which isolates the Pods in
tenant Namespaces. When there's a ClusterNetworkPolicy like `ingress-egress-gateway`
present in the cluster, the above policy will be overridden as `Deny` rules have
higher precedence than `Allow` rules. In that case, the `app=system` Namespaces
needs to be allowed using an `Authorize` action.

A strict tenant isolation policy can also be written if that's required (for
compliance purposes etc):
```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterNetworkPolicy
spec:
  appliedTo:
    namespaceSelector:
      matchLabels:
        type: tenant  # assuming all tenant namespaces will be created with this label
  ingress:
    - action: Authorize
      from:
      - namespaces:
          scope: self
    - action: Deny
      from:
      - namespaces:
          selector: {}
```

#### Story 5: Restrict egress to well known destinations

As a cluster admin, I want to explicitly limit which workloads can connect to
well known destinations outside the cluster.

```yaml
apiVersion: networking.k8s.io/v1alpha1
kind: ClusterNetworkPolicy
metadata:
  name: restrict-egress-to-db
spec:
  appliedTo:
    namespaceSelector: {}
    podSelector:
      matchExpressions:
        - {key: app, operator: NotIn, values: [authorized-client]}
  egress:
    - action: Deny
      to:
      - ipBlock:
          cidr: 10.220.0.8/32  # restricted database running behind static IP
```

### Test Plan

- Add e2e tests for ClusterNetworkPolicy resource
  - Ensure `Authorize`rules are always allowed
  - Ensure `Deny` rules override all allowed traffic in the cluster, except for `Authorize` traffic.
  - Ensure `Allow` rules override K8s NetworkPolicies
  - Ensure that in stacked ClusterNetworkPolicies/K8s NetworkPolicies, the following precedence is maintained
    aggregated `Deny` rules > aggregated `Allow` rules > K8s NetworkPolicy rules
- Add e2e tests for ClusterDefaultNetworkPolicy resource
  - Ensure that in absence of ClusterNetworkPolicy rules and K8s NetworkPolicy rules, ClusterDefaultNetworkPolicy rules are observed
  - Ensure that K8s NetworkPolicies override DefaultNetworkPolicies by applying policies to the same workloads
  - Ensure that stacked DefaultNetworkPolicies are additive in nature
- e2e test cases must cover ingress and egress rules
- e2e test cases must cover port-ranges, named ports, integer ports etc
- e2e test cases must cover various combinations of `podSelector` in `appliedTo` and ingress/egress rules
- e2e test cases must cover various combinations of `namespaceSelector` in `appliedTo`
- e2e test cases must cover various combinations of `namespaces` in ingress/egress rules
  - Ensure that `self` field works as expected
- Add unit tests to test the validation logic which shall be introduced for cluster-scoped policy resources
  - Ensure that `self` field cannot be set along with `selector` within `namespaces`
  - Test cases for fields which are shared with NetworkPolicy, like `ipBlock`, `endPort` etc.
- Ensure that only administrators or assigned roles can create/update/delete cluster-scoped policy resources

### Graduation Criteria

#### Alpha to Beta Graduation

- Gather feedback from developers and surveys
- At least 2 CNI provider must provide the implementation for the complete set
  of alpha features
- Evaluate "future work" items based on feedback from community

#### Beta to GA Graduation

- At least 4 CNI providers must provide the implementation for the complete set
  of beta features
- More rigorous forms of testing
  — e.g., downgrade tests and scalability tests
- Allowing time for feedback
- Completion of all accepted "future work" items

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

#### Upgrade considerations

As such, the cluster-scoped policy resources are new and shall not exist prior
to upgrading to a new version. Thus, there is no direct impact on upgrades.

#### Downgrade considerations

Downgrading to a version which no longer supports cluster-scoped policy APIs
must ensure that appropriate security rules are created to mimick the cluster-scoped
policy rules by other means, such that no unintended traffic is allowed.

### Version Skew Strategy

n/a

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

1.22:
- disable by default
- allow gate to enable the feature
- release note

1.24:
- enable by default
- allow gate to disable the feature
- release note

1.26:
- remove gate

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: ClusterNetworkPolicy
  - Components depending on the feature gate: kube-apiserver

###### Does enabling the feature change any default behavior?

Enabling the feature by itself has no effect on the cluster.
Creating a ClusterNetworkPolicy/ClusterDefaultNetworkPolicy does have an effect on
the cluster, however they must be specifically created, which means the
administrator is aware of the impact.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Once enabled, the feature can be disabled via feature gate. However, disabling
the feature may cause created cluster-scoped policy resources to be deleted,
which may impact the security of the cluster. Administrators must make provision
to secure their cluster by other means before disabling the feature gate.

###### What happens if we reenable the feature if it was previously rolled back?

n/a

###### Are there any tests for feature enablement/disablement?

n/a

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

Enabling this feature by itself will have no impact on the cluster and no
new API calls will be made.

###### Will enabling / using this feature result in introducing new API types?

Enabling this feature will introduce new API types as described in the [design](#design-details)
section. The supported number of objects per cluster will depend on the individual
CNI providers who will be responsible to provide the implementation to realize
these resources.

###### Will enabling / using this feature result in any new calls to the cloud provider?

n/a

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

n/a

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

n/a

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

n/a

## Implementation History

- 2021-02-18 - Created initial PR for the KEP

## Drawbacks

Securing traffic for a cluster for administrator's use case can get complex.
This leads to introduction of a more complex set of APIs which could confuse
users.

## Alternatives

Following alternative approaches were considered:

### NetworkPolicy v2

A new version for NetworkPolicy, v2, was evaluated to address features and use cases
documented in this KEP. Since the NetworkPolicy resource already exists, it would be
a low barrier to entry and can be extended to incorporate admin use cases.
However, this idea was rejected because the NetworkPolicy resource was introduced
solely to satisfy a developers intent. Thus, adding new use cases for a cluster admin
would be contradictory. In addition to that, the administrator use cases are mainly
scoped to the cluster as opposed to the NetworkPolicy resource, which is `namespaced`.

### Single CRD with DefaultRules field

We evaluated the possibility of solving the administrator use cases by introducing a
single resource, similar to the proposed ClusterNetworkPolicy resource, as opposed to
the proposed two resources, ClusterNetworkPolicy and ClusterDefaultNetworkPolicy. This alternate
proposal was a hybrid approach, where in the ClusterNetworkPolicy resource (introduced
in the proposal) would include additional fields called `defaultIngress` and
`defaultEgress`. These defaultIngress/defaultEgress fields would be similar in structure to
the ingress/egress fields, except that the default rules will not have `action` field.
All default rules will be "allow" rules only, similar to K8s NetworkPolicy. Presence of
at least one `defaultIngress` rule will isolate the `appliedTo` workloads from accepting
any traffic other than that specified by the policy. Similarly, the presence of at least
one `defaultEgress` rule will isolate the `appliedTo` workloads from accessing any other
workloads other than those specified by the policy. In addition to that, the rules specified
by `defaultIngress` and `defaultEgress` fields will be evaluated to be enforced after the
K8s NetworkPolicy rules, thus such default rules can be overridden by a developer written
K8s NetworkPolicy.

Adding default rules along with the stricter ClusterNetworkPolicy rules allows us to
satisfy all admin use cases with a single resource. Although this might be appealing,
separating the two broad intents of a cluster admin in two different resources makes
the definition of each resource much cleaner and simpler.

### Single CRD with IsOverrideable field

An alternative approach is to combine `ClusterNetworkPolicy` and `ClusterDefaultNetworkPolicy`
into a single CRD with an additional overrideable field in Ingress/ Egress rule
as shown below.

```golang
type ClusterNetworkPolicyIngress/EgressRule struct {
	Action        RuleAction
	IsOverridable bool
	Ports         []networkingv1.NetworkPolicyPort
	From/To       []networkingv1.ClusterNetworkPolicyPeer
}
```

If `IsOverridable` is set to false, the rules will take higher precedence than the
Kubernetes Network Policy rules. Otherwise, the rules will take lower precedence.
Note that both overridable and non overridable cluster network policy rules have explicit
allow/ deny rules. The precedence order of the rules is as follows:

`ClusterNetworkPolicy` Deny (`IsOverridable`=false) > `ClusterNetworkPolicy` Allow (`IsOverridable`=false) > K8s `NetworkPolicy` > `ClusterNetworkPolicy` Allow (`IsOverridable`=true) > `ClusterNetworkPolicy` Deny (`IsOverridable`=true)

As the semantics for overridable Cluster NetworkPolicies are different from
K8s Network Policies, cluster administrators who worked on K8s NetworkPolicies
will have hard time writing similar policies for the cluster. Also, modifying
a single field (`IsOverridable`) of a rule will change the priority in a
non-intuitive manner which may cause some confusion. For these reasons, we
decided not go with this proposal.

### Single CRD with BaselineAllow as Action

We evaluated another single CRD approach with an additional `RuleAction` to cover
use-cases of both `ClusterNetworkPolicy` and `ClusterDefaultNetworkPolicy`

In this approach, we introduce a `BaselineRuleAction` rule action.

```golang
type ClusterNetworkPolicyIngress/EgressRule struct {
	Action       RuleAction
	Ports        []networkingv1.NetworkPolicyPort
	From/To      []networkingv1.ClusterNetworkPolicyPeer
}
const (
	RuleActionDeny          RuleAction = "Deny"
	RuleActionAllow         RuleAction = "Allow"
	RuleActionBaselineAllow RuleAction = "BaselineAllow"
)
```

RuleActionDeny and RuleActionAllow are used to specify rules that take higher
precedence than Kubernetes NetworkPolicies whereas RuleActionBaselineAllow is
used to specify the rules that take lower precedence Kubernetes NetworkPolicies.
The RuleActionBaselineAllow rules have same semantics as Kubernetes NetworkPolicy
rules but defined at cluster level.

One of the reasons we did not go with this approach is the ambiguity of the term
`BaselineAllow`. Also, the semantics around `RuleActionBaselineAllow` is
slightly different as it involves implicit isolation compared to explicit
Allow/ Deny rules with other `RuleActions`.