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

Any blockchain can deploy routers or gateways (implemented as smart contracts or at the consensus layer) that receive packets in the format proposed below. Any relayer can pick up packets from the gateway queues and deliver them to destination chains. Interoperability, bridging, and application-level protocols sit on top of the blockchain packet protocol and can write to their local chain gateways to communicate with remote chains. 

## Motivation

Our objective is to: 
a) make it easy to develop new interoperability or bridging protocols across distinct blockchains while,
b) unifying and leveraging a common relay packet protocol format and relayer infrastructure to send packets from one chain to another. 

Without this standard, every bridging protocol will need to roll out its own swarm of relayers just to post messages from one chain to another.  

## Specification

BPP is not a reliable protocol. Packets may be dropped or delivered multiple times. 
Bridging protocols on top of it are responsible for ensuring reliable or sequential reconstruction of packets. 

|     |     |     |
| uint: verion | uint: remaining hops | uint or string: flow type | 
| uint: len(src chain_id) | byte[] src chain_id | uint: len(src address) | byte[] src address |
| uint: len(dst chain_id) | byte[] dst chain_id | uint: len(dst address) | byte[] dst address |
| uint: len(payload) | byte[] payload | |
| Optional: [uint: signature type | uint: sig length | signature of packer under src address] | 

The specification mirrors the IP packet used to deliver information on the Internet. 

* uint = 4 bytes. All other fields are variable length. 
* Version: the protocol version number. 
* Remaining hops: we do not want packets to circulate forever from one hop to the next (potentially ending up in a loop). Each time a packet is relayed, the number of remaining hops is decremented by one. 
* Flow type identifier: specifies asset class, type of interoperability protocol, or information that is propagated. 
* src/dst chain_id: source/destination chain identifier. Potentially following: https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-2.md
* src/dst address: the sender/receiver of the packet. Typically the sender/receiver will be a higher level interoperability protocol (potentially implemented at the consensus or smart contracts) that is responsible for ensuring a reliable data stream between applications. [chain_id and address may be merged following: https://github.com/ChainAgnostic/CAIPs/blob/master/CAIPs/caip-10.md]
* payload: the message that needs to be transmitted from one chain to another. 
* (TODO) signature: some packets might be signed under the sender's address. This ensures that no forged packets may be created. This field is optional.

 packet must be signed 

## Rationale

* In a multi-chain world, relaying packets from one network to another is a basic requirement. 
* Regardless of the underlying interoperability protocol relaying packets should be a standard and unified operation. 

* How will the transaction fees be paid? 

Any relay provider can register a gateway "smart contract" on-chain (or module at consensus layer) to accept incoming packets of the form above. It may set it own fee mechanism, DDOS protection techniques, or relayer policies. The callers of the gateways will be required to follow its rules to use the relay services. 

* Would BPP support token transfers?

Token transfers require application-level interoperability protocols and typically involve (a) locking tokens on the source chains, (b) relayer the proof, and (c) minting tokens on the destination chain. Hence, any "locker" contract on the source chain can write a packet to the destination chain asking the corresponding "minter" contract to produce the corresponding token. Relayers will simply pass the message across chains. 

* Would BPP support arbitrary message passing across chains? 

Yes, the payload can include any data that must be passed from one chain to another. 

* Will BPP support multi-hop relaying?

Yes. Suppose a packet arrives at the gateway contract. The relayer may either 
(a) have a direct connection to the destination chain where it can post the packet. The packet will then
immediately be delivered to the destination address, or
(b) know the network that can be its "next hop" and post the packet to another gateway deployed on that chain. Subsequent relayers may pick up the packets from the gateway and deliver them to the destination or the next hop network. The relayer should decrement the hop counter by one. 

* Why is it sufficient to gurantee unreliable delivery for this protocol? 

BPP does not try to establish reliable data streams. This is the responsibility of the interoperability protocols sitting on top of it. A relayer may forge a packet, or change its fields, or drop altogether. 
Regardless, any relayer (or any adversary on the blockchains) may call contracts with arbitrary functions or inputs, and it's the responsibility of the interoperability protocols and blockchains themselves to mitigate those attacks (e.g., DOS is prevented via transaction fees). An interoperability protocol or application that is trying to establish a reliable data stream with a destination contract may retransmit the packet, hoping that another relayer (or a set of relayer) will deliver it to the destination. Regardless, this interoperability protocol must assume an "at least once" delivery from external infrastructure that it cannot control without a direct connection to the destination chain. 

* How do you prevent packet forging? An optional signature may be added under the sender's address. 

## Test Cases
Please add test cases here if applicable.


