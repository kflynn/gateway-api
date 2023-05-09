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

- Provide a Gateway API standardized mechanism for _explicitly_ selecting how
  ingress traffic should be routed to workload backends.
- Help provide official guidance on which roles are at play for route method
  selection and how those roles apply.

## TODO

The following are concerns or issues that we want to see resolved as we move
forward with this GEP, _prior_ to graduating beyond `Provisional` status:

- Decide whether Jane (the developer) should ever specifically care about
  whether or not her applications are being routed through a mesh, and decide
  if we want some kind of _flexibility_ in roles to be at play here so as to
  better serve different use cases.
- Decide whether we're trying to do this at the route level only, or if we
  want to have this apply at the `GatewayClass` or `Gateway` levels.

## References

This concept was [briefly touched on][993] in an experimental PR (that
ultimately ended up being closed), due to a use case in [Kong's ingress
controller][kic] which is currently handled via annotation.

[993]:https://github.com/kubernetes-sigs/gateway-api/pull/993#issuecomment-1013321918
[kic]:https://github.com/kong/kubernetes-ingress-controller
