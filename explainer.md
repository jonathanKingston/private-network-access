# Explainer: Private Network Access

## Quick links

- [Specification](https://wicg.github.io/private-network-access)
- [Repository](https://github.com/WICG/private-network-access)
- [Issue tracker](https://github.com/WICG/private-network-access/issues)

## Introduction

Private Network Access is a web specification which aims to protect websites
accessed over the private network (either on localhost or a private IP address)
from malicious cross-origin requests.

Say you visit evil.com, we want to prevent it from using your browser as a
springboard to hack your printer. Perhaps surprisingly, evil.com can easily
accomplish that in most browsers today (given a web-accessible printer
exploit).

This specification takes its name from
[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS), which
provides a mechanism for securing websites against cross-origin requests,
and [RFC 1918](https://tools.ietf.org/html/rfc1918), which describes IPv4
address ranges reserved for private networks.

## Goals

The overarching goal is to prevent malicious websites from pivoting through
the user agent's network position to attack devices and services which
reasonably assumed they were unreachable from the Internet at large, by
virtue of residing on the user’s local intranet or the user's machine.

For example, we wish to mitigate
attacks on:

- Users' routers, as outlined in
  [SOHO Pharming](https://www.team-cymru.com/ReadingRoom/Whitepapers/2013/TeamCymruSOHOPharming.pdf).
  Note that status quo CORS protections don’t protect against the kinds of
  attacks discussed here as they rely only on
  [CORS-safelisted methods](https://fetch.spec.whatwg.org/#cors-safelisted-method)
  and
  [CORS-safelisted request-headers](https://fetch.spec.whatwg.org/#cors-safelisted-request-header).
  No preflight is triggered, and the attacker doesn’t actually care about
  reading the response, as the request itself is the CSRF attack.
- Software running a web interface on a user’s loopback address. For better or
  worse, this is becoming a common deployment mechanism for all manner of
  applications, and often assumes protections that simply don’t exist (see
  [recent](https://code.google.com/p/google-security-research/issues/detail?id=679)
  [examples](https://code.google.com/p/google-security-research/issues/detail?id=693)).

## Non-goals

Provide a secure mechanism for initiating HTTPS connections to services
running on the local network or the user’s machine. This piece is missing to
allow secure public websites to embed non-public resources without running into
mixed content violations. While a useful goal, and maybe even a necessary one in
order to deploy Private Network Access more widely, it is out of scope of this
specification.

## Proposed design

### Address spaces

We extend the [RFC 1918](https://tools.ietf.org/html/rfc1918) concept of
private IP addresses to build a model of network privacy. In this model,
there are 3 main layers to an IP network from the point of view of a node,
which we organize from most to least private:

- Localhost - accessible only to the node itself, by default
- Private IP addresses - accessible only to the members of the local network
- Public IP addresses - accessible to anyone

We call these layers **address spaces**: `local`, `private`, and `public`.

The mapping from IP address to address space is defined in terms of the IANA
[IPv4](https://www.iana.org/assignments/iana-ipv4-special-registry/iana-ipv4-special-registry.xhtml)
and
[IPv6](https://www.iana.org/assignments/iana-ipv6-special-registry/iana-ipv6-special-registry.xhtml)
special-purpose address registries. Globally-routable IP addresses are `public`.
Details about the `private` vs. `local` distinction are under
[discussion](https://github.com/WICG/private-network-access/issues/15).

It might also be useful to consider web origins when determining address spaces.
For example, the `.local.` top-level DNS domain (see
[RFC 6762](https://tools.ietf.org/html/rfc6762)) might be always be considered
`private`. See this
[discussion](https://github.com/WICG/private-network-access/issues/4).

### Private network requests

The address space concept and accompanying model of IP address privacy lets us
define the class of requests we wish to secure.

We define a **private network request** as a request crossing an address space
boundary to a more-private address space.

Concretely, there are 3 kinds of private network requests:

1. `public` -> `private`
2. `public` -> `local`
3. `private` -> `local`

Note that `private` -> `private` is not a private network request, as well as
`local` -> anything.

### Integration with Fetch

Private network requests are handled differently than others, like so:

- If the client is not in a
  [secure context](https://www.w3.org/TR/secure-contexts/), the request is
  blocked. 
- Otherwise, the original request is preceded by a
  [CORS pre-flight request](https://fetch.spec.whatwg.org/#cors-preflight-request).
  - There are no exceptions for CORS safelisting.
  - The pre-flight request carries an additional
    `Access-Control-Request-Private-Network: true` header.
  - The response must carry an additional
    `Access-Control-Allow-Private-Network: true` header.

### Integration with HTML

`Document`s and `WorkerGlobalScope`s store an additional **address space**
value. This is initialized from the IP address the document or worker was
sourced from.

A new directive is introduced to
[CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP):
`treat-as-public-address`. If a document or worker's CSP list includes this
directive, then its address space is set to `public` unconditionally.

A new attribute is added to `Document` and `WorkerGlobalScope`:

```idl
enum AddressSpace { "local", "private", "public" };

partial interface Document {
  readonly attribute AddressSpace addressSpace;
};

partial interface WorkerGlobalScope {
  readonly attribute AddressSpace addressSpace;
};
```

It indicates the address space of the current context, and allows for feature
detection. It is not clear whether this is useful, and it might be harmful.
See discussion [here](https://github.com/WICG/private-network-access/issues/21).

### Integration with WebSockets

The initial handshake for private network requests is modified like so:

- The handshake request carries an additional
  `Access-Control-Request-Private-Network: true` header.
- The response must carry an additional
  `Access-Control-Allow-Private-Network` header.

## Alternatives considered

### Cover all cross-origin requests targeting the local network

Define **private network request** instead as: any request targeting an IP
address in the local or private address spaces, regardless of the requestor’s
address space.

This definition is strictly broader than the one we are currently working
with. It seems likely to cause more widespread breakage in the web ecosystem.
We would like to first launch the current version and consider this instead
as an interesting avenue for future work.

### Remove addressSpace IDL

Remove the `addressSpace` attribute from `Documents` and `WorkerGlobalScopes`.
This makes testing the specification through WPTs harder than
[it already is](https://github.com/web-platform-tests/wpt/issues/26166), since
we can only observe the changes to request behavior. On the other hand its
presence might leak some information about users to websites?

See discussion [here](https://github.com/WICG/private-network-access/issues/21).
