# MPW Pseudowire Project

## Overview

This project implements a Linux Ethernet pseudowire device called `mpw`, backed by kernel MPLS changes so that:

- **ingress MPLS packets** are delivered to the correct pseudowire by **local PW label lookup in MPLS core**
- **egress pseudowire traffic** is built by `mpw` as:
  - inner Ethernet frame
  - control word
  - remote PW label
  - optional outer transport label stack derived from **route MPLS encapsulation state**
- the `mpw` netdevice acts as the local **Ethernet-facing pseudowire endpoint**

The design started out more cluttered, with explicit transport labels and egress ifindex stored on the pseudowire itself, but it was gradually refactored into a much cleaner split:

- **MPLS core owns label dispatch and transport state**
- **MPW owns pseudowire service processing**
- **the `mpw` interface is mostly service-oriented**, not transport-oriented

At this point, the implementation supports:

- creating static MPW interfaces
- local PW-label registration in MPLS core
- real bidirectional Ethernet/IP traffic over the pseudowire
- PW-label-only adjacent-PE operation
- outer transport label imposition from route `encap mpls ...`
- readable LFIB output in iproute2 showing `local-pw dev mpw0`

---

## High-level architecture

### 1. Ingress path

The final ingress model is:

```text
MPLS packet arrives
-> MPLS core looks up top label in LFIB
-> if the label is a local PW label, MPLS local-PW action is taken
-> MPLS pops the matched PW label
-> MPLS passes the skb directly to the resolved MPW instance
-> MPW consumes [ CW ][ inner Ethernet ]
-> inner Ethernet frame is delivered upward
```

Important properties:

- ingress is no longer keyed by a transport label
- ingress is keyed by the **local PW label**
- MPW no longer does runtime PW-label lookup in the RX datapath
- MPLS core resolves the target PW instance and hands it off directly

This is the biggest conceptual cleanup that happened during the project.

### 2. Egress path

The final egress model is:

```text
inner Ethernet frame enters mpw0
-> MPW ensures skb starts at inner Ethernet header
-> MPW prepends:
   - remote PW label
   - control word
-> MPW calls mpls_pw_xmit()
-> mpls_pw_xmit() resolves the route to peer_ipv4
-> if the route carries MPLS encap state, that outer label stack is prepended
-> packet is transmitted as ETH_P_MPLS_UC
```

Important properties:

- `mpw` no longer stores `remote_transport_label`
- `mpw` no longer stores `egress_ifindex`
- egress transport comes from **normal Linux route/nexthop MPLS encap state**
- this makes route/LFIB/MPLS state the source of truth instead of a separate pseudowire-specific transport table

---

## Why the design changed

Originally the design had these concepts on the `mpw` interface:

- `pw_id`
- `local_vc_label`
- `remote_vc_label`
- `local_transport_label`
- `remote_transport_label`
- `peer_ipv4`
- `egress_ifindex`

That turned out to be too much transport detail living in the pseudowire interface itself.

The design was gradually refactored toward:

- `peer_ipv4`
- `local_pw_label`
- `remote_pw_label`
- optional `pw_id`

with transport handled elsewhere.

This aligns much better with the actual pseudowire model:

- the **PW label** is the service demultiplexer
- labels above that are **transport**
- local label dispatch belongs in MPLS core
- transport-imposition belongs to the routing/MPLS transport side

---

## What was changed in the kernel

### A. New local-PW action in MPLS core

A new internal MPLS route action was introduced for pseudowire delivery.

Conceptually:

- MPLS LFIB entries can now represent “this local MPLS label belongs to a pseudowire”
- the route stores:
  - action kind = local PW
  - callback ops
  - opaque private pointer

This lets MPLS core do real local-label dispatch instead of making MPW pretend to be the ingress MPLS logic.

### B. MPLS local-PW registration API

The kernel exports registration helpers that let MPW register and unregister a local PW binding in the MPLS platform-label table.

These functions now exist in MPLS core:

- `mpls_local_pw_register(struct net *net, u32 in_label, const struct mpls_local_input_ops *ops, void *priv)`
- `mpls_local_pw_unregister(...)`

These are used by the MPW RTNL layer when an interface is created and deleted.

### C. `struct mpls_local_input_ops`

This ops structure now supports:

- `input`
- `dump_info`
- `release`
- `owner`

The important addition was:

- `dump_info`

That is used only for userspace visibility. It lets the owner of the local-PW action supply additional route-dump metadata, such as `RTA_OIF`.

This avoided making `af_mpls.c` know about `struct mpw_priv`.

### D. `mpls_forward()` local-PW handling

`mpls_forward()` was changed so that when the resolved LFIB entry is a local-PW action:

- it does not follow normal forwarding
- it pops the matched PW label
- it calls the registered local-PW input callback with the resolved private object

This is what gave the clean handoff contract:

```text
MPW RX begins at [ CW ][ inner Ethernet ]
```

instead of the older transitional model where MPW had to parse and demux by PW label itself.

### E. Local-PW route allocation defaults

The helper that allocates the MPLS route for a local PW was cleaned up.

Originally, it produced route dump output like:

```text
local 100 local-pw dev mpw0 proto unspec ttl-propogate disabled
```

This came from route object defaults like:

- protocol unset
- TTL propagation explicitly disabled

That was improved so local PW routes now initialize more cleanly, using more meaningful/default route metadata.

Key changes:

- local-PW route dump type changed from `RTN_UNICAST` to `RTN_LOCAL`
- forwarding-oriented dead/linkdown flags were suppressed for local-PW routes
- local-PW route defaults were cleaned up so output is less misleading

### F. `mpls_pw_xmit()` redesign

This function changed a lot over time.

#### Early version
It took:

- `remote_transport_label`
- `peer_ipv4`
- `oif`

and manually pushed a single outer transport label.

#### Intermediate version
It stopped taking `oif` and used route lookup to reach the peer.

#### Final version
It takes only:

- `peer_ipv4`
- `ttl`

and derives transport-imposition from the resolved route’s `dst->lwtstate`.

That means:

- route lookup is done with `ip_route_output_key()`
- if the route has MPLS lwtunnel encap (`encap mpls ...`), the label stack is read from that route
- those transport labels are pushed outside the PW label
- if there is no MPLS route encap, PW-label-only egress is possible

This was a major architectural improvement because it made **route/MPLS state** the source of truth for transport labels.

---

## What was changed in the MPW module

### A. MPW ingress was simplified

Originally MPW RX still had logic to:

- parse the VC/PW label
- look up the pseudowire by label
- validate that label against the interface
- then consume CW and Ethernet

That logic was removed from the runtime ingress datapath.

Now MPW RX is direct:

- MPLS core chooses the target PW
- MPW RX only consumes:
  - control word
  - inner Ethernet frame

This significantly simplified `mpw_core.c`.

### B. Naming cleanup: VC -> PW

The older `vc` terminology was renamed to `pw` where it mattered.

Examples:

- `local_vc_label` -> `local_pw_label`
- `remote_vc_label` -> `remote_pw_label`

This makes the code much easier to follow.

### C. Transport fields were removed from `mpw`

Over time, the following were removed from the `mpw` configuration model:

- `local_transport_label`
- `egress_ifindex`
- `remote_transport_label`

This was the main API cleanup.

### D. TX path bug fix for real host-generated traffic

One of the most important late fixes was in the transmit path.

#### Symptom
Injected frames worked, but real IP traffic over `mpw0` failed. Underlay tcpdump showed garbage-looking MPLS label stacks because tcpdump was actually parsing the raw inner IP payload as if it were MPLS labels.

#### Root cause
The TX path was not preserving the inner Ethernet header correctly when encapsulating host-generated traffic. The skb still contained the right Ethernet frame at `mpw0`, but after service encapsulation the header offsets were wrong, and the wire packet effectively lost the PW label/CW in practice.

#### Fix
Two related changes solved this:

1. `mpw_ensure_inner_eth()`
   - makes sure TX operates on an skb whose `data` starts at the inner Ethernet header

2. after `mpw_encap_service_skb()`
   - reset skb header offsets to the current packet start

This fixed real host-generated Ethernet/IP traffic over the PW, including ARP.

This is why **real ping** started working, not just injected test frames.

### E. `mpw_encap_service_skb()`

This function now correctly prepends:

- remote PW label
- control word

and then resets header offsets so the rest of the transmit path sees a coherent packet.

### F. `mpw_xmit()`

The current TX path behavior is:

1. reject if shutting down or dying
2. call `mpw_ensure_inner_eth()`
3. remember original inner frame length
4. allocate next sequence number
5. call `mpw_encap_service_skb()`
6. call `mpls_pw_xmit()`
7. update TX stats

This is now the core of real PW egress.

---

## Current user-facing `mpw` API

The current `mpw` interface is service-oriented.

### Interface creation parameters

The effective model is now:

- `pw_id`
- `local_pw_label`
- `remote_pw_label`
- `peer_ipv4`

`pw_id` is still present and useful, especially for future control-plane integration.

Transport parameters are no longer part of the `mpw` interface.

### `mpw_link_tool`

The userspace helper was updated to match the simplified API and made more user-friendly.

Current add syntax:

```text
add <ifname> <pw_id> <local_pw_label> <remote_pw_label> <peer_ipv4>
```

Set supports natural updates of those fields.

Error messages were improved so it does not just print usage with no explanation.

Aliases were also tolerated during the transition in some stages, but the desired naming is PW-based, not VC-based.

---

## Current route/MPLS transport model

Transport is now provided by normal route/nexthop MPLS encapsulation state.

Example:

```bash
ip route replace 192.0.2.2/32 encap mpls 777 via 192.0.2.2 dev veth0
```

This means:

- route to `192.0.2.2` carries MPLS transport encap
- `mpls_pw_xmit()` resolves the route to the peer
- it reads MPLS lwtunnel state from the route
- it uses that outer label stack as transport

This replaced the earlier design where the pseudowire itself carried `remote_transport_label`.

This was explicitly tested by setting the route encap label to a value different from the old pseudowire transport label and confirming the packet used the **route-derived** label.

---

## Current LFIB / route visibility

### LFIB side
`ip -f mpls route show` now shows local PW bindings more clearly.

Through the kernel dump callback and iproute2 rendering change, a route now shows like:

```text
local 100 local-pw dev mpw0 ...
```

instead of only a generic route line.

This means userspace can now see:

- the local label
- that it is a pseudowire binding
- which interface owns it

### Route/iproute2 side
iproute2 was patched to identify local-PW LFIB entries and render them as `local-pw`.

The current match is still heuristic-based, because the kernel does not yet export a dedicated “this is local-pw” route-kind marker. But it is working.

One subtle point discovered during this work:

- the route dump sometimes still includes `RTA_MULTIPATH`, so the iproute2 detection had to be loosened
- the original matcher incorrectly required `!RTA_MULTIPATH`
- removing that requirement made `local-pw` rendering work

---

## What has been tested successfully

### 1. Basic create/delete/update
Creating and deleting `mpw` interfaces works.

Label uniqueness checks and unsupported field changes were handled in RTNL.

### 2. Ingress injection tests
Crafted MPLS packets sent into the underlay were successfully delivered to the pseudowire when labels matched.

Wrong labels were correctly rejected.

### 3. Egress injection tests
Sending inner Ethernet data through `mpw0` produced the expected MPLS encapsulated output.

### 4. Route-derived transport label test
By changing route `encap mpls ...` values and observing the outer label on the wire, it was proven that:

- egress transport label now comes from the route/MPLS transport state
- not from `mpw` config

### 5. Real two-ended pseudowire test
A real bidirectional pseudowire was brought up between:

- host namespace
- peer namespace `pe`

with one `mpw` endpoint on each side.

This was tested with:

- real IP addresses on the PW interfaces
- ordinary `ping`
- dynamic ARP
- real bidirectional ICMP

This is a major milestone because it proves the datapath works for real host-generated traffic, not just injected frames.

---

## Current test model

There are effectively two useful test modes now.

### 1. Injection-oriented test
Useful for deterministic label-path debugging.

- inject raw MPLS packets with expected labels
- verify ingress decapsulation and stats
- verify malformed or wrong labels fail as expected

### 2. Real PW connectivity test
Useful for validating actual L2 service behavior.

- two MPW endpoints
- IP on each side
- no static ARP needed anymore after the TX fix
- can run ordinary ping over the pseudowire

This is the more important “real world” validation.

---

## Important technical lessons learned

### 1. The PW label is the service anchor
A big conceptual improvement was realizing that the ingress binding should be on the **local PW label**, not the transport label.

That made the split clean:

- transport forwarding happens before the PW handoff
- MPLS core dispatches on the local PW label
- MPW receives CW + Ethernet

### 2. Transport state belongs to routing/MPLS state
Another major improvement was removing transport labels from the pseudowire config and deriving them from route MPLS encap state.

This avoids duplicating transport truth in a separate table.

### 3. Direct host-generated traffic exercises different bugs than injected frames
Injected frames made the early design look more correct than it really was. Real host-generated Ethernet/IP traffic found the header-offset bug in TX.

That was an important turning point in making the implementation robust.

---

## Current code organization and behavior

### MPLS core responsibilities

- own LFIB lookup
- own local-PW label dispatch
- expose registration API for local PW bindings
- dump enough route metadata for userspace visibility
- on egress, derive transport labels from route MPLS encap state

### MPW responsibilities

- own pseudowire interface state
- consume CW + inner Ethernet on RX
- prepend remote PW label + CW on TX
- maintain sequence handling
- maintain stats
- act as the local Ethernet-facing PW endpoint

---

## What is left to tackle

The architecture is now in a good place. Remaining work is mostly polish, visibility, correctness, and control-plane integration rather than foundational redesign.

### 1. Polish LFIB/iproute2 visibility further
The current output is much better, but still not perfect.

Possible remaining improvements:

- eliminate remaining misleading fields in output if any still appear
- reduce the heuristic nature of iproute2 detection by exporting a more explicit kernel-side route marker
- maybe improve JSON output too

### 2. Clean up debug logging
Temporary debugging hex dumps were useful and should be removed or reduced to ratelimited optional traces now that the datapath is working.

### 3. Improve documentation
This project now needs a short but clear “how it works” and “how to test it” document for future contributors.

The two most important documented workflows should be:

- adjacent-PE / PW-label-only static pseudowire
- PW with outer transport labels derived from route `encap mpls`

### 4. Add better self-tests
At this point, selftests would be very valuable.

Especially:

- create/delete PW
- wrong label rejection
- route-derived transport label selection
- real ping over the pseudowire
- ARP behavior
- optional control word / sequence tests

### 5. Control-plane integration
The design is finally in a shape that could support future integration with something like FRR.

The likely model would be:

- routing/MPLS daemon programs route/nexthop MPLS encap transport state
- some control-plane component programs MPW service state:
  - peer
  - local PW label
  - remote PW label
  - maybe PW ID

This is much more realistic now than earlier in the project.

### 6. Consider how explicit the control-plane/kernel contract should be
At some point you may want to decide whether `pw_id` remains only a stored identifier or becomes more meaningful in userspace interactions and diagnostics.

### 7. Optional deeper cleanup in route dump semantics
The current iproute2 rendering still depends on heuristics. A more explicit route-dump contract for local-PW actions would make the userspace side cleaner.

---

## Current practical workflow for a static PW

A contributor starting from here should think in these terms:

### Ingress
- create `mpw0` with local PW label `100`
- MPLS core registers local label `100` as a local-pw action
- packets arriving with top label `100` are delivered to `mpw0`

### Egress
- `mpw0` knows peer IP and remote PW label
- when sending, MPW prepends remote PW label and CW
- MPLS core transport-imposes outer labels from the route to the peer if present
- otherwise the packet can go out as PW-label-only in adjacent-PE mode

### Visibility
- `ip -f mpls route show` shows the local PW binding
- `ip route show` shows route/MPLS transport encap to the peer

That split is intentional and correct.

---

## Summary of the biggest milestones

In order of importance, the project’s biggest milestones were:

1. **Created a working MPW netdevice**
2. **Added local-PW label registration to MPLS core**
3. **Moved ingress binding from transport label to PW label**
4. **Refactored RX so MPLS core resolves the PW and MPW consumes only CW + Ethernet**
5. **Removed transport labels and ifindex from the pseudowire API**
6. **Moved egress transport-imposition to route/MPLS state**
7. **Fixed TX header handling so real host-generated Ethernet/IP traffic works**
8. **Verified real bidirectional ping over the pseudowire**
9. **Improved LFIB/iproute2 visibility so local PW label bindings are readable**

---

## Recommended immediate next steps

If someone were continuing this tomorrow, I would recommend:

1. remove temporary debug dumps
2. stabilize and clean the real-PW test script
3. document the static setup and architecture
4. add selftests