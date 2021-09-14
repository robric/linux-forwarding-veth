# linux-forwarding-veth
Investigation on linux native forwarding for pod routing.

## l3-only forwarding 

Model networking as a collection of point-to-point links and /32 routing.
All routing between pods is based on L3 forwarding even when in a same subnet. This is equivalent to the contrial l3-only forwarding mode.

## Test environment

Linux ubuntu 20.04.1 LTS.

## Test 1 - local pod networking

### Define two namespaces



