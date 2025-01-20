---
title: "Storage Notes"
date: 2016-06-25 10:00:00
tags: [distributed system, storage]
categories: [tech]
---



### Storage System Design Considerations

A storage system, used to save data, is often a fundamental service for many backend applications. Designing a storage system involves three critical considerations: **Data Safety**, **Performance**, and **Availability**.


#### Data Safety

Data safety is the most important aspect in most scenarios. The system must make every effort to protect user data, as any loss of data is unacceptable.


#### Availability

While availability may not always be an immediate concern, it remains essential for system stability. In modern storage systems, data is typically replicated across multiple nodes. If one node becomes unavailable, other nodes can handle requests. However, poor availability can lead to system instability, manifesting as increased latency or reduced throughput when alternate nodes are used.

One of the key factors affecting availability is the speed of stopping and restarting nodes, which impacts how quickly the system can recover from failures.


#### Performance

Performance is typically measured in two dimensions: **latency** (how quickly a system responds) and **throughput** (the amount of data the system processes). Performance is the second most important consideration and is influenced by data safety and availability. Optimizing performance requires careful design choices, ranging from hardware selection to user requirements.

##### Network Interface Card (NIC)

The NIC is a crucial determinant of network speed, with options like 1-Gb, 10-Gb, and Infiniband affecting overall system performance. For example, with 1-Gb NICs, two approaches are commonly used for replicating data to ensure safety:

1. **Star-Write**: Sends all three replicas simultaneously from one node to others. This approach minimizes latency but can saturate the client’s bandwidth, reducing throughput.
2. **Chain-Write**: Sends the first replica to one node, which then forwards it to others. This approach maximizes throughput by utilizing bandwidth across multiple nodes, but it introduces higher latency due to forwarding delays.

To mitigate these challenges, **two-phase commit** strategies can be employed:
- **Asynchronous writes**: Writes are first completed asynchronously and then retried until committed.
- **Commit phase**: Ensures data integrity while balancing throughput and latency.

With the widespread adoption of 10-Gb NICs, Star-Write can now achieve both low latency and high throughput, especially when integrated with a two-phase commit model.


##### Disk

Disks are the backbone of storage systems, and their configurations—such as HDD, SSD, SAS, RAID, and combinations—can significantly influence design and performance. Each choice has trade-offs in cost, speed, and durability.


##### Memory Hierarchy

Modern computer architecture relies heavily on a memory hierarchy. Efficient use of cache significantly affects performance. Cache misses introduce latency, making locality optimization critical. 

**Types of Locality**:

    **Spatial Locality**: Improves by ensuring data close in memory is cached together.
    - Larger cache sizes.
    - Memory indexing structures for data continuity.
    - Optimized binary layouts to improve instruction cache locality.

    **Temporal Locality**: Improves by reusing data or instructions over time.
    - Prefetching techniques.
    - Cache replacement algorithms.
    - Modular design: Breaking requests into self-contained modules connected by queues. This design enables the system to group similar operations, improving temporal locality. Advanced implementations like **stagedDB** ensure proper context-switching within modules, even when cache capacity is limited.

At the microarchitecture level, out-of-order processors help tolerate instruction misses by executing subsequent instructions. Many of these techniques rely on accurate prediction, which is constrained by unpredictable data sets and request patterns. However, good predictions can yield substantial performance benefits.

#### Stability Requirements

**Multitenancy**  
   When multiple customers share a storage system, ensuring fairness is critical. The system may implement priority levels to safeguard high-priority requests without starving low-priority ones. Effective multitenancy involves:
   - Fine-grained controls for individual requests.
   - End-to-end priority management across jobs and tenants.

**Graceful Degradation**  
   Graceful degradation ensures that when the system approaches maximum capacity, throughput remains steady, and latency increases proportionally. This requires the system to implement throttling mechanisms to reject excessive requests gracefully.
