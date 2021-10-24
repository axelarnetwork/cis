---
title: Blockchain Packet Protocol (BPP)
author: Sergey Gorbunov @sergeynog
discussions-to: <URL>
status: Draft
type: <Standard>
created: 2021-10-07 
---

## Summary

Multiple bridging technologies and protocols are being developed in the ecosystem in parallel. 
What's common between them? They all rely on relayers to propagate information from one chain to another. This proposal standardizes the blockchain packet that can be relayed from one chain to another. 

Any blockchain can deploy routers or gateways (implemented as smart contracts or at the consensus layer or event queues) that receive packets in the format proposed below. Any relayer can pick up packets from the gateway queues and deliver them to destination chains. Interoperability, bridging, and application-level protocols sit on top of the blockchain packet protocol and can write to their local chain's gateways to communicate with remote chains. 

## Motivation

Our objective is to: 

a) make it easy to develop new interoperability or bridging protocols across distinct blockchains while,

b) unifying and leveraging a common relay packet protocol format and relayer infrastructure to send packets from one chain to another. 

Without this standard, every bridging protocol will need to roll out its own swarm of relayers just to post messages from one chain to another.  

## Specification

BPP is not a reliable protocol. Packets may be dropped or delivered multiple times. 
Bridging protocols on top of it are responsible for ensuring reliable or sequential reconstruction of packets. 

| Variable Type | Variable Name | Description |
| --- | --- | --- | 
| <span style="color:blue">`uint`</span>| `verion` | The protocol version number |  
| <span style="color:blue">`uint`</span>| `remaining_hops` | We do not want packets to circulate forever from one hop to the next (potentially ending up in a loop). Each time a packet is relayed, the number of remaining hops is decremented by one. Initially, it's set to the maximum number of chain hops the sender decides is reasonable (this may depend on the fee the sender pays to the relayer) |
|<span style="color:blue">`uint`</span>|`flow_type` | This is an informational field that may help relayers process the traffic. For instance, asset transfers vs general messaging passing packets might be handled differently by the relayers. A separate list of flow types will be recorded (asset transfers, NFT transfers, general message passing, etc.) | 
| <span style="color:blue">`uint`</span>|`src_chain_id_len` | Byte length of the source chain id that follows | 
|<span style="color:blue">`byte[]`</span>|`src_chain_id` | Source chain identifier. Following [CAIP-2](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md) | 
| <span style="color:blue">`uint`</span>|`src_address_len`| Byte length of the sender address | 
| <span style="color:blue">`byte[]`</span>|`src_address` | Sender of the packet (maybe be empty in some cases where irrelevant.  Typically the sender/receiver will be a higher level interoperability protocol (potentially implemented at the consensus or smart contracts) that is responsible for ensuring a reliable data stream between applications.(chain_id and address may be merged following [CAIP-10](https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md)) | 
| <span style="color:blue">`uint`</span>|`dst_chain_id_len` | Byte length of the destination chain identifier | 
| <span style="color:blue">`byte[]`</span>|`dst_chain_id` | Destination chain identifier | 
| <span style="color:blue">`uint`</span>|`dst_address_len`| Byte length of the destination address | 
| <span style="color:blue">`byte[]`</span>|`dst_address` | Destination address | 
| <span style="color:blue">`uint`</span>|`payload_len`| Byte length of the payload | 
| <span style="color:blue">`byte[]`</span>|`payload` | Payload |

* uint = 4 bytes. All other fields are variable length. 
* (TO DISCUSS) Adding a signature under the sender's address where applicable: some packets might be signed under the sender's address. This ensures that no forged packets may be created.

## Rationale

- In a multi-chain world, relaying packets from one network to another is a basic requirement. Regardless of the underlying interoperability protocol, relaying packets should be a standard and unified operation. 

- **Who pays the the transaction fees?**
Any relay provider can register a gateway "smart contract" on-chain (or module at consensus layer or introduce an event queue) to accept incoming packets of the form above. It may set its fee mechanism, DDOS protection techniques, or relayer policies. The callers of the gateways will be required to follow its rules to use the relay services and pay any fees required. 

- **Would BPP support token transfers?** Token transfers require application-level interoperability protocols and typically involve (a) locking tokens on the source chains, (b) relayer the proof, and (c) minting tokens on the destination chain. Hence, any "locker" contract on the source chain can write a packet to the destination chain asking the corresponding "minter" contract to produce the corresponding token. Relayers will simply pass the message across chains. 

- **Would BPP support arbitrary message passing across chains?** Yes, the payload can include any data that must be passed from one chain to another. 

- **Will BPP support multi-hop relaying?**  Yes. Suppose a packet arrives at the gateway contract. The relayer may either 
(a) have a direct connection to the destination chain where it can post the packet. The packet will then
immediately be delivered to the destination address, or
(b) know the network that can be its "next hop" and post the packet to another gateway deployed on that chain. Subsequent relayers may pick up the packets from the gateway and deliver them to the destination or the next hop network. The relayer should decrement the hop counter by one. 

- **Why is it sufficient to gurantee unreliable delivery for this protocol?** BPP does not try to establish reliable data streams. This is the responsibility of the interoperability protocols sitting on top of BPP. A relayer may forge a packet, or change its fields, or drop altogether. An interoperability protocol or application that aims to establish a reliable data stream across separate chains needs to decide how to handle undelivered packets or forged requests. 
For instance, the protocol may retransmit the packet to the same or different gateway, waiting for the same relayer to deliver the packet to the destination. 


- **How do you prevent packet forging?** An optional signature may be added under the sender's address. This is not aplicable when a caller is a smart contract. 

## Test Cases


