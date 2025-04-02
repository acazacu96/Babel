
1. [Class](#Class)
2. [Intro](#Intro)
	1. [Layers of internet](#Layers%20of%20internet)
	2. [Headers](#Headers)
	3. [Timing Diagrams](#Timing%20Diagrams)
3. [Routing](#Routing)
	4. [Introduction](#Introduction)
	5. [Interior Gateway Protocols](#Interior%20Gateway%20Protocols)
		1. [Distance-Vector Protocol](#Distance-Vector%20Protocol)
		2. [Link-State Protocol](#Link-State%20Protocol)
	6. [Exterior Gateway Protocols](#Exterior%20Gateway%20Protocols)


## Class 
https://textbook.cs168.io/

## Intro
### Layers of internet
![[Pasted image 20250311103601.png]]
### Headers

![[Pasted image 20250311105954.png]]
![[Pasted image 20250311110332.png]]
### Timing Diagrams

```cpp
Packet_delay = transmission delay + propagation delay + queuing delay.

// time it takes to load last bit in the link
transmission_delay = number_of_bits_to_send / bandwidth  
propagation_delay // time it takes for bits to travel through link
queuing_delay // time the router takes to process the incoming bits
```


![[Pasted image 20250311112033.png]]
![[Pasted image 20250311113233.png]]

Overloaded Links
![[Pasted image 20250311113340.png]]

--------
## Addressing


![[Pasted image 20250318140745.png]]


-  IPv4 32 bits
- domains (networks) are assigned a prefix: 
	- A.B.C.D/P (P fixed ports)
		- e.g.: 10110000.0.0.0/4 (1011 is the prefix of an internet provider)
- Netmask is the alternative to bits prefix. 
	- e.g.: 11110000.0.0.0 mask indicates the first 4 bits represent the network
 - IPv6 has 128 bits

Routers support both versions and have separate forwarding tables.



## Routing

### Introduction

The network will compute a Delivery Tree for each destination (oriented spanning tree)
- no deadends
- no loops
![[Pasted image 20250314170650.png]]

### Interior Gateway Protocols

#### Distance-Vector Protocol

- Forward table based on destinations: (Destination, Hop/Direct, Cost)
	- Each entry can expire
- Relax the paths and the network will converge
- Periodic/Triggered updates to connections about FT (30s)
	- Poison-Reverse: Poison to next Hop
	- Split-Horizon: Don't send to next Hop
- Count to infinity to avoid loops
![[Pasted image 20250314165901.png]]

#### Link-State Protocol

IS-IS (Intermediate System to Intermediate System) and OSPF (Open Shortest Path First) are two major examples of link-state protocols. Both are widely deployed today.

The exact opposite of Distance-Vector (where the FW tables are computed in a distributed way, each router helping and getting help from its connections).

Here, each router learns the entire network by sending messages about neighbors, and populates the FW table individually using shortest path algorithms. 

### Exterior Gateway Protocols