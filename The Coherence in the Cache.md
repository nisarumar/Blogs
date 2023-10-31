# The Coherence in the Cache Part I

In this blog post, we will explore the term 'coherence' in the cache. The underlying assumption is that the basics of cache coherence protocols are clear. Just a recap of coherence protocols is as follows:


Consider the following sequence:
|Agent#1    |Agent#2    |epoch   |
|---        |---        |---     |
|A=0        |           |1       |
|Fence      |Fence      |2       |
|           |A=1        |3       |

The expectation of the programmer subscribing to any memory consistency model is that the memory of A will take on 0 first and then 1. However, in almost all the modern processors with their own memory (read: cache), without coherence, this might not be the case. Agent#1 might update its copy of the memory, and Agent#2 does its own. Should Agent#1 want to re-use A's value in its program order but fail to capture the
changes made by Agent#2. Hence, the basic idea of coherence is to **observe** the changes at a given memory location in the order they were committed. Notice the curious 'Fence' instruction; this is to ensure that the program order (before and after the fence) is respected by individual agents (cores). Discussion around the program order vs memory order or memory attributes is out of the scope of this blog.

In order to ensure the coherency of the memory, a coherence protocol is implemented, which helps ensure that changes to a certain memory location are observed in sequential order, the order they were committed by individual agents. Agents can be an entity with access to the memory and can modify the memory location. An agent can be a core with its own set of cache hierarchy, a LLC/memory controller, or a device that would like to leverage the benefit of hidden latency offered by the caches.

Typical coherence protocols are 'MSI', 'MESI', 'MOSI' and 'MESIF'. The basic idea of 'MSI' protocol is that a memory location can be in either a state of 'M', 'S' or 'I'. In all the types of protocol, somehow, a  stable state of a memory block is maintained across the agents. There are two categories of protocols: one is snooping, and the other is directory. A snooping protocol relies on the broadcast messages on the snoop interconnect, where a total order of the broadcasts is guaranteed on the interconnect. A directory protocol, on the other hand, is more scalable, as the requests need not be broadcast and typically are unicast with some multicast follow-ups. A snoop protocol utilizes fewer transactions on the snoop interconnect than a directory protocol so the tradeoff could be latency vs scalability. However, the cost incurred on a very high-speed interconnect is not very noticeable. The following quote is apt when the innovation brought on by directory protocol is considered.
> "All problems in computer science can be solved by another level of indirection"

## Coherence in Snooping Protocols

The above-mentioned states, 'MSI'for instance, are referred to as stable states. However, they make certain assumptions which are not realizable in the physical system. For instance, a state change from I->M indicates that the data was made available instantly. In the following, we will discuss some of the subtler points which are not considered while describing the coherence protocols:

### Split Coherence Request and Response Network:
For snooping protocol, it is important that the coherence request are observable to the agents in the same order. This requires some arbitration logic on the bus level to serialize the requests. On the other hand, data interconnect require wider data paths (64B cache line for instance); however, with a few exceptions they need not be coherent. Data responses might also take considerable time, especially for a block that has to be
fetched from a memory controller. Therefore, it makes sense to split a high-performance coherence network than the response network. This distinction calls for a state which is a transient state where a coherence request
has been observed by the agents, but data is still not received. Following the convention of [^1], we can refer to it as IS<sup>D</sup> or IM<sup>D</sup> or SD<sup>D</sup>, which basically says I->S; however waiting for
data. The status of this cache line is effectively the latter letter however the data transaction is still pending on the response network.

### Total Order of Coherence Requests:
It is trivial to show why per block ordering of coherency requests in a coherency fabric is important. A set of broadcasts, observed differently by a cache controller, might result in an incoherent state for the block. However, referring to Table 7.3 and Table 7.4 of [^1], will show how not observing the total order of coherency requests (irrespective of per block) might lead to violation of sequential consistency (SC) and total store order (TSO) models of memory consistency. Therefore, with the broadcast protocol as snooping, it is important that all the participating agents of coherency network should observe the coherency request
in the same order.

### Atomic Request
The basic model of coherency protocol also makes an assumption of _atomic_ request. It implies that as soon as a cache controller processes a certain coherency operation (for instance, GetS, GetM, PutM), it assumes that it is already ordered on the coherency bus. This assumption does not cater to the physical phenomenon of racing between two different cache controllers. For physical implementation, this constraint has to be removed. In order to relax this constraint, we can use the fact that cache controllers are snooping their own requests on the broadcast network. If a cache controller self-snoops its own request, then it
can be sure that the request has been ordered on the broadcast network. However, this introduces another transient state where a cache controller has processed the request but is waiting for it to be ordered on the 
network. Following conventions of [^1], this can be denoted as IS<sup>AD</sup> or IM<sup>AD</sup> or SD<sup>AD</sup> or MI<sup>A</sup>. Where the letters **IM** signify the intended transition to two stable states i-e I->M. _A_ in superscript signifies that waiting for the transaction to appear on the coherency network. As discussed previously, <sup>D</sup> signifies waiting for data. 

