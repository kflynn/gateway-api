# GEP-XXX: Support Routing Method Selection for xRoute

* Issue: [#XXX](https://github.com/kubernetes-sigs/gateway-api/issues/XXX)
* Status: Provisional

(See status definitions [here](overview.md#status).)

## Definitions

Throughout this GEP we'll refer to several named personas used to help
illustrate some of the ideas, and the roles associated with those ideas:

- Jane: the application developer
- Julian: the ops counterpart to Jane

## TLDR

When using `BackendRefs` in routes (e.g. `HTTPRoute`, `TCPRoute`, e.t.c.)
there may be a question as to how the `BackendRef` should be handled for
routing. At times it may be appropriate to resolve a `Service` to `Endpoints`
and let the `Gateway` load-balance those, or in some other situations it may be
appropriate to let some underlying implementation (e.g. `kube-proxy`, or a
service mesh) take over.

The purpose of this GEP is to propose API specification that makes it possible
to explicitly select how inbound traffic to an application will be routed.

## Goals

- Describe how Jane, Julian, etc. can *explicitly* select how traffic coming
  into the cluster should be routed to workload backends. (Currently, each
  implementation handles this in its own way, which may include a way to
  explicitly choose between routing modes, but may not. We want the Gateway
  API to provide a standardized mechanism for making this choice.)

- ?? should we have a goal around separation of concerns between Jane & Julian
  & Jasmine?

- ?? When, if ever, should Jane (need to) care about whether or not her
  routing is going to go through a mesh?

  - Flynn: I would argue that the Correct Answer™ is "never", but... it's a
    lovely question -- or, at least, Jane shouldn't always need to be aware.
    She might _choose_ to be aware in some cases...

  - Shane: Might we need to add some optionality to which role goes with which
    functionality?

- Shane: open question in my head right now: is it reasonable to have an
  officially-sanctioned spec for this kind of policy that can both be
  associated with an xRoute and as a policy attachment that can be associated
  with GatewayClass, Gateway, xRoute, etc.?

## Introduction

(Can link to external doc -- but we should bias towards copying
the content into the GEP as online documents are easier to lose
-- e.g. owner messes up the permissions, accidental deletion)

## API

(... details, can point to PR with changes)

## Alternatives

(List other design alternatives and why we did not go in that
direction)

## References

This concept was [briefly touched on][993] in an experimental PR (that
ultimately ended up being closed), due to a use case at Kong which is currently
handled via annotation.

[993]:https://github.com/kubernetes-sigs/gateway-api/pull/993#issuecomment-1013321918
