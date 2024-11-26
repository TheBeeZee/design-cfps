# CFP-34577: Per-service Load Balancing Algorithm Selection

**SIG: SIG-DATAPATH**
**Begin Design Discussion: 2024-08-28**
**Cilium Release:** 1.17
**Authors:** Aleksander Mistewicz \<amistewicz@google.com\>, Katarzyna Łach \<katarzynalach@google.com\>

## Summary

Add annotation `service.cilium.io/lb-algorithm` support to Cilium which will allow a user to select the load balancing algorithm on a per-service basis.

## Motivation

Before this CFP, a user needs to make a choice of load-balancing algorithm to
use in their cluster. There is no one-size-fits-all choice.

`cilium-agent` can be configured by setting `bpf-lb-algorithm` to use one of the following load balancing algorithms:

1. Maglev:
   1. compute [a hash](https://github.com/cilium/cilium/blob/v1.16.0-rc.2/bpf/lib/hash.h#L9-L17) based on Source Address, Source Port, Destination Port and Protocol
   2. select a `backend_id` from a list of repeated `backend_id`s with deterministic sequence and distribution by weight
2. Random:
   1. select a `backend_id` from a list of repeated `lb_{4,6}_service`s
      where each `backend_id` occurs once

| Algorithm | Static backend selection | Memory use (per service[^1]) | Memory use (total[^2]) |
| :---- | :---- | :---- | :---- |
| Maglev | \+ (as long as backends list is the same) | [65.5 KB \= 4B x 16381](https://github.com/cilium/cilium/blob/v1.16.0-rc.2/bpf/lib/lb.h#L168) (constant) | 655 MB (10 000 services) |
| Random | \- | 120 KB \= [12 B](https://github.com/cilium/cilium/blob/v1.16.0-rc.2/bpf/lib/common.h#L1053-L1067) x backends | 3 MB (250 000 endpoints) |

Memory footprint is smaller for `maglev` when there are between `5461` and `16381` endpoints in a service. There were two attempts to improve to improve `maglev`’s memory use in [cilium\#14396](https://github.com/cilium/cilium/issues/14396) and [cilium\#15019](https://github.com/cilium/cilium/pull/15019).

Similar requests that failed to gain traction:

1. [cilium\#8723](https://github.com/cilium/cilium/issues/8723) \- algorithm selection in service annotation
2. [cilium\#26264](https://github.com/cilium/cilium/issues/26264) \- more load-balancing algorithms

`kube-router` lets users [select a load-balancing algorithm based on service annotation](https://github.com/cloudnativelabs/kube-router/blob/v2.1.3/docs/user-guide.md#load-balancing-scheduling-algorithms).
At the time of this writing, there are [14 algorithms](https://keepalived-pqa.readthedocs.io/en/latest/scheduling_algorithms.html) to choose from.

## Goals

1. Support multiple load-balancing algorithms on a single node

## Non-Goals

1. Add support for new (other than `maglev`/`random`) load-balancing algorithms
2. Support migration without packet loss when enabling the mode to support this feature.

## Proposal

Add a new flag `bpf-lb-algorithm-configuration` with values `global` and `annotation`. Add a new `service.cilium.io/lb-algorithm` annotation which will allow Users to configure loadbalanicng algorithm on service level. It will initially accept `maglev` or `random` values.
If `bpf-lb-algorithm-configuration` = `annotation` controller will check the annotation on the service level. If annotation `service.cilium.io/lb-algorithm` exists then lodbalancing algorithm will be fetched from its value. If the annotation is not set on Service the default value will be taken from the `bpf-lb-algorithm` flag.

### Overview

1. Add new value `annotation` for the flag `bpf-lb-algorithm`
2. Add a new flagflag bpf-lb-algorithm-configuration with values global and annotation.
3. Rename `lb{4,6}_select_backend_id` functions to include algorithm name
   [code](https://github.com/cilium/cilium/blob/v1.16.0/bpf/lib/lb.h#L1310-L1332)
4. Compile eBPF with `LB{4,6}_MAGLEV_MAP_OUTER`
   (default or behind a new value of `bpf-lb-algorithm`)
   [code](https://github.com/cilium/cilium/blob/v1.16.0/bpf/lib/lb.h#L78-L94)
5. Add new uint8 field `lb_alg` to `lb{4,6}_service` hidden behind the config flag.
   [code](https://github.com/cilium/cilium/blob/v1.16.0/bpf/lib/common.h#L1064-L1065)
6. Pass the `lb-alg` value  based on `service.cilium.io/lb-algorithm` annotation
   to `lb{4,6}_service`
7. Diverge load-balancing code paths based on the flag set in `lb{4,6}_service`

#### Pros

1. Slow rollouts \- a different algorithm can be selected/tested without
   affecting the entire cluster
2. Extensible:
   1. A new algorithms can be introduced without affecting current behavior (opt-in)
   2. It is possible to achieve a feature-parity with `envoy`/`kube-router` in the future

#### Cons

1. Adding/Removing annotation to a service may be hard to handle (without disruption)
2. Agent would need to support all LB algorithms at the same time
   When compared to uniform-only clusters. It will have:
   1. Slower eBPF compilation
   2. Slower startup due to map precomputation
   3. More BPF maps need to be allocated

### Default behavior

In order to ensure that any disruptions to service load balancing are kept to minimum we will not support load balancing algorithms changes for already created services.

##### Alternative (not selected for implementation)

Add a new value, `bpf-lb-algorithm=all`, which will switch from the current behavior (of a single LB algorithm per cluster) to the one based on service annotations. In this case, the LB algorithm will default to `random` and could be changed to `maglev` only through setting `service.cilium.io/lb-algorithm=maglev` annotation.

### Transitions

In order to ensure that any disruptions to service load balancing are kept to minimum we will not support load balancing algorithms changes for already created services.
If User want to change loadbalancing algorithm then given Service needs to be recreated with a new annotation value.

## Critical User Journeys

### Upgrade from Cilium 1.16 to 1.17

* LB algorithm used doesn’t change
* `cilium-agent` may allocate additional maps (`LB{4,6}_MAGLEV_MAP_OUTER`) if needed
* User should not set `service.cilium.io/lb-algorithm` annotation before the upgrade

### Add service annotation

Service annotation should be added on Service creation. Migrating between lb algorithms is to risky and we will not support that.

### Update service annotation

If the User wants to change the value for `service.cilium.io/lb-algorithm` annotation, given Service should be deleted and recreated with a new annotation.

[^1]:  With 10 000 endpoints

[^2]:  With 10 000 services across 250 000 endpoints (set by `bpf-lb-service-map-max`)