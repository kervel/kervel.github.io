---
layout: post
title:  "Cache coherency in VexRiscv"
date:   2020-10-14 20:45:21 +0200
categories: spinalhdl
---

I bought an [uxl3s](https://www.crowdsupply.com/radiona/ulx3s) with the goal of playing around with [spinalhdl](https://spinalhdl.github.io/). Since i read about cache coherency protocols recently, i tought it would be nice to figure out how the cache coherency mechanism works on the SMP-version of VexRiscv.

At first, i was completely overwhelmed by the complexity of VexRiscv. Luckily, i got some help from its creator on the gitter channel. VexRiscv cores have a small write-trough cache ([wikipedia](https://en.wikipedia.org/wiki/Cache_(computing)#Writing_policies) explains it nicely), which is generally simpler than a writeback cache.

Keeping write-trough caches coherent means invalidating cache lines on other cores when one core writes data to these cache lines.

# Cache invalidation streams in the banana memory bus (BMB)

A short introduction about the banana memory bus can be read [here](https://github.com/SpinalHDL/SaxonSoc). On the bottom of that readme, the signals that travel through the BMB are enumerated, in the documentation of the `cmd` and the `rsp` stream. 

When the optional cache invalidation feature is enabled (see [here](https://github.com/SpinalHDL/SpinalHDL/blob/dev/lib/src/main/scala/spinal/lib/bus/bmb/Bmb.scala#L462)), 3 more streams are added to the BMB bus:

- *inv* for cache invalidation requests
- *ack* for acknowledgement of cache invalidation (? is it ? *seems to contain no signals*)
- *sync* don't know what it is for

The *inv* stream carries the following signals

| Name    | Bitcount       | Description                                                                                        |
| ------- | -------------- | ------------                                                                                       |
| all     | 1              | ???                                                                                                |
| address | addressWidth   | The starting address of the memory block to be invalidated                                         |
| length  | invalidateLen  | The length of the memory block to be invalidated                                                   |
| source  | sourceWidth    | Transaction source ID, allow out of order completion between different sources, similar to AXI ID  | 

The *sync* stream carries the following signals

| Name    | Bitcount       | Description                                                                                        |
| ------- | -------------- | ------------                                                                                       |
| source  | sourceWidth    | Transaction source ID, allow out of order completion between different sources, similar to AXI ID  | 


# The connection from the cores to the data bus

In SaxonSoc the cores are connected to the BMB bus [here]https://github.com/SpinalHDL/SaxonSoc/blob/01f44c3a93a43e78478db4b6a0922c5a552dc702/hardware/scala/saxon/VexRiscvClusterGenerator.scala#L54). The relevant piece of code is:

```scala
    cores.produce(for(cpu <- cores.cpu) {
      interconnect.addConnection(
        cpu.iBus -> List(iBus.bmb),
        cpu.dBus -> List(dBusCoherent.bmb)
      )
    })
```

Here, all cores are connected to the same data bus, `dBusCoherent`. Note that the *produce* syntax is used here which is part of the [generator framework](https://spinalhdl.github.io/SpinalDoc-RTD/SpinalHDL/Libraries/generator.html). Basically, the variable `cores` here is not a list of cores but a list of core *handles*. Handles are basically area's that will become available at some point. The scala code block inside the `produce` call will then only be called when the `cores` structure has been generated.


