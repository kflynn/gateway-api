# GEP-XXX: Support Routing Method Selection for xRoute

* Issue: [#XXX](https://github.com/kubernetes-sigs/gateway-api/issues/XXX)
* Status: Provisional

(See status definitions [here](overview.md#status).)

## TLDR

When using an HTTPRoute, TCPRoute, etc., the implementor of the routing logic
(the _router_) needs to connect to one of the route's `backendRefs`. The user
creating the xRoute resource should be able to choose whether this connection
is made to the ClusterIP of the Service ("Service routing") or directly to an
endpoint IP ("endpoint routing").

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

(Add any additional document links. Again, we should try to avoid
too much content not in version control to avoid broken links)