### Atomic Transaction
One other assumption by the basic coherency protocol is of _atomic_ transactions. It means that while a transaction is carried out for a certain block on the coherency network, no other transaction can proceed.
This has implications for the performance of the coherency fabric. An atomic transaction is where a request awaits a response before issuing a new transaction. A response can be from the off-chip DRAM memory, taking a longer time. Not processing another request while awaiting previous response seems rather inefficient; therefore snooping protocols consider alternative implementation. One such implementation could be a split transaction bus.

### Split Transaction Bus
A split transaction bus is where the data, and address/request bus are split and data/response bus can be out of order. It makes sense because some data could already be in one of the cache while some data has to be obtained from off-chip DRAM. Similarly, the subsequent requests do not have to wait for the completion of earlier response. Relaxing this constraint means that additional states need to be added. For instance, now
it is legal for a core to observe a GetM request for the same block while it is in IS<sup>D</sup> (shared, however, waiting for data). The simplest solution would be to stall the request. However, stalling sacrifices
some performance, also increases the risk of a potential deadlock, and could also allow the response to be observed before the request is observed by the same cache agent. The motivation to add IM<sup>A</sup> is due
to the last reason, where a controller sees the response before it sees its own request due to stalling on some other request. As the incoming requests are stored in the FIFO and if the controller's own outgoing
request is queued behind some stalling request in the arrival queue, it could be a possibility that the controller sees the data response before it sees its own request and be able to change the state to IM<sup>D</sup>. 

The following table is taken from [^1] (table 7.17), presenting a non-stalling, split transaction bus protocols. The first column of the table signifies the current state, while all the other columns are possible events which can happen while the memory block is in that state

|x                  |Load                   |Store                  |Replacement    |Own-GetS           |Own-GetM           |Own-PutM       |Other-GetS     |Other-GetM         |Other-PutM     |Own Data Response  |
|---                |---                    |---                    |---            |---                |---                |---            |---            |---                |---            |--                 |
|I                  |GetS/IS<sup>AD</sup>   |GetM/IM<sup>AD</sup>   |x              |x                  |x                  |x              |-              |-                  |-              |x                  |
|IS<sup>AD</sup>    |Stall                  |Stall                  |Stall          |-/IS<sup>D</sup>   |x                  |x              |-              |-                  |-              |-/IS<sup>A<sup>|               
|IS<sup>D</sup>     |Stall                  |Stall                  |Stall          |x                  |x                  |x              |-              |IS<sup>D</sup>I    |x              |Load hit/S         |
|IS<sup>A</sup>     |Stall                  |Stall                  |Stall          |Load Hit/S         |x                  |x              |-              |-                  |x              |x                  |
|IS<sup>D</sup>I    |Stall                  |Stall                  |Stall          |x                  |x                  |x              |-              |-                  |x              |Load hit/I         |
|IM<sup>AD</sup>    |Stall                  |Stall                  |Stall          |x                  |-/IM<sup>D</sup>   |x              |-              |-                  |-              |-/IM<sup>A</sup>   |
|IM<sup>D</sup>     |Stall                  |Stall                  |Stall          |x                  |x                  |x              |IM<sup>D</sup>S|IM<sup>D</sup>I    |x              |Store hit/M        |
|IM<sup>A</sup>     |Stall                  |Stall                  |Stall          |x                  |Store hit/M        |x              |-              |-                  |-              |x                  |
|IM<sup>D</sup>I    |Stall                  |Stall                  |Stall          |x                  |x                  |x              |-              |-                  |x              |Store hit/I        |
|IM<sup>D</sup>S    |Stall                  |Stall                  |Stall          |x                  |x                  |               |-              |-/IM<sup>D</sup>SI |x              |Store hit/S        |
|IM<sup>D</sup>SI   |Stall                  |Stall                  |Stall          |x                  |x                  |x              |-              |-                  |x              |Store hit/I        |
|S                  |Hit                    |Issue GetM/SM<sup>AD</sup>|-/I         |x                  |x                  |x              |-              |-/I                |x              |x                  |
|SM<sup>AD</sup>    |Hit                    |Stall                  |Stall          |x                  |-/SM<sup>D</sup>   |x              |-              |-/I                |x              |x                  |
|SM<sup>D</sup>     |Hit                    |Stall                  |Stall          |x                  |x                  |x              |SM<sup>D</sup>S|SM<sup>D</sup>I    |x              |Store hit/M     |      
|SM<sup>A</sup>     |Hit                    |Stall                  |Stall          |x                  |Store Hit/M        |x              |-              |-/IM<sup>A</sup>   |x              |X                  |
|SM<sup>D</sup>I    |Hit                    |Stall                  |Stall          |x                  |x                  |x              |-              |-                  |x              |Store hit/I        |
|SM<sup>D</sup>S    |Hit                    |Stall                  |Stall          |x                  |x                  |x              |-              |-/SM<sup>D</sup>SI |x              |Store Hit/S        |
|SM<sup>D</sup>SI   |Hit                    |Stall                  |Stall          |x                  |x                  |x              |-              |-                  |x              |Store hit/I        |
|M                  |Hit                    |Hit                    |PutM/MI<sup>A</sup>|x              |x                  |x              |Send data/S    |Send data/I        |x              |x                  |
|MI<sup>A</sup>     |Hit                    |Hit                    |Stall          |x                  |x                  |Send data/I    |Send Data/II<sup>A</sup>|Send Data/II<sup>A</sup>  |x|x                |
|II<sup>A</sup>     |Stall                  |Stall                  |Stall          |x                  |x                  |-/I            |-              |-                  |-              |x                  |

Here is a brief explanation of the states. The superscript identifies the transient states and events being waited on. For instance, <sup>AD</sup> specifies the two events being waited on are 1) Self snoop of the request and 2)Data response. The dual letters can mean different things based on superscript. If the superscript still has an <sup>A</sup> in it, it means that the request is not self-snooped and the state of the block
is effectively in the state denoted by the preceding letter. For instance, in the case of MI<sup>A</sup>, the state of the block is still M, which intends to transition to I. As you can see from the table above, in state MI<sup>A</sup>, the
controller still responds with data as it is the primary owner of the data. With just <sup>D</sup> superscript in a transient state, the state of the block is effectively in the state denoted by the latter letter, however, it still awaits data on the response bus. For instance, observing GetS while in state IM<sup>D</sup> transitions to IM<sup>D</sup>S, which can be described as currently in modified waiting for data (as this is non-stalling); once the data is there, sending the data to the owner of GetS and changing the state to shared. This state change can potentially induce the problem of livelock. A livelock is where for instance a core is in IM<sup>D</sup> and receives a GetS from another core. It changes to IM<sup>D</sup>S (currently in M, waiting on data once it arrives, sending it to the requestor of S), if in this state, the core does not perform a store and send the data as soon as it arrives then the same core might issue GetM again and the same situation can arise again and again. Therefore, to circumvent this, it is required for the core to perform either store or load in IS<sup>D</sup>I, IM<sup>D</sup>I, IM<sup>D</sup>S, or IM<sup>D</sup>SI and so on.

With the transient states such as SM<sup>D</sup>I, the controller has to remember the requestor of GetM in order to send the data to the requestor later on. For the states like SM<sup>D</sup>S, the controller has to
send the data to the requestor and the memory as it is no longer the owner of the block. Subsequent GetS requests from other agents might be entertained by the owner memory/LLC controller.

The following table is from the perspective of the LLC/memory controller:

|x                  |GetS                           |GetM                       |PutM from Owner                    |PutM from Non-Owner                |Data                       |
|---                |---                            |---                        |---                                |---                                |---                        |
|IorS               |Send data                      |Send data, change owner/M  |x                                  |-                                  |x                          |
|M                  |Clear owner/IorS<sup>D</sup>   |Set owner to requestor     |Clear Owner/IorS<sup>D</sup>       |-                                  |Write Data/IorS<sup>A</sup>|
|IorS<sup>D</sup>   |Stall                          |Stall                      |Stall                              |-                                  |Write data/IorS            |
|IorS<sup>A</sup>   |Clear Owner/IorS               |-                          |Clear Owner/IorS                   |-                                  |x                          |

The above table is taken from ref [^1] table 7.18.

One important thing to highlight here is that LLC has the owner field for each block. Consider an example where a block is in MI<sup>A</sup> state that is in M going to I waiting for own request on the bus. Before its own PutM request gets serialized on the bus, another GetM or GetS from another core gets serialized on the bus. Now, the owner is still the previous block, and it responds with the data to the requestor and
goes to II<sup>A</sup>. LLC has yet to see the PutM from the previous owner however it will see GetS or GetM from other agents and update the owner accordingly (in the case of GetS itself). Later on, PutM from the non-owner (or from the original owner) will be ignored, and LLC expects no data.

In the next part, we will explore directory protocols...


[^1]: A primer on Memory Consistency and Cache Coherence (Second Edition) by Vijay Nagarajan, Daniel J. Sorin, Mark D. Hill, David A. Wood