# GEP-713: Meta resources and Policy Attachment

* Issue: [#713](https://github.com/kubernetes-sigs/gateway-api/issues/713)
* Status: Standard

(See status definitions [here](/geps/overview/#gep-states))

## TLDR

This GEP aims to standardize terminology and processes around using one Kubernetes object to modify the functions of one or more other objects.

It lays out guidelines for Gateway API implementations and other stakeholders for the design and/or handling of custom “meta resources” definitions in compliance with a standard known as Policy Attachment.

This GEP specifies a _pattern_, not an API field or new object. It defines some terms, including _Meta resource_, _Policies_ and _Policy Attachment_, and their related concepts.

## Goals

* Establish a pattern for meta resources which will be used for any meta resources and policies included in the Gateway API spec.
* Establish a pattern that must be adopted for any implementation-specific meta resources and policies used with Gateway API resources.
* Discuss the problems with communicating status for meta resource objects, and suggest mechanisms that APIs can use to mitigate some of them.
* Provide a way to distinguish between required and default values for all meta resource API implementations.
* Enable Policy Attachment at all relevant scopes in Gateway API, including Gateways, Routes, Backends, along with how values should flow across a hierarchy if necessary.
* Ensure the Policy Attachment specification is generic and forward thinking enough that it could be easily adapted to other grouping mechanisms like Namespaces in the future.
* Provide a means of attachment that works for both ingress and mesh implementations of Gateway API.
* Provide a consistent specification that will ensure familiarity between both included and implementation-specific meta resources so they can both be interpreted the same way.
* Provide a reference pattern to other implementations of meta resource and policy APIs outside of Gateway API, that are based on similar concepts (i.e., augmenting the behavior of other Kubernetes objects, attachment points, nested contexts and inheritance, Defaults & Overrides, etc.)

## Out of scope

* Define all potential meta resource and policy kinds that may be attached to resources.

## Background

When designing Gateway API, it was found a frequent need to change the behavior of objects without being able to make changes to the spec of those objects. Sometimes, this is because changing the spec of the object to hold the new piece of information is not possible (e.g., `ReferenceGrant`, from [GEP-709](https://gateway-api.sigs.k8s.io/geps/gep-709/), when affecting Secrets and Services), and sometimes it’s because the behavior change is intended to flow across multiple objects (see [Semantics](#semantics-why) of meta resources and [Inherited](#inherited) class of meta resources).

To put this another way, sometimes we need ways to be able to affect how an object is interpreted in the API, without representing the description of those effects inside the spec of the object. This document describes the ways to design objects to meet use cases like these.

A new concept of “meta resource” has been introduced, a term used to describe the class of objects that _only_ augment the behavior of another Kubernetes object, regardless of what they are targeting.

“Meta” here is used in its Greek sense of “more comprehensive” or “transcending”, and “resource” rather than “object” because “meta resource” is more pronounceable than “meta object”.

After a few iterations of Gateway API experimenting with this pattern－both, with its own common kinds of meta resources such as the `BackendTLSPolicy`, as well as via multiple implementation-specific kinds of meta resources (see examples of [current use of meta resources](#current-use-of-meta-resources))－ and rounds of discussion such as the one at [kubernetes-sigs/gateway-api/discussions#2927](https://github.com/kubernetes-sigs/gateway-api/discussions/2927), the pattern has been enhanced to its current form.

## Definition

### Meta resources

*Meta resources* are objects that augment the behavior of other objects (*targets*) in a clean and standard way, by declaring additional specification for these targets “from the outside”－i.e., without directly modifying the objects whose behavior the meta resource intends to affect, nor requiring any reference from the objects to the meta resources that affect them.

Meta resources follow a well-known structure that declares clearly a *target* and an *intent*.

#### Meta resources are well-structured CRDs

Meta resources are typically implemented as Custom Resource Definitions (CRDs) that comply with a particular structure. This structure includes fields for specifying references to one or more other objects－called “targets“－whose behavior the instances of the meta resource intend to augment (i.e. declare additional specification), along with resource-specific fields－the “spec proper”－to describe the intended augmented behavior.

While the targets of a meta resource give the meta resource a *context*, the spec proper declares the *intent* within that context.

A typical meta resource looks like the following – example provided based on a hypothetical `ColorPolicy` kind of meta resource:

```yaml
apiVersion: policies.controller.io/v1
kind: ColorPolicy
metadata:
  name: my-color-policy
spec:
  targetRefs: ## target objects whose behaviour to augment
  - group: gateway.networking.k8s.io/v1
    kind: Gateway
    name: my-gateway
  rules:      ## the "spec proper" describing the intent, i.e. color the traffic blue
    color: blue
```

#### Properties of meta resources

##### Targetability (“where”)

Meta resources specify one or more target resources or specific sections of resources, whose behavior the meta resource intends to augment.

##### Semantics (“why”)

Targeting a resource (or section of a resource) must be interpreted within a given semantics that is proper to the meta resource kind.

Two different meta resource kinds that allow targeting resources of a same a given kind X may have very different semantics, not only because the purpose of the two meta resource kinds differ, but because the mechanics of calculating and applying the augment behavior for X can as well be very different between meta resource kinds.

Often, the semantics of a meta resource is tightly coupled to the relationships and connections a target has with other kinds of objects, typically organized in a hierarchy of nested contexts. In this sense, targeting a given resource kind may have the semantics of spanning effect across yet other objects to which the target is related.

##### Mergeability (“how”)

Meta resources specify so-called *merge strategies* that dictate how multiple instances of the meta resource affecting a same resource (or section of a resource) should be dealt with in terms of calculating and enforcing a single set of rules specified by the meta resources.

The merge strategies typically include strategies for dealing with conflicting and/or missing specs, such as for applying default and/or override values on the target resources.

### Policies and Policy Attachment

Often when applying a meta resource, the intent has to do with specifying *rules* that control the behavior of the targets in action. E.g., for networking resources, the augmented behavior specified by a meta resource targeting an object can relate to conditions to let traffic flow or to block traffic at particular points of the network; in a different scenario, it can describe rules to mutate the traffic while it flows through the network at the point represented by the target object.

The semantics of *applying rules to a target* characterizes these particular instances of meta resources as *policies*, thus here emerging in relation to this pattern the concept of *Policy Attachment*.

Policies are therefore a strict subset of meta resources which distinguish itself subtly from the whole on the fact that the augmented behavior described by policies resemble *conditional rules*, while the mechanism of targeting an object or set of objects (“attaching to objects”) is nonetheless exactly the same as in the general definition of meta resources.

Without loss of generality, the very activation of the context (targets) of meta resources can be understood as the _conditions_ for the enforcement of the intended augment behavior (rules) that are specified in the meta resources, and therefore these can often be refered to as policies interchangeably.

## Applications

### Reasons for using meta resources and policies

These are a few reasons for using meta resources and policies over another (possibly more direct) way to modify the spec (“augment the behavior”) of an object:

* Extending otherwise stable APIs – e.g. to specify additional network settings for the Kubernetes Service object.
* Defining implementation-specific functionalities for otherwise common APIs－e.g. to specify implementation-specific behavior for Gateway API HTTPRoute objects.
* Decoupling concerns for targeting personas with specific functionality and configuration－ delegation of responsibilities, fine-grained RBAC.
* Decoupling responsibility over the management and implementation of the meta resources themselves.
* Avoid alternatives based on annotations which are often non-standardized, poorly documented, and generally hard to maintain.

### Examples of applications of meta resources and policies

* Specifying traffic rules for Kubernetes Services
* Configuring implementation-specific authentication features on ingress Gateways
* Delegating responsibility over parts of the configuration (e.g. auth) to other users
* Inspecting and auditing focused aspects of a resource (e.g. auth policies)

## Guide-level explanation

### Defining context: scoping intent in relation to targets

The objects targeted by a meta resource define a *context* where the *intent* that is specified in the meta resource itself is expected to be honored. This context exists in the form of other Kubernetes objects (or parts of objects) and referenced in the meta resources directly or indirectly by name or other referencing mechanisms.

#### Ways of targeting objects

Meta resources can be designed to allow targeting objects by name (“reference by name”), using label selectors, and with or without cross-namespace references allowed. In all cases, the targets shall be declared within a `targetRefs` field within the spec of the meta resource instance.

##### Reference by name

The target reference includes the exact name of an object whose behavior to augment:

```yaml
apiVersion: policies.controller.io/v1
kind: ColorPolicy
metadata:
  name: my-color-policy
spec:
  targetRefs:
  - group: gateway.networking.k8s.io/v1
    kind: Gateway
    name: my-gateway ## name of the target object of Gateway kind
  rules:
    color: blue
```

##### Label selectors

The target reference includes the label selectors used to select a set of objects whose behavior to augment:

```yaml
apiVersion: policies.controller.io/v1
kind: ColorPolicy
metadata:
  name: my-color-policy
spec:
  targetRefs:
  - group: gateway.networking.k8s.io/v1
    kind: Gateway
    selector: ## label selectors to a set of objects of the Gateway kind
      matchLabels:
        env: production
  rules:
    color: blue
```

##### Cross namespace references

Meta resources can opt for allowing instances to target objects across Kubernetes namespaces, in which case an optional `namespace` field MUST be defined with the target reference.

Although not strictly forbidden, this is in general discouraged due to [discoverability](#the-discoverability-problem) issues and security implications. Implementations that opt for designing meta resources that allow for cross namespace references should consider supporting [ReferenceGrants](https://gateway-api.sigs.k8s.io/api-types/referencegrant/?h=referencegrant) to address the security concern.

#### Spanning behavior across relationships of a target

Because the target objects that give a meta resource context sometimes are themselves inserted into a broader context of their own, composed of other interrelated objects, the effects of a meta resource can be limited to a target object itself or span across the links between this object and other objects that the object is related to.

E.g. – a meta resource that targets a Namespace may declare intent that affects the behavior of the namespace itself (for what concerns to the implementation of Namespaces in Kubernetes) or alternatively it can act as a means to affect the behavior of other objects that exist in the referred namespace (e.g. ConfigMaps). While in the former case, the (direct) target object is the Namespace itself, in the latter the (indirect) target is a set of objects of a different kind (e.g. ConfigMaps.)

To avoid any ambiguity in the interpretation of the targets, meta resources MUST clearly define the extent of their effects respectively to the object kinds they target (semantics of attaching a meta resource). This is usually defined in terms of a known hierarchy of resource kinds.

See also: [Declared targets versus Effective targets](#declared-targets-versus-effective-targets) and [Hierarchy of target kinds and mechanics for calculating Effective meta resources](#hierarchy-of-target-kinds-and-mechanics-for-calculating-effective-meta-resources).

#### Narrowing the target to sections of an object

Meta resource CRDs can offer the option to target a section of an object whose spec defines sections uniquely identifiable by name. These meta resources typically include a field `spec.targetRefs.sectionName` that can be used along with compatible kinds.

E.g. – a meta resource that specifies additional behaviour for a given listener of a Gateway API Gateway object, though not for all listeners of the Gateway, MUST (i) require the Gateway listener to be uniquely named and (ii) provide the `sectionName` field of target reference with the name of the targeted listener.

#### Targeting virtual types

In some cases, one may want to apply meta resources to objects that are not actual Kubernetes kinds. An example of such, from Gateway API mesh case, would be a hypothetical need for defining a policy to "color" requests to external services. To accomplish this, implementations MAY choose to support a reference to a _virtual resource type_. E.g.:

```yaml
apiVersion: policies.controller.io/v1
kind: ColorPolicy
metadata:
  name: my-color-policy
spec:
  targetRef:
    group: networking.acme.io
    kind: ExternalService
    name: foo.com
  rules:
    color: blue
```

As a pattern, targeting virtual types has prior art in Kubernetes with the Role Based Access Control (RBAC), where Roles and ClusterRoles can used to specify permissions regarding any kind of resource including non-Kubernetes resources.

### Conflicting specs and Inheritance

Declaring additional specifications to objects from the outside can yield conflicts that need to be addressed in the implementation of meta resources. Multiple instances of a meta resource kind may affect an object (directly or indirectly), thus posing a possible conflict to resolve regarding which intent among the multiple meta resource specs a controller shall honor.

In some cases, the most recent between two conflicting specs may be desired to win, whereas in other cases it might be the oldest; often the winning spec is determined by the hierarchical level (implicit or explicit) of the context which the meta resource applies, and sometimes other criteria must be adopted to resolve conflicts between meta resources ultimately affecting a same target or section of a target.

The hierarchical relationships of the object that are targeted by meta resources – whether corresponding to their parent contexts or in relation to their inner sections – may yield indirect conflicts of specs (conflicting intents). Meta resource kinds that allow for their instances to target at multiple levels of a hierarchy of resource kinds (e.g. Gateway API `Gateway` and `HTTPRoute` kinds), entire resources as well as sections of a resource, or resources and filtered contexts of these resource kind (e.g. with `gatewayClassName`) will often generate cases where the behavior specified by the meta resource either is fully enforced or partially enforced, either honored or overridden by another.

Meta resource CRDs MUST clearly define the hierarchy of target resources they have effects upon, as well as the semantics of targeting each kind of resource in the hierarchy. Moreover, lower levels in the hierarchy *inherit* the definitions applied at the higher levels, in such a way that higher level rules may be understood as having an “umbrella effect” over everything under that level.

E.g., in Gateway API’s hierarchy of network resources for the ingress use case `GatewayClass` \> `Gateway` \> `HTTPRoute` \> `Backend`, a meta resource that attaches to a `GatewayClass` object, if defined as a meta resource kind ultimately to augment the behavior of `HTTPRoute` objects, affects all `Gateways` under the `GatewayClass`, as well as all `HTTPRoutes` under those `Gateways`. Any other instance of this meta resource kind targeting a lower level than the `GatewayClass` (e.g. `Gateway` or `HTTPRoute`) should be be treated as a conflict against the higher level meta resource spec, for the specific scope (“context”) of the subset of the hierarchy it attaches to. This conflict MUST be resolved according to some defined *merge strategy*.

### Declared targets versus Effective targets

A kind specified in the target reference of a meta resource can be the actual kind of object whose behavior the meta resource intends to augment or an indirection to targeting other kinds the object is hierarchically related to.

E.g. targeting a Gateway API `Gateway` object with a meta resource can be a way to augment the behavior of the `Gateway` object itself (e.g. reconcile cloud infrastructure provider settings from the spec declared by the `Gateway` according the rules specified by the meta resource attached to the `Gateway`) or a means to augment the behavior of all `HTTPRoute` objects attached to the `Gateway` (in a way that every new `HTTPRoute` that gets created or modified so it enters the context of the `Gateway` is automatically put in the scope of the meta resource.)

The target kinds specified in the target references of a meta resource are referred to as *Declared target* kinds. When declared targets are not equal to the actual targets augmented by the meta resource, but rather they serve as a means for reaching other levels (typically lower level) of related object kinds, these other kinds of target objects whose behavior are actually augmented by the meta resource are referred to as *Effective target* kinds.

### Hierarchy of target kinds and mechanics for calculating Effective meta resources

Target kinds MUST be organized in a well-known hierarchy of target kinds, from the least specific declared target kinds to the most specific effective ones. This hierarchy shall thus induce a logical representation of the relationships between instances of target objects, in the form of a Directed Acyclic Graph (DAG) whose roots are the least specific objects and the leaves are the most specific ones (and ultimately the effective targets of the meta resources).

The hierarchy of target kinds, as well as the logical representation of hierarchical relationships between instances of target objects, serve as a map for orderly resolving meta resource specs for each combinatorial context that is ultimately affected by the meta resources (i.e. for each Effective target). This process corresponds to resolving the intended augmented behavior that is collectively described by the set of meta resource specs in the context. The spec that summarizes the intended augmented behavior for a given context is referred to as the *Effective meta resource* (or *Effective policy*).

The process of calculating Effective meta resources (Effective policies) consists of walking the hierarchy of target objects, from most specific to least specific (i.e., bottom-up, from the leaves towards the roots of the DAG of target objects) or from least specific to most specific (top-down), map reducing to a single meta resource spec each pair of meta resources adjacent to each other in the hierarchy, applying at each step one of the supported merge strategies described below, until no more than one spec remains for each effective target.

Meta resource kinds that implement more than one merge strategy MUST provide fields for the instances of the meta resource to specify a chosen strategy (described in the next section). The least specific meta resource of the pair of meta resources whose specs are merged into one dictates the merge strategy to apply in such cases.

#### Conflict resolution

If multiple meta resources target the same context, this is considered to be a conflict.

Conflicts must be resolved by applying a defined *merge strategy* (see further definition in the next section), where the meta resource considered higher between two conflicting specs dictates the merge strategy according to which the conflict must be resolved, defaulting to the lower spec (more specific) beating the higher one if not specified otherwise.

The following criteria MUST be applied to determine which meta resources are higher or lower and thus infer which merge strategy to apply to calculate the effective spec, continuing on ties:

1. Between two meta resources at different levels of the hierarchy, the one attached higher in the hierarchy MUST be considered higher than the one attached lower in the hierarchy.
2. Between two meta resources at the same level of the hierarchy, the older meta resource based on creation timestamp MUST be considered higher than the newer one.
3. Between two meta resources at the same level of the hierarchy and identical creation timestamps, the meta resource appearing first in alphabetical order by `{namespace}/{name}` MUST be considered higher than the other.

#### Abstract process for calculating Effective meta resources

The following is a description of an abstract process for calculating effective meta resources.

Given:

* the target resource kinds `A`, `B` and `C`, organized in a hierarchy of resource kinds where `A` \> `B` \> `C`, i.e. `A` is the least specific kind (roots of the hierarchical tree) and `C` is the most specific kind (leaves of the tree)－without loss of generality for cases where these kinds are not necessarily proper Kubernetes kinds, but also possibly named sections of a proper Kubernetes kind;
* the meta resource kind `M`, whose instances can target resources of kind `A`, `B` or `C`, ultimately intending to augment the behavior of instances of resource kind `C`;
* the tree of targetable resources `a1` \> (`b1` \> `c1`, `b2` \> (`c1`, `c2`)), where `x` \> `Y` represents all the directed relationships from targetable resource `x` of kind `X` and its children, and recursively for `Y`, without loss of generality for any other set of instances of target resources;
* the instances of meta resource `m1` \-\> `a1` and `m2` \-\> `b2`, where `m` \-\> `y` represents the attachment of meta resource `m` of kind `M` to the target resource `y` of kind `A`, `B` or `C`, without loss of generality for any other set of instances of meta resources.

For each expanded context that is induced by the instances of targetable resource of kind `C` and its relationships given by the hierarchy, i.e. for each of: `a1` \> `b1` \> `c1`, `a1` \> `b2` \> `c1`, and `a1` \> `b2` \> `c2`, stack the meta resources targeting the context at any level, ordered from the most specific level (i.e. `C`) to the least specific one (i.e. `A`)－i.e. `A` is on top of the stack:

1. Pop two meta resources from the stack and combine them into one effective meta resource applying the conflict resolution rules (described in the previous subsection).
2. Push the effective meta resource back into the stack.
3. Repeat until there is no more than one meta resource in the stack.

The last meta resource in each stack (if any) specifies the intended augmented behavior for the effective target resource of kind `C` within that corresponding context.

For the exemplified instances, the expected outcome of the described process is:

* `c1` is augmented by `m1`, whenever activated in the context of `b1`;
* `c1` is augmented by the combination of `m1` \+ `m2`, whenever activated in the context of `b2`;
* `c2` is augmented by the combination of `m1` \+ `m2`.

The next section describes the different ways to combine meta resource instances (known as *merge strategiesI*), including a trivial merge strategy of not merging specs at all.

In the most trivial case where meta resources can only directly target the objects whose behavior they intend to augment (i.e. instances of `C` without any indirections) and no meta resource specs are merged at all, the outcome of the process of calculating effective meta resources is simplified to a 1:1 mapping between meta resource and target object at most, where the declared meta resource equals the effective one, with no combinatorial specs nor contextual variations.

### Merge strategies

#### Basic merge strategies

There are 3 *basic merge strategies*:

* **None:** the meta resource with the oldest creation timestamp that is attached to a target wins, while all the other meta resources attached to the same target are rejected (`Accepted` status condition set to false).
* **Defaults:** more specific specs beats less specific ones.
* **Overrides:** less specific specs beats more specific ones.

Meta resource CRDs may opt to implement any of these strategies, including multiple strategies.

Meta resource CRDs that implement more than one merge strategy MUST define a clear structure for the instances of meta resource to specify which of the supported strategies to apply. Instances of these meta resources MUST NOT be allowed to declare more than one merge strategy, but only one of the supported strategies at a time.

#### Atomicity of merging specs

Meta resource CRDs that implement specifically the Defaults or the Overrides base merge strategies SHOULD specify one or more *atomicity levels* to dictate how these base merge strategies must be applied:

* **Atomic spec:** the spec of the meta resource is treated as atomic, i.e., either one spec wins or another, but 2 specs are never mixed into a composition of specs. This is the default atomicity applied if not specified otherwise.
* **Scalar values (“Patch”):** the specs of 2 meta resources are merged into one by applying the winning spec (according to semantics dictated by the base merge strategy, i.e., the more specific if Defaults or the less specific one if Overrides) over the other spec, in a JSON patch operation.
* **\<Custom\>:** the spec of 2 meta resources are mixed into a composition of both specs, following a custom merge algorithm specified by the meta resource or policy kind.

#### Combined merge strategies

The final set of *merge strategies* therefore supported by a meta resource CRD (base \+ atomicity) is any subset of the following, where \<Custom\> is implementation-specific:

* None
* Atomic Defaults
* Atomic Overrides
* Patch Defaults
* Patch Overrides
* \<Custom\> Defaults
* \<Custom\> Overrides

Meta resource CRDs that support combined merged strategies are encouraged to define a `strategy` field for the instances to specify the exact strategy to apply.

### Classes of meta resources

The following two classes of meta resource kinds are defined: *Direct* and *Inherited*.

#### Direct

* A single kind supported in `spec.targetRefs.kind`
* Effects of the meta resources do not span across the hierarchy, i.e. the _Declared target kind_ is equal to the _Effective target kind_
* *None* is the only merge strategy supported
* If supported, could typically be implemented by directly extending the API of the target kind with the fields otherwise defined at the meta resource (e.g. Gateway API filter)

#### Inherited

* Superset of the above
* Any meta resource kind that do not comply with at least one characteristic of the Direct class of meta resources

## End-to-end examples

This section presents a series of synthetic examples of applications of meta resources for different kinds of topologies and contexts.

In all cases, the background of targetable object kinds is assumed to be a hierarchy of network resource kinds `Gateway` (`g`) \> `Route` (`r`) \> `Backend` (`b`), where `Gateway` is the least specific kind (instances denoted “`gX`”) and `Backend` is the most specific kind (instances denoted “`bX`”).

Moreover, a `ColorPolicy` meta resource kind (or “policy kind”, equivalently) is defined however with variations in its semantics across examples to accommodate for each case. Instances of the `ColorPolicy` kind (denoted “`pX[spec]`” and referred to simply as “policies”) may target one or more kinds of targetable resources, depending on each example. A policy represents an intent to “color” the network traffic that flows through the portion of the network corresponding to the target with a given color or color set that is specified in the policy.

### Example 1. Direct Policy

In this example, the `ColorPolicy` policy kind is defined as an instance of the Direct class of meta resources. Instances of the `ColorPolicy` kind in this example can only target `Backend` resources.

Given:

the following state of targetable resources:

* `g1` \> `r1` \> `b1`
* `g1` \> `r2` \> `b2`
  and the following state of `ColorPolicy` (`p`) policies, where `pX[spec]` \-\> `bX` denotes a policy `pX` attached to (“targeting”) a `Backend` resource `bX`, intending to augment `bX`‘s behavior with `spec`:

* `p1[color:red]` \-\> `b1`
* `p2[color:blue]` \-\> `b1` (conflicting policy, `p2.creationTimestamp` \> `p1.creationTimestamp`)

The expected outcome to be implemented by the controller is:

1. All traffic directed to `Backend` `b1` must be colored `red`.
2. Status of `Backend` `b1` should be reported as affected by the `ColorPolicy` `p1`.
3. Status of `Backend` `b2` should NOT be reported as affected by any policy.
4. Status of `ColorPolicy` `p1` must be reported as enforced.
5. Status of `ColorPolicy` `p2` must be reported as NOT enforced, due to conflict with `ColorPolicy` `p1`.

### Example 2. Defaults & Overrides

In this example, the `ColorPolicy` policy kind is defined as an instance of the Inherited class of meta resources. Instances of the `ColorPolicy` kind in this example can target resources of the `Gateway` and `Route` kinds, always aiming to augment the behavior of resources of the `Backend` kind in the hierarchy. The policies can specify either `defaults` (assumed unless specified otherwise) or `overrides`, that are always treated at the atomic level.

Given:

the following state of targetable resources:

* `g1` \> `r1` \> `b1`
* `g1` \> `r2` \> `b1`
* `g2` \> `r3` \> `b1`
* `g2` \> `r4` \> `b2`
  and the following state of `ColorPolicy` (`p`) policies, where `pX[spec]` \-\> `yX` denotes a policy `pX` attached to (“targeting”) a resource `yX`, `y` ∈ {`g`, `r`}, intending to augment with `spec` the behavior of `Backend` resources when activated via `yX`:

* `p1[color:red]` \-\> `g1`
* `p2[color:blue]` \-\> `r1`
* `p3[overrides:{color:yellow}]` \-\> `g2`
* `p4[color:green]` \-\> `r4`

The expected outcome to be implemented by the controller is:

1. Traffic directed to `g1` \> `r1` \> `b1` must be colored `blue` (more specific `p2` spec beats less specific defaults at `p1`).
2. Traffic directed to `g1` \> `r2` \> `b1` must be colored `red` (implicit defaults specified at `p1` not replaced by any other policy).
3. Traffic directed to `g2` \> `r3` \> `b1` must be colored `yellow` (overrides specified at `p3` not replaced by any other policy).
4. Traffic directed to `g2` \> `r4` \> `b2` must be colored `yellow` (overrides specified at `p3` beats more specific policy `p4`).
5. Status of `Backend` `b1` should be reported as affected by the `ColorPolicy` resources `p1`, `p2` and `p3`.
6. Status of `Backend` `b2` should be reported as affected by the `ColorPolicy` resource `p3`.
7. Status of `ColorPolicy` `p1` must be reported as partially enforced, due to in some cases beaten by `p2`.
8. Status of `ColorPolicy` `p2` must be reported as enforced.
9. Status of `ColorPolicy` `p3` must be reported as enforced.
10. Status of `ColorPolicy` `p4` must be reported as NOT enforced, due to being overridden by `ColorPolicy` `p3`.

### Example 3. Merged specs

In this example, the `ColorPolicy` policy kind is defined as an instance of the Inherited class of meta resources. Instances of the `ColorPolicy` kind in this example can target resources of the `Gateway` and `Route` kinds, always aiming to augment the behavior of resources of the `Backend` kind in the hierarchy. The policies can specify either `defaults` (assumed unless specified otherwise) or `overrides`. Moreover, policies specify a complex color scheme composed of `dark` and `light` entries, as well as a `strategy` field to specify one of two supported merge strategies, `atomic` (assumed unless specified otherwise) or `patch`.

Given:

the following state of targetable resources:

* `g1` \> `r1` \> `b1`
* `g1` \> `r2` \> `b1`
* `g2` \> `r3` \> `b1`
* `g2` \> `r4` \> `b2`
  and the following state of `ColorPolicy` (`p`) policies, where `pX[spec]` \-\> `yX` denotes a policy `pX` attached to (“targeting”) a resource `yX`, `y` ∈ {`g`, `r`}, intending to augment with `spec` the behavior of `Backend` resources when activated via `yX`:

* `p1[colors:{dark:brown,light:red},strategy:atomic]` \-\> `g1`
* `p2[colors:{light:blue}]` \-\> `r1`
* `p3[overrides:{colors:{light:yellow},strategy:patch}]` \-\> `g2`
* `p4[colors:{dark:olive,light:green}]` \-\> `r4`

The expected outcome to be implemented by the controller is:

1. Traffic directed to `g1` \> `r1` \> `b1` must be colored `dark:UNDEFINED,light:blue` (more specific `p2` spec beats less specific atomic defaults from `p1`.
2. Traffic directed to `g1` \> `r2` \> `b1` must be colored `dark:brown,light:red` (implicit atomic defaults specified at `p1` not replaced by any other policy).
3. Traffic directed to `g2` \> `r3` \> `b1` must be colored `dark:UNDEFINED,light:yellow` (patch overrides specified at `p3` not replaced, nor extended by any other policy).
4. Traffic directed to `g2` \> `r4` \> `b2` must be colored `dark:olive,light:yellow` (patch overrides specified by `p3` beats more specific policy `p4`, which still extends the spec with a specific value for `dark`.
5. Status of `Backend` `b1` should be reported as affected by the `ColorPolicy` resources `p1`, `p2` and `p3`.
6. Status of `Backend` `b2` should be reported as affected by the `ColorPolicy` resource `p3` and `p4`.
7. Status of `ColorPolicy` `p1` must be reported as partially enforced, due to in some cases atomically beaten by `p2`.
8. Status of `ColorPolicy` `p2` must be reported as enforced.
9. Status of `ColorPolicy` `p3` must be reported as enforced.
10. Status of `ColorPolicy` `p4` must be reported as partially enforced, due to being partially overridden by `ColorPolicy` `p3`.

## Managing meta resources in real life

### Responsibility

Meta resources are typically implemented and managed by a custom controller. This controller can be the same controller that is responsible for managing the objects that are targeted by the meta resources or another controller specifically responsible for the aspect of the object that the meta resource augments or modifies. For policy kinds of meta resources, this controller is often referred to as the “policy controller”.

Ultimately, it is the responsibility of the controller to provide enough information to resource owners that help circumvent or mitigate the discoverability problem (described in the next section). This typically involves populating the status stanza of the target objects, although may as well resort to additional tools (e.g. CRDs, CLI tools) that help visualize the hierarchical topology of target objects and meta resources, effective meta resource specs, etc.

### The discoverability problem

A well-known problem of declaring specifications into separate objects, that ultimately will reshape or govern the behavior of their targeted ones, regards the discoverability of meta resources. That is, how an object owner gets to know what meta resource (or set of meta resources) is affecting their object and with what content.

Even though Kubernetes already has analogous problems in its core－the most obvious example being the Kubernetes Role Based Access Control (RBAC)－, the discoverability issue remains a challenging one to be addressed. To better understand it, consider the following parable described in the context of Gateway API, with thanks to @flynn:

#### The Parabol

It's a sunny Wednesday afternoon, and the lead microservices developer for Evil Genius Cupcakes is windsurfing. Work has been eating Ana alive for the past two and a half weeks, but after successfully deploying version 3.6.0 of the `baker` service this morning, she's escaped early to try to unwind a bit.

Her shoulders are just starting to unknot when her phone pings with a text from Chihiro, down in the NOC. Waterproof phones are a blessing, but also a curse.

**Chihiro**: *Hey Ana. Things are still running, more or less, but latencies on everything in the `baker` namespace are crazy high after your last rollout, and `baker` itself has a weirdly high load. Sorry to interrupt you on the lake but can you take a look? Thanks\!\!*

Ana stares at the phone for a long moment, heart sinking, then sighs and turns back to shore.

What she finds when dries off and grabs her laptop is strange. `baker` does seem to be taking much more load than its clients are sending, and its clients report much higher latencies than they’d expect. She doublechecks the Deployment, the Service, and all the HTTPRoutes around `baker`; everything looks good. `baker`’s logs show her mostly failed requests... with a lot of duplicates? Ana checks her HTTPRoute again, though she's pretty sure you can't configure retries there, and finds nothing. But it definitely looks like clients are retrying when they shouldn’t be.

She pings Chihiro.

**Ana**: *Hey Chihiro. Something weird is up, looks like requests to `baker` are failing but getting retried??*

A minute later they answer.

**Chihiro**: 🤷 *Did you configure retries?*

**Ana**: *Dude. I don’t even know how to.* 😂

**Chihiro**: *You just attach a RetryPolicy to your HTTPRoute.*

**Ana**: *Nope. Definitely didn’t do that.*

She types `kubectl get retrypolicy -n baker` and gets a permission error.

**Ana**: *Huh, I actually don’t have permissions for RetryPolicy.* 🤔

**Chihiro**: 🤷 *Feels like you should but OK, guess that can’t be it.*

Minutes pass while both look at logs.

**Chihiro**: *I’m an idiot. There’s a RetryPolicy for the whole namespace – sorry, too many policies in the dashboard and I missed it. Deleting that since you don’t want retries.*

**Ana**: *Are you sure that’s a good–*

Ana’s phone shrills while she’s typing, and she drops it. When she picks it up again she sees a stack of alerts. She goes pale as she quickly flips through them: there’s one for every single service in the `baker` namespace.

**Ana**: *PUT IT BACK\!\!*

**Chihiro**: *Just did. Be glad you couldn't hear all the alarms here.* 😕

**Ana**: *What the hell just happened??*

**Chihiro**: *At a guess, all the workloads in the `baker` namespace actually fail a lot, but they seem OK because there are retries across the whole namespace?* 🤔

Ana's blood runs cold.

**Chihiro**: *Yeah. Looking a little closer, I think your `baker` rollout this morning would have failed without those retries.* 😕

There is a pause while Ana's mind races through increasingly unpleasant possibilities.

**Ana**: *I don't even know where to start here. How long did that RetryPolicy go in? Is it the only thing like it?*

**Chihiro**: *Didn’t look closely before deleting it, but I think it said a few months ago. And there are lots of different kinds of policy and lots of individual policies, hang on a minute...*

**Chihiro**: *Looks like about 47 for your chunk of the world, a couple hundred system-wide.*

**Ana**: 😱 *Can you tell me what they’re doing for each of our services? I can’t even* look *at these things.* 😕

**Chihiro**: *That's gonna take awhile. Our tooling to show us which policies bind to a given workload doesn't go the other direction.*

**Ana**: *...wait. You have to* build tools *to know if retries are turned on??*

Pause.

**Chihiro**: *Policy Attachment is more complex than we’d like, yeah.* 😐 *Look, how about roll back your `baker` change for now? We can get together in the morning and start sorting this out.*

Ana shakes her head and rolls back her edits to the `baker` Deployment, then sits looking out over the lake as the deployment progresses.

**Ana**: *Done. Are things happier now?*

**Chihiro**: *Looks like, thanks. Reckon you can get back to your sailboard.* 🙂

Ana sighs.

**Ana**: *Wish I could. Wind’s died down, though, and it'll be dark soon. Just gonna head home.*

**Chihiro**: *Ouch. Sorry to hear that.* 😐

One more look out at the lake.

**Ana**: *Thanks for the help. Wish we’d found better answers.* 😢

#### The Problem, restated

What this parable makes clear is that, in the absence of information about what meta resource is affecting an object, it’s very easy to make poor decisions.

It’s critical that this proposal solve the problem of showing up to three things, listed in increasing order of desirability:

* *That* some meta resource/policy is affecting a particular object
* *Which* meta resource/policy is (or meta resources/policies are) affecting a particular object
* *What* settings in the meta resource/policy are affecting the object.

In the parable, if Ana and Chihiro had known that there were Policies affecting the relevant object, then they could have gone looking for the relevant Policies and things would have played out differently. If they knew which policies, they would need to look less hard, and if they knew what the settings being applied were, then the parable would have been able to be very short indeed.

(There’s also another use case to consider, in that Chihiro should have been able to see that the meta resource on the namespace was in use in many places before deleting it.)

To put this another way, Meta resources and Policy Attachment is effectively adding another persona among the stakeholders, the Policy Admin, and without a solution to the discoverability problem, their actions are largely invisible to the Application Developer. Not only that, but their concerns cut across the previously established levels.

![Gateway API diagram with Policy Admin](images/713-the-diagram-with-policy-admin.png)

From the Policy Admin’s point of view, they need to know across their whole remit (which conceivably could be the whole cluster):

* *What* meta resource/policy has been created
* *Where* it’s applied
* *What* the resultant (effective) meta resource/policy is saying

Which again, comes down to discoverability, and can probably be addressed in similar ways at an API level to the Application Developer's concerns.

An important note here is that a key piece of information for Policy Admins and Cluster Operators is “How many things does this Policy affect?”. In the parable, this would have enabled Chihiro to know that deleting the Namespace Policy would affect many other people than just Ana.

#### Gateway API personas and the discoverability problem

Let's go through the various users of Gateway API and what they need to know about Meta resources affecting their objects.

In all of these cases, keeping the troubleshooting distance low is desired; that is, that there should be a minimum of hops required between objects from the one owned by the user to the one responsible for a setting.

Another way to think of the troubleshooting distance in this context is "How many `kubectl` commands would the user need to do to understand that a meta resource is relevant, which meta resource is relevant, and what configuration the full set of meta resource is setting?"

##### Application Developer Discoverability

How does Ana, or any Application Developer who owns one or more Route objects know that their object is affected by a meta resource, which meta resource is affecting it, and what the content of the meta resource is?

The best outcome is that Ana needs to look only at a specific route to know what meta resource settings are being applied to that Route, and where they come from. However, some of the other problems below make it very difficult to achieve this.

##### Policy Admin Discoverability

How does the Policy Admin know what meta resource is applied where, and what the content of that meta resource is? How do they validate that the meta resource is being used in ways acceptable to their organization? For any given meta resource object, how do they know how many places it's being used?

##### Cluster Admin Discoverability

The Cluster Admin has similar concerns to the Policy Admin, but with a focus on being able to determine what's relevant when something is broken.

How does the Cluster Admin know what meta resource is applied where, and what the content of that meta resource is?

For any given meta resource object, how do they know how many places it's being used?

#### Hinting on a solution for the discoverability problem

Querying the status of objects stored in the cluster may be the Kubernetes way of knowing the state of the system, in a world where objects are declarative and there are only so many links between objects to jump in between. However, this is still a proxy used to model a real life problem that often has otherwise different ways to be thought about as well.

In the context of traffic networking, for example, often the question asked by users is *“What happens when a network request X comes in?”*, stated in terms of a concrete “X”. There is an implicit expectation that a set of Kubernetes resources suffices to represent all the rules for a given workload to be activated and thus process request X, and often that is the case. For more complex cases however (multiple personas, application concerns separated into dedicated resource kinds, interaction between groups of users, etc), real life can get more complicated than a simple `kubectl get x`, or at least additional steps must be automated to encompass complexity into what can be achieved with a single declarative object.

With that in mind, a possible solution for the discoverability problem may involve designing tools (e.g. CLI tools/plugins), new CRDs, etc that let users ask questions in terms of the real life problems they have to deal with on a daily basis, rather than shaped by the underlying technologies used in the process. For instance, a simple Kubernetes object that is used to declare the rules to process a HTTP request cannot have its status reported simply as Ready/Not ready. By being a complex object composed of multiple routing rules, potentially affected by specifications declared from other objects as well, its status MUST account for that complexity and be structured in such a way that informs the owner with respect to each possible case, whether the ones induced by the internal specification declared by the object itself or its external relationships.

In other words, the discoverability problem exists and must be addressed in light of the complexity associated with the topology of contexts induced by a set of hierarchically related resources. One should always have that topology in mind while asking questions regarding the behavior of a given resource, because just like a routing object (e.g. HTTPRoute) does not exist independently from its parent contexts (e.g. Gateways) or its children (e.g. Backends), any resource in focus may be just a part of a whole.

### Status reporting

#### Meta resource status

Meta resource CRDs should define a status stanza that allows for reporting the status of the meta resource with respect to each context the resource may apply.

The basic status conditions are:

* **Accepted**: the meta resource passed both syntactic validation by the API server and semantic validation enforced by the controller, such as whether the target objects exist.
* **Enforced**: the meta resource’s spec is guaranteed to be fully enforced, to the extent of what the controller can ensure.
* **Partially enforced**: parts of the meta resource’s spec is guaranteed to be enforced, while other parts are known to have been superseded by other specs, to the extent of what the controller can ensure. The status should include details highlighting which parts of the meta resource are enforced and which parts have been superseded, with the references to all other related meta resources.
* **Overridden**: the meta resource’s spec is known to have been fully overridden by other specs. The status should include the references to the other related meta resources.

#### Target object status

Implementations of meta resources MUST put a condition into `status.Conditions` of any objects affected by the meta resource.

That condition MUST be named according to the pattern `<meta-resource-kind>Affected` (e.g. `colors.controller.k8s.io/ColorPolicyAffected`), and have the optional `observedGeneration` field kept up to date when the spec of the target object changes.

Implementations should use their own unique domain prefix for this condition type. Gateway API implementations, for instance, should use the same domain as in the `controllerName` field on `GatewayClass` (or some other implementation-unique domain for implementations that do not use `GatewayClass`.)

For objects that do not have a `status.Conditions` field available (`Secret` is a good example), that object SHOULD instead have an annotation of `colors.controller.k8s.io/ColorPolicyAffected: true` added instead.

#### Status needs to be namespaced by implementation

Because an object can be affected by multiple implementations at once, any added status MUST be namespaced by the implementation.

In Gateway API's Route Parent status, `parentRef` plus the controller name have been used for this.

For a meta resource, something similar can be done, namespacing by the reference to the implementation's controller name.

Namespacing by the originating meta resource cannot easily be done because the source could be more than one meta resource object.

#### Creating common data representation patterns

Defining a _common_ pattern for including the details of an _arbitrarily defined_ object, to be included in a library for all possible implementations, is challenging, to say the least.

Structured data cannot be used because there is no way of knowing what the structure will be beforehand. This suggests a need to use unstructured data for representing the main body of the arbitrary meta resource objects. Practically, this will have to be a string representation of the YAML form (or JSON, equivalently) of the body of the meta resource object (absent the metadata part of every Kubernetes object).

Meta resources and Policy Attachment does not mandate anything about the design of the object's top level except that it must be a Kubernetes object, so the only possible thing to rely upon here is the presence of the Kubernetes metadata elements: `apiVersion`, `kind`, and `metadata`.

Therefore, a string representation of the rest of the file is likely the best that can be done here.

#### Fanout status update problems

The fanout problem is that, when an update takes place in a single object (a meta resource, or an object with a meta resource attached), an implementation may need to update _many_ objects if it needs to place details of what meta resource applies, or what the resultant set of policy is on _every_ object.

Historically, this is a risky strategy and needs to be carefully applied, as it's an excellent way to create apiserver load problems, which can produce a large range of bad effects for cluster stability.

This does not mean that nothing at all that affects multiple objects can be done, but that careful consideration of what information is stored in status, so that _every_ meta resource update does not require a corresponding status update, is advised.

## Current use of meta resources

### Implementations

These are a few known implementations of meta resources in compliance with this standard:

#### Gateway API (core)

Gateway API defines two kinds of Direct meta resources (Direct policies), both for augmenting the behavior of Kubernetes `Service` resources:

* **BackendTLSPolicy:** Direct policy type for specifying the TLS configuration of the connection from the Gateway to a backend pod (set of pods) via the Service API object.
* **BackendLBPolicy:** Direct policy for Session Persistence (Experimental).

#### Envoy Gateway

<small>https://gateway.envoyproxy.io/docs/api/extension_types/</small>

Gateway API implementation that defines the following kinds of meta resources:

* **ClientTrafficPolicy:** to configure the behavior of the connection between the downstream client and Envoy Proxy listener.
* **BackendTrafficPolicy:** to configure the behavior of the connection between the Envoy Proxy listener and the backend service.
* **EnvoyExtensionPolicy:** to configure various envoy extensibility options for the Gateway.
* **EnvoyPatchPolicy:** to modify the generated Envoy xDS resources by Envoy Gateway using this patch API.
* **SecurityPolicy:** to configure various security settings for a Gateway.

#### Istio

<small>https://istio.io/latest/docs/reference/config/</small>

Gateway API implementation that defines the following kinds of meta resources:

* **EnvoyFilter:** to customize the Envoy configuration generated by istiod, e.g. modify values for certain fields, add specific filters, or even add entirely new listeners, clusters.
* **RequestAuthentication:** to define request authentication methods supported by a workload.
* **AuthorizationPolicy:** to enable access control on workloads in the mesh.
* **WasmPlugin:** to extend the functionality provided by the Istio proxy through WebAssembly filters.
* **Telemetry:** defines how telemetry (metrics, logs and traces) is generated for workloads within a mesh.

#### NGINX Gateway Fabric

<small>https://docs.nginx.com/nginx-gateway-fabric/overview/gateway-api-compatibility</small>

Gateway API implementation that supports Gateway API’s `BackendTLSPolicy` as well as the following kinds of meta resources:

* **ClientSettingsPolicy:** Inherited policy to configure connection behavior between client and NGINX.
* **ObservabilityPolicy:** Direct policy to define settings related to tracing, metrics, or logging.

#### Gloo Gateway

<small>https://docs.solo.io/gateway/latest/about/custom-resources/#policies</small>

Gateway API implementation that defines the following kinds of meta resources:

* **ListenerOption:** to augment behavior of one, multiple, or all gateway listeners.
* **HTTPListenerOption:** to augment behavior of one, multiple, or all HTTP and HTTPS listeners.
* **RouteOption:** to augment behavior of one, multiple, or all routes in an HTTPRoute resource.
* **VirtualHostOption:** to augment behavior of the hosts on one, multiple, or all gateway listeners.

#### Kuadrant

<small>https://docs.kuadrant.io</small>

First Gateway API integration entirely based on the Meta resources and Policy Attachment pattern. Defines the following kinds of meta resources:

* **DNSPolicy:** to manage the lifecycle of DNS records in external DNS providers such as AWS Route53, Google DNS, and Azure DNS.
* **TLSPolicy:** to manage the lifecycle of TLS certificate configuration on gateways using CertManager.
* **AuthPolicy:** Inherited policy with full support of Defaults & Overrides and merge strategies that can be attached to gateways and routes to specify authentication and authorization rules.
* **RateLimitPolicy:** Inherited policy with full support of Defaults & Overrides and merge strategies that can be attached to gateways and routes to specify rate limiting rules.

### Other meta resource-like implementations

#### Network Policy API (Working Group, SIG-NETWORK)

<small>https://network-policy-api.sigs.k8s.io/</small>

Defines two kinds of meta resources respectively for specifying *default* and *override* of networking policy rules: **AdminNetworkPolicy** and **BaselineAdminNetworkPolicy**. Builds on top of Kubernetes core `NetworkPolicy` kind.

Although the Network Policy API custom resources do not strictly implement the Meta resources and Policy Attachment pattern, they are based on similar concepts that involve policy rules for augmenting the behavior of other Kubernetes objects (pods), attachment points, nested contexts (through namespaces and pod selectors), and Defaults & Overrides.

#### Open Cluster Management

<small>https://open-cluster-management.io/docs/getting-started/integration/policy-controllers/policy-framework/</small>

Does not implement Meta resources and Policy Attachment. However, defines a virtual policy kind (**ConfigurationPolicy**) and supports distributing other third-party kinds of policies such as Gatekeeper's **ConstraintTemplate** kind, via a **Policy** resource whose targets are nonetheless controlled by a separate set of resource (**Placement** and **PlacementBinding**).

## Tools

The following tools can be useful for implementing and supporting meta resources and meta resource custom controllers.

#### gwctl

<small>https://github.com/kubernetes-sigs/gwctl</small>

CLI tool for visualizing and managing Gateway API resources in a Kubernetes cluster. Includes commands to visualize effective policies affecting the resources in compliance with the Meta resources and Policy Attachment pattern.

#### policy-machinery

<small>https://github.com/Kuadrant/policy-machinery</small>

Golang library for implementing policy controllers. Defines types and functions to build Directed Acyclic Graphs (DAG) to represent hierarchies of targetable resources and attached meta resources, calculate effective policies based on standard and custom merge strategies, etc. Includes helpers for applications based on Gateway API.

## References

**Issues**
* [Extensible Service Policy and Configuration](https://github.com/kubernetes-sigs/gateway-api/issues/611)

**Docs**
* [Policy Attachment and Binding](https://docs.google.com/document/d/13fyptUtO9NV_ZAgkoJlfukcBf2PVGhsKWG37yLkppJo/edit?resourcekey=0-Urhtj9gBkGBkSL1gHgbWKw)
