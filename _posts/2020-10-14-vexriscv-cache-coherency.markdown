---
layout: post
title:  "Cache coherency in VexRiscv"
date:   2020-10-14 20:45:21 +0200
categories: spinalhdl
---

This article is WIP and still contains incorrect stuff.

I bought an [uxl3s](https://www.crowdsupply.com/radiona/ulx3s) with the goal of playing around with [spinalhdl](https://spinalhdl.github.io/). Since i read about cache coherency protocols recently, i tought it would be nice to figure out how the cache coherency mechanism works on the SMP-version of VexRiscv.

At first, i was completely overwhelmed by the complexity of VexRiscv. Luckily, i got some help from its creator on the gitter channel. VexRiscv cores have a small write-trough cache ([wikipedia](https://en.wikipedia.org/wiki/Cache_(computing)#Writing_policies) explains it nicely), which is generally simpler than a writeback cache.

Keeping write-trough caches coherent means invalidating cache lines on other cores when one core writes data to these cache lines.

# The banana memory bus

The BMB bus is a *memory bus*. it connects two types of participants together:

- bus *masters* are entities that operate on the memory space (they write / read certain addresses), for instance CPU's
- bus *slaves* act upon these reads and write requests, for instance memories or GPIOs.

When there are multiple masters on the bus, an *arbiter* is needed to regulate which master has access to the bus at what time. 
This arbiter passes the read/write requests  from masters to the slaves and passes the responses back. When the masters have a cache, extra communication signals are created between the masters and the arbiter for cache coherency purposes.

A short introduction about the banana memory bus can be read [here](https://github.com/SpinalHDL/SaxonSoc). On the bottom of that readme, the signals that travel through the BMB are enumerated, in the documentation of the `cmd` and the `rsp` stream.

# Cache invalidation streams in the banana memory bus (BMB) 

When the optional cache invalidation feature is enabled (see [here](https://github.com/SpinalHDL/SpinalHDL/blob/dev/lib/src/main/scala/spinal/lib/bus/bmb/Bmb.scala#L462)), 3 more streams are added to the BMB bus:

- *inv* for cache invalidation requests (sent from the arbiter to all the masters)
- *ack* for acknowledgement of cache invalidation (sent from the bus master to the arbiter)
- *sync* to communicate to a bus master that the caches are in sync (or invalidated) after that master did a write request.

The *ack* stream has no signals apart from the *valid* and the *ready* signal of the Stream primitive.

The *inv* stream carries the following signals

| Name    | Bitcount       | Description                                                                                        |
| ------- | -------------- | ------------                                                                                       |
| all     | 1              | The bus arbiters sets this to 1 when this inv request was caused by a write by this source         |
| address | addressWidth   | The starting address of the memory block to be invalidated                                         |
| length  | invalidateLen  | The length of the memory block to be invalidated                                                   |
| source  | sourceWidth    | Transaction source ID, allow out of order completion between different sources, similar to AXI ID  | 

The *sync* stream carries the following signals

| Name    | Bitcount       | Description                                                                                        |
| ------- | -------------- | ------------                                                                                       |
| source  | sourceWidth    | Transaction source ID, allow out of order completion between different sources, similar to AXI ID  | 


# The cache coherency protocol

Since we are dealing with a write through cache, there is only one way a cache line can get outdated or invalid: by the write to the cached memory location by another core. The cache coherency protocol hence has to ensure that when one core writes to the memory, the right cache lines are invalidated.

*TODO add part about fence*

This will happen as follows:

* a bus master (eg a core) writes to a memory location
* the arbiter sends a *inv* request to all masters on the bus (with the information about the cache lines to be invalidated).
* the bus masters are then supposed to invalidate their cache lines and respond with an *ack* signal. Note that the ack signal has no source transaction ID, as there can be no two concurrent inv transactions.
* when the ack signal has been received from all masters, a *sync* transaction will be issued to the bus master that wrote to the memory location.





# The connection from the cores to the data bus

In SaxonSoc the cores are connected to the BMB bus [here](https://github.com/SpinalHDL/SaxonSoc/blob/01f44c3a93a43e78478db4b6a0922c5a552dc702/hardware/scala/saxon/VexRiscvClusterGenerator.scala#L54). The relevant piece of code is:

```scala
    cores.produce(for(cpu <- cores.cpu) {
      interconnect.addConnection(
        cpu.iBus -> List(iBus.bmb),
        cpu.dBus -> List(dBusCoherent.bmb)
      )
    })
```

Here, all cores are connected to the same data bus, `dBusCoherent`. Note that the *produce* syntax is used here which is part of the [generator framework](https://spinalhdl.github.io/SpinalDoc-RTD/SpinalHDL/Libraries/generator.html). Basically, the variable `cores` here is not a list of cores but a list of core *handles*. Handles are basically area's that will become available at some point. The scala code block inside the `produce` call will then only be called when the `cores` structure has been generated.

