# GEP-XXX: Support Routing Method Selection for xRoute

* Issue: [#XXX](https://github.com/kubernetes-sigs/gateway-api/issues/XXX)
* Status: Provisional

(See status definitions [here](overview.md#status).)

## TLDR

When using `BackendRefs` in routes (e.g. `HTTPRoute`, `TCPRoute`, e.t.c.) it's
possible for the referent object to insinuate an additional choice for routing
logic: that is to say it may be a question as to whether the `Gateway` itself
should resolve the `BackendRef` (e.g. in the case of `Service` by collecting
the `Endpoints` and using them directly) or if it would hand that choice off to
some underlying implementation (e.g. `kube-proxy`, or a service mesh).

The purpose of this GEP is to propose API specification that makes it possible
for Jane (the developer) to explicitly select how inbound traffic to her
application will be routed to her application.

## Goals

(Primary goals of this proposal.)

## Non-Goals

(What is out of scope for this proposal.)

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
