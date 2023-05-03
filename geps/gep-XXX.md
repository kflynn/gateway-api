# GEP-XXX: Support Routing Method Selection for xRoute

* Issue: [#XXX](https://github.com/kubernetes-sigs/gateway-api/issues/XXX)
* Status: Provisional

(See status definitions [here](overview.md#status).)

## TLDR

When using `BackendRefs` in routes (e.g. `HTTPRoute`, `TCPRoute`, e.t.c.) it's
possible for the referent object to insinuate an additional choice for routing
logic: that is to say it may be a question as to whether the `Gateway` itself
should resolve the `BackendRef` (e.g. in the case of `Service` by collecting
the `Endpoints` and using them directly) or if it would hand that choice off
to some underlying implementation (e.g. `kube-proxy`, or a service mesh).

The purpose of this GEP is to propose API specification that makes it possible
for Jane (the developer) to explicitly select how inbound traffic to her
application will be routed to her application.

## Definitions

- Jane: app dev

- Julian: ops counterpart of Jane

- Jasmine: probably unnecessary here, but who knows

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
