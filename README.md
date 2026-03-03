# Vyomanaut_Research
The V1 of the project is available here 
https://github.com/masamasaowl/Vyomanaut
</br>It failed due to 
1. Lack of research-based in architecture
2. Structural compromises made during build
3. Inefficient transfer speed, storage & peer discovery
   
# System Design 
## Functional requirements
```java
What the system must do.
1. A service for users/companies/data owners to store data 
2. The data is stored inside the idle memory of phones/laptops/devices.
3. Data owner pays; storage provider earns (escrow)
4. Storage provider can't access the stored data 
5. Data must be available to the data owner at all times
6. The storage provider can alter the storage contribution
7. Storage provider can exit the network; data owner can delete the data
8. The service never stores/reads the given data
```
## Non-Functional Requirements (Decided Direction, Quantities Open)

| Property | Decision |
| --- | --- |
| Availability | No planned downtime. Emergency downtime as last resort only |
| Privacy | Zero-knowledge. Service has no access to plaintext or keys |
| Encryption | Client-side, mandatory. Type and stage TBD via research |
| Redundancy | Best-effort model. Exact replica count TBD via research |
| Proof of Storage | Cryptographic or audit-based. Mechanism TBD |
| Provider Reliability | Scored. Score drives assignments, earnings, and exit permissions |
| Payments | Fiat escrow. Basis of provider payment TBD |
| Background Operation | Required on all device types. OS strategy TBD |
| Architecture | Microservices coordination layer + pure P2P transfers |
| Decentralization | Hybrid. Microservices for orchestration, P2P for all data movement |

## Research dependency order 
```java
[1] Coordination Architecture          ← Everything depends on this
        |
        |──[2] Proof of Storage        ← Depends on how coordination works
        |
        |──[3] Erasure Coding          ← Determines redundancy strategy
                |
                |──[4] Replication Protocol    ← Depends on redundancy math
                        |
                        |──[5] Peer Selection Algorithm
                        |
                        |──[6] Availability / Polling Protocol
                                |
                                |──[7] Provider Exit State Machine
                                        |
                                        |──[8] Reliability Scoring Model

[9] Client-Side Encryption             ← Parallel, blocks key management
        |
        |──[10] Key Management Strategy

[11] Background OS Execution           ← Parallel, blocks mobile provider design

[12] P2P Transfer Protocol             ← Parallel, blocks all transfer designs

[13] Escrow & Payment Basis            ← Last, depends on the reliability model
```

## Topics under study
| Concepts | Study | Summary |
| --- | --- | --- |
| Coordination layer design | **BitTorrent spec**, **IPFS architecture** | Tracker vs trackerless, DHT design |
| Pure P2P architecture | **Kademlia DHT paper** (Maymounkov & Mazières 2002) | The gold standard for decentralized peer discovery |
| Proof of Storage | **Filecoin whitepaper**, **Storj whitepaper** | Both solve this differently — great for trade-off study |
| Erasure Coding | **Reed-Solomon coding**, **Storj's erasure implementation** | Storj is your closest architectural cousin overall |
| Replication & availability | **Apache Cassandra replication model**, **Dynamo paper** (Amazon 2007) | How large systems maintain replica health |
| Availability polling | **Consul health checks**, **etcd** | Heartbeat and failure detection in distributed systems |
| Reliability/reputation scoring | **BitTorrent tit-for-tat**, **EigenTrust paper** | Scoring peers in untrusted P2P networks |
| Client-side encryption | **Signal Protocol**, **libsodium docs**, **age encryption** | Modern, audited, well-documented |
| Key management | **Keybase architecture**, **Tresorit whitepaper** | Zero-knowledge key management done right |
| Background execution | **Android WorkManager docs**, **iOS BGTaskScheduler docs**, **WhatsApp engineering blog** | OS-level background strategies |
| P2P file transfer protocol | **WebRTC**, **libp2p**, **QUIC protocol** | Modern P2P transport options |
| Escrow & payments | **Stripe Connect docs**, **Upwork escrow model**, **Airbnb payment architecture blog** | Marketplace escrow patterns |
| Microservices design | **Building Microservices — Sam Newman (book)**, **Netflix tech blog** | Industry-grade service decomposition |

## System design concepts to study
```java
1. Distributed systems fundamentals
      - CAP theorem 

2. Distributed Hash tables 
      - Trackerless P2P transfer 

3. Erasure Coding
      - Reed Solomon algorithm
 
4. Merkle trees
      - proof of storage used by IPFS
      
5. Consisten hashing
      - replication handling
      
6. Heartbeat and Gossip Protocols 
      - Availibilty protocol
      
7. Zero-Knowledge Architecture
      - Service routes data it cannot read
      
8. Saga Pattern (Distributed Transactions) 
      - When upload fails, how to rollback
      
9. Backpressure and Flow Control
      - Slow providers slow the system intead of collapse                                                
```

## Decisions made 
1. All file encryption happens client-side, before upload. The service never possesses plaintext data.
2. The coordination layer is hybrid: microservices for critical orchestration, Kademlia DHT for peer discovery and metadata routing (The exact  functionalities which microservices handle are yet to be finalised).
3. File data moves purely peer-to-peer. The coordination layer moves metadata only.
4. Three provider exit types with distinct handling: accidental (penalty), announced (no penalty, smooth migration), promised (conditional, high-reliability providers only).
5. Payments are fiat-based via escrow. No crypto tokens.
6. Content addressing via cryptographic hash is used for chunk integrity verification (Merkle DAG principle from IPFS).
7. Provider identity must persist across sessions — cross-session reliability scoring requires a stable identity (public key derived NodeId from S/Kademlia / IPFS pattern).
8. Chunk pinning (IPFS model) is the storage contract primitive: pin = commit, unpin = terminate.
   
## Research briefing  
```java
Step 1: Start with a survey paper and not original paper 
        When entering a new topic, it summarises the landscapes effectively

Step 2: Read paper in order 
        (start) Abstract -> Conclusion 
        (if still relevant) -> Introduction -> Figures & Diagrams 
        (if still relevant) -> read entire paper        
 
Step 3: Use citations in both directions 
        Visualise the connected papers 
        
Step 4: For real systems prefer reading 
        Engineering white papers and conference talks
        eg:
            1. USENIX -> system papers
            2. OSDI, SOSP, EuroSys -> conferences
            3. Company engineering blogs
            4. IETF RFCs -> formal specification of protocols
            
Step 5: Don't memorise
        Ask these three questions 
        1. What problem does it solve?
        2. What trade-offs did it accept?
        3. What would break in my context at scale?
        
Step 6: Read implementation after reading paper
        eg: After reading Kademlia  
            Study the implementation of it by torrents or libp2p  
            
Step 7: Before finalising always search for disagreements
        Search for reports against the approach
        1. Why the technology failed
        2. Commonly reported problems
        3. What are all the alternatives
        
        It helps defend a decision                                     
```

## Research summary template 
```java
PAPER: [Title, Authors, Year, Venue, Citation Count]
TOPICS: [From research agenda #list]

PROBLEM SOLVED: [2–3 sentences. Precise. Not just 'distributed storage' — what specific failure mode does it address?]

TRADE-OFFS (intentional choices with understood costs):
  • [Mechanism chosen] over [alternative] → [consequence for our system]

BREAKS IN OUR CASE (where it fails to map):
  • [Assumption in paper] ≠ [our reality] → [adaptation required]

DECISIONS INFLUENCED (concrete, named decisions):
  • Decision #N [topic area]: [choice made or forced] because [reason from this paper]

DISAGREEMENTS (papers that challenge this):
  • [Paper name, year]: challenges [specific claim] because [reason]. Implication for us: [X]

OPEN QUESTIONS (must not repeat questions answered by a previous paper):
  • [Question] — blocked on: [which future paper or prototype answers this]
```
### Research Paper 1
```java
Paper: Incentives Build Robustness in BitTorrent, Bram Cohen, May 22, 2003

Research topics addressed: #1

Problem solved: 
1. Infinite scaling as incoming downloaders arrive
2. Track pieces
3. Handle peer churn
4. Maintain speed fairness of upload vs download
5. 70% of Gnutella (first P2P DSN) users uploaded no files at all
	 
Trade-offs:
1. Random peer selection over optimised (result -> benefited)
2. 20-second average of upload speed instead of historical upload contribution

Breaks in our case:
1. The data owner uploads once and laxes, torrent is being used when two storage
   providers try to maintain the redundancy
2. No choking algorithm required as all peers are individually incentivised, 
   but adversarial provider behaviour (data deletion, false audit responses) must
   still be explicitly designed against
3. A tracker acts as a vulnerability and a single point of failure  
4. The reliability scoring based on uptime must consider days or larger widths 
   of time spans; the 20-second rolling window of torrent wouldn't work

Decisions influenced:
1. A random peer set is more robust than a structured one to handle peer churn
2. Piece size 256KB, sub-piece size 16KB, with 5 requests lined up
3. The anti-snubbing mechanism can be seen as a heartbeat detection method
4. Reliability scoring need not begin from zero; rather, by the Optimistic unchoking 
   assigning method, a random piece can decide the reliability of a peer

Disagreements:
1. "BitTorrent is an Auction" (Levin 2008): Talks about the tit for tat model
    Not needed for us, as choking algorithms are not needed

Open Questions after reading:
1. How to avoid the tracker, as it is efficient but vulnerable?
   Ans:  #1 Kademlia use it
   
2. Should we also keep requests lined up, so TCP is always loaded?
   Ans: #12 Yes, it's good for P2P transfers
   
3. Random peer set works great, but 
   a) What criteria decide the reliability of a peer?
   b) How to decide the initial reliability of a peer?
      Ans: #5 Try random optimistic assignment
      
   c) How to re-rank based on reliability
5. Is it possible to keep geographic proximity as a parameter while assignments
   promote quick transfers?
6. Should reliability score influence the volume of storage assigned to a 
   provider, and if so, does this create runaway concentration of data in 
   the hands of a few top-rated providers (Matthew effect)?
7. Should the Providers with higher transfer speeds, uptime and storage be 
   rewarded accordingly by the escrow model?
```
### Research paper 2
```java
Paper: Kademlia: A P2P information system based on XOR metric, 
       Peter Maymounkov & David Mazieres, New York University

Research topics addressed: #1

Problem solved: 
1. In an infinitely large library, how can each visitor find any book individually
2. Each book and visitor is given an ID
3. The Closeness metric between two IDs is calculated using XOR
4. Each visitor has a small handbook of other visitors at specific distances
5. This step is not imposed by us, but each node learns about other nodes 
   as a side effect of a search; this is self-configuration -> ends peer 
   churn problem
6. When a book is needed
       - We call the visitor nearest to the book Id
       - They point to someone even closer to the book 
       - Finally, we converge to the closest visitor in logarithmic steps
7. A 10-million-node network takes roughly 20 hops to get to the book
8. In high peer flooding and churning, k-buckets keep long-lived nodes alive
	 
Trade-offs:
1. Used the XOR metric throughout the search as opposed to Pastry and Tapestry
2. Older node if live is always kept instead of a new node
3. Bandwidth usage changes for parallel node searches handled by the alpha parameter
   (default value 3) which improves fault tolerance at the cost of O(alpha × log n)
   messages vs O(log n) for sequential lookups
4. Key value pairs need to be refreshed every 24 hours so they require active maintenance.
5. Node IDs are random so a personalised distribution of IDs is not possible

Breaks in our case:
1. Kademlia only stores IDs as keys and not large files 
2. So, file storage would be handled by a completely different protocol
3. Kademlia would strictly run on the device only to look up which providers
   have the file for the given fileID (This part can also be centralised by
   maintaining a lookup table with the microservices)
4. Data owner uploads and forgets, it is the duty of the Availability service to 
   update the DHT every 24 hours as a republish   

Decisions influenced:
1. Provider discovery, metadata routing, and peer lookup can all happen using
   this algorithm
2. People with high uptime continue to stay connected (Gnutella graph), so they 
   must be rewarded   
3. Caching can be handled autonomously by the nodes
4. What to do when a new storage provider joins -> discussed in Kalemdia protocol
   If the head of the k-bucket responds -> we reject the  new value and shift the table up
   If he doesn't respond -> We remove the oldest node
5. The availability checker must update the key value pair every 24 hours
   as Kademlia doesn't keep stale values  
6. To find a chunk, we use FIND_VALUE, and for replication, we use FIND_NODE to 
   look for the k best candidates for storage  
                                                    
Disagreements:
1. S/Kademlia -> BitTorrent contains malicious nodes at any given time 

Open Questions after reading:
1. Is there a specific edge case where it doesn't converge to the right visitor?
   how to tackle it?
   Ans: #1 S/kademlia fixes it via sibling nodes
   
2. How to decide the replication parameter k?
   Ans: #1 Decided by S/kademlia as 8.16
3. How to set our accelerated lookups partner b, which is our branching factor?
4. Should the data owner also be part of the network, being a node with no
   stored value? We saw how there is no way to distribute node IDs based on
   roles or geographic values. Should we keep the tracker only for the data
   owner, and once the file enters the provider swarm, it works on Kademlia?
   
5. How to do the republication of key value pairs in k-buckets? Include examples 
   where a strategy has been devised for such a task.
   Ans: If node A stores a key-value pair, and A knows that node B 
        (also close to the key) stored it 50 minutes ago, A skips 
        republication. Only one node republishes per hour. 
        Availability Service can use the same logic
```

### Research paper 3
```java
Paper: S/Kademlia: A Practicable Approach Towards Secure Key-Based Routing,
       Ingmar Baumgart and Sebastian Mies, Institute of Telematics Universitat Karlsruhe
       Karlsruhe, Germany

Research topics addressed: #1

Problem solved: 
1. The Kademlia protocol by itself is not secure
2. Shows ways to avoid the Eclipse and Sybil attack on the network by adversarial nodes
3. The attacker must not 
    - be free to choose an ID
    - able to easily generate multiple node IDs
	 
Trade-offs:
1. Node ID generation ceases to be instantaneous because of public keys hashes
2. If supervised signatures are chosen, then we compromise pure decentralisation
3. A sibling list adds network overhead
4. Network traffic is multiplied by the value of disjoint paths 

Breaks in our case:
1. Creating public key hashed node IDs might not be necessary, as our users
   registers before entering the network, this is the supervised signature 
   approach discussed in the paper, which automatically limits the number of
   nodes entering the network (true if we become the certificate authority)
   (pure decentralisation is not achievable with this model)

Decisions influenced:
1. Node Ids must be derived from a public key, making it computationally 
   expensive for an attacker to flood the network (alternative: If the 
   registration is proven effective, then it won't be needed)
2. Maintain disjoint paths d=4.8 and replication parameter k=8.16 (i.e. 2*d),
   system maintains high efficiency (99%) even at 30% share of adversarial nodes
3. Every peer to maintain a sibling list containing information of the 
   neighbouring nodes with s=20, c=2.5 giving failure probability of ~5×10⁻⁷
   (It guarantees the survival of each replica), it also makes k-bucket
   splitting algorithm unnecessary
4. Nodes talk to each other via gRPC or QUIC (Self decision)  

Disagreements:
No disagreements noted yet

Open Questions after reading:
1. Should we hash node IDs using a  public key? If not, then how to generate a node ID?
   Is making the user register before entering the network a good way to avoid flooding of adversarial nodes?
   Ans: #1 make the decision for pure P2P or hybrid
   
2. Is it feasible to implement a pure P2P sibling broadcast? What are the
   challenges associated with this implementation? I wish to make the network
   less dependent on a central entity 
   Ans: #1 Pure P2P doesn't work as researched with less than 100 peers
   
3. Kademlia began operating in 2003 (Skademlia in 2007), since then, has there not been a better way for P2P networks to interact without a central entity?   
   Ans: #1 libp2p is a model currently being used in large P2P networks   
```

### Research paper 4
```java
Paper: IPFS - Content Addressed, Versioned, P2P File System, Juan Benet, 2015

Research topics addressed: #1, #2

Problem solved: 
1. It remains to be the closest system to our architecture, knowing its 
   strengths and failures can save a lot of rediscovering
2. It moves more towards Web3 (data not saved on a single server) 
   whereas we try to incentivise the data each node holds 
2. Files are addressable by their content and not location, so they live as 
   providers change   
3. We address a chunk by the hash of its content and not based on the provider 
   who stores it -> ensures files live through peer churns
	 
	 
Trade-offs:
1. As content is immutable in the Merkle DAG, making it difficult if 
   future updates demand a PATCH of a file by the data owner, while it 
   is with the publisher
2. Performance at first is slower than an HTTP request, as a DHT lookup is 
   needed 

Breaks in our case:
1. It doesn't aim to create a network alternative to data centers rather it's
   a general-purpose content-addressable distributed file system spread across 
   the internet
   a) No persistent storage guarantee
   b) No incentive for storage
   c) No access control as data owner is not given privileges (anyone can retrive)
2. BitSwap's tit-for-tat block exchange is replaced by escrow-motivated direct
   transfers, The transfer mechanism (direct P2P, chunked, pipelined) is
   retained; the incentive mechanism is replaced. 
3. A retrieve call can be made by any node, so an additional control layer is 
   needed 

Decisions influenced:
1. Make use of Coral with 
    small store value -> info of other nodes
    large store value -> metadata of actual files
2. It examples an server made in GoLang
3. The provider earnings tracker can be modelled from the BitSwap ledger 
4. Verification of a chunk can be done by the content adressing mechanism
   of Merkle DAGs, if the hash doesn't match -> the chunk is corrupted
   A proof of storage mechanism needs to exist separately, which simply checks
   a cryptographic challenge; should require less computation.
5. To store a chunk, we can simply pin it to the node; to remove/delete, simply
   unpin the object (chunk)
6. The reliability score could be based on storage uptime and response latency
7. The provider earnings ledger use these mathematical relations
   Debt ration (r) = bytes_sent/bytes_recv + 1 
   
   Then a sigmoid function
   The probability of sending to a debtor is:
   P(send|r) = 1 - (1/ 1 + exp(6 - 3r))

Disagreements:
1. The disagreements criticise the Web3 aspect and not the properties useful 
   for us

Open Questions after reading:
1. What does the P2P replication follow?
   a) BitSwap (without tit-for-tat)
   b) libp2p
   c) QUIC
   d) a better alternative
   
2. The links array inside an object in the DAG, can it contain the path to its
   redundant siblings who store the same chunk? Do a feasibility check and if 
   there exists a better usecase of this feature
   Ans: The links are made to store file-chunk relationships. Storing the
        chunk-provider data can confuse use cases. Kademlia is meant to handle 
        provider peer lookups already
           
3. How would we make use of the IPNS if the data owner is not part of the 
   network? The routing of files is already handled by Kademlia, so how to
   use the pointers? Can the IPNS be used as a backup method for a file lookup
   Ans: If the data owner agrees, then a microservice can do proxy signatures,  
        resolving the DHT republication problem (needs design to implement)

```
### Research paper 5
```java
Paper: Storj: A Decentralised Cloud Storage Network Framework
       Storj Labs, Inc. | v3.0 Oct 2018, v3.1 Mar 2024

Research topics addressed: #1, #2, #3, #4, #5 

Problem solved: 
#(Listed at start of each topic in the detailed notes of the paper)
	 
Trade-offs:
1. Dropped Kademlia for the central coordination server (satellite) discussed
   in Appendix A
2. Reed Solomon over simple replication -> optimised redundancy
3. Explicit node selection over Dynamo -> geographic/performance/reputation
   filtering possible
4. Stripe-level erasure coding over segment-level -> disputable solve by paper 
   Separation and Optimisation of Encryption and Erasure Coding in Decentralised Storage
5. Random stripe audits (Berlekamp-Welch) over pre-generated Merkle proofs
   provide avoids Audit false positive risk
6. No payments for the first 6 months to incentivise long-lived nodes
  

Breaks in our case:
1. It assumes MTTF to be 6–12 months due to NAS operators in the network, 
   With the inclusion of mobile operators, MTTF comes to 1 month, changing 
   erasure parameters and increasing repair bandwidth 
2. It targets desktop users with symmetric bandwidth, whereas mobile
   operators have 1-5 Mbps upload, so the piece size needs to be reduced 
3. It depends on the satellite for repairs; we need to be robust with repairs
   to handle mobile users volatility
4. The waiting period of 6 months is very taxing for an average user 
5. No background execution takes place as a dedicated daemon or server 
   runs on the desktop 

Decisions influenced:
1. Repair bandwidth consumption is the biggest challenge of the entire 
   project 
2. Use erasure coding instead of simple replication, look for more
   optimised erasure algorithms
3. Use four erasure parameters (k, m, o, n) for long tail transfers
   The starting values can be k=29, n=80, m=35, o=52 but need to be 
   optimised for the 1-month MTTF, as much higher 'n' might be needed to 
   distribute load
4. Use Berlekamp-Welch error detection instead of Merkle challenges 
   for audits 
5. For the betting period, use Bayesian audit scoring with Jeffery prior 
   β(0.5, 0.5) -> 80 audits give 99% trust in provider
6. A pointer must be created and maintained at the end of the data owner
   storing node IDs, erasure parameters, piece IDs, encryption info,
   repair threshold, piece hashes, and a signature
7. We hold the earnings for 6 months. This is bound to change with research
8. Adopt Neuman's proxy protocol, which reads the used bandwidths agnostically
   of payments, incremental accounting prevents frauds
9. Utilise bloom filters [cit:50] for garbage collection to check if the 
   marked out segment has been deleted or not 
   

Disagreements:
1. "Design & Evaluation of IPFS" (Trautwein 2022): Shows how DHT is the 
    optimised method for node discovery 
2. "Handling Churn in a DHT" (Rhea et al. 2004): Shows MTTF of a few hours 
    or days, challenging the 6-12 months MTTF assumption
3. "High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two"
  (Blake & Rodrigues, 2003) -> Storj chooses availability and scalability, 
  accepting high repair bandwidths utilised in peer networks, we can disagree
  
1. Satellite coordination dropped the reliability of data owners in the system
2. Repair consumed significant bandwidth -> Walrus and EC survey tackled
   this problem with regenerating codes and 2D coding schemes
3. Provider payments are volatile with the Storj token; providers earn for
   storing and not for continuous availability
4. Provider discovery suffered latency due to the satellite
5. Encrypt, then apply erasure, opposed by Separation and Optimisation of
   Encryption and Erasure Coding in Decentralized Storage**,** *Marcell Szabó et al*

Open Questions after reading:
1. What challenges or drawbacks does a central entity like satellites pose? Can 
   all of its tasks be performed in a decentralised manner
2. At what MTTF does the model become viable, with repair bandwidth being
   the biggest challenge in the architecture?
3. 

```
### Research papers to continue reading
| **#** | **Title** | **Type** | **Phase** | **Priority** | **Topics Covered** |
| --- | --- | --- | --- | --- | --- |
| **1** | Storj Whitepaper v3 | Whitepaper | **Ph 1** | **MANDATORY** | #1, #2, #3, #4, #5 |
| 1 | High availability, scalable storage, dynamic peer networks: Pick two, Charles Blake and Rodrigo Rodrigues | paper  | ph1 | mandatory | #1 |
| 1 |  **A practical guideline to be lazy. IEEE Global CommunicationsConference (GlobeCom), 12 2010** | Paper | Ph1 | Mandatory | #1 |
| **2** | Filecoin Whitepaper | Paper | **Ph 1** | **MANDATORY** | #1, #2, #5, #13 |
| **3** | Design & Evaluation of IPFS (Trautwein 2022) | Paper | **Ph 1** | **MANDATORY** | #1, #5, #6 |
| **4** | Coral DSHT (Freedman 2004) | Paper | **Ph 1** | **MANDATORY** | #1, #5 |
| **5** | Measurement Study of P2P File Sharing (Saroiu) | Paper | **Ph 1** | **MANDATORY** | #3, #5, #6 |
| 6 | SoK: Decentralized Storage Network (Chuanlei Li et al.) | paper  | Ph 1 | Mandatory | #1, #2, #3, #5, and #13 |
| 7 | Coordination avoidance in database systems, Peter Bailis, Alan Fekete | paper  | ph2 | mandatory
(storj) | #14 |
| **6** | A Tutorial on Reed-Solomon Coding (Plank 1997) | Paper | **Ph 2** | **MANDATORY** | #3 |
| 7 | Erasure Codes for Cold Data in Distributed Storage Systems (Chao Yin et al.) | Paper | ph 2 | Mandatory  | #3 |
| **7** | Erasure Codes for Storage Systems (Plank 2013) | Paper | **Ph 2** | **MANDATORY** | #3, #4 |
| 7  | **Separation and Optimization of Encryption and Erasure Coding in Decentralized Storage,** *Marcell Szabó et al., Budapest University of Technology* | paper  | ph2 | mandatory | #3, #9, #15 |
| 7 | Survey of the Past, Present, and Future of Erasure Coding for Storage Systems (Shen, Cai, Lee et al. — ACM ToS Dec 2024) | paper | ph2 | mandatory | #3 |
| 7 | Towards Benchmarking Erasure Coding Schemes in Object Storage Systems (Upoma, Afrin et al. — FGCS 2025) | paper | ph2 | recommended | #3 |
| 7 | ELECT: Enabling Erasure Coding Tiering for LSM-tree-based Storage (USENIX FAST 2024) | paper | ph2 | recommended | #16 — Provider-Side Storage Engine |
| 8 | Walrus: An Efficient Decentralized Storage Network | paper | ph2 | recommended | #1, #2, #3 |
| **8** | Dynamo: Amazon's Highly Available KV Store | Paper | **Ph 2** | **MANDATORY** | #4, #6 |
| 8 | Sanjay Ghemawat, Howard Gobioff, and Shun-Tak Leung. The Google File System | paper  | ph 2 | Read only |  |
| 9 | Lustre. Introduction to Lustre Architecture | paper | ph 2 | Read only |  |
| **9** | OceanStore: Architecture for Global Storage | Paper | **Ph 2** | **RECOMMENDED** | #2, #3, #4 |
| **10** | EigenTrust: Managing Trust in P2P Networks | Paper | **Ph 3** | **MANDATORY** | #8 |
| 10  | B. C. Neuman. Proxy-based authorization and accounting for distributed systems. | paper  | ph3 | mandatory  | #13 |
| **11** | S/Kademlia Sibling List & Attack Model | Paper | **Ph 3** | **— (done) —** | #1 ✓ |
| 12 | "Handling Churn in a DHT" (Rhea et al. 2004) | paper  | ph3  | mandatory  |  |
| **12** | The Double Ratchet Algorithm (Marlinspike) | Paper | **Ph 3** | **RECOMMENDED** | #9 |
| **13** | Keybase Architecture (blog + whitepaper) | Blog | **Ph 3** | **RECOMMENDED** | #9, #10 |
| **14** | Tresorit Whitepaper | Whitepaper | **Ph 3** | **RECOMMENDED** | #9, #10 |
| **15** | libp2p Spec & Architecture Docs | Docs | **Ph 3** | **MANDATORY** | #1, #12 |
| **16** | QUIC: A UDP-Based Multiplexed Transport (IETF) | Paper | **Ph 4** | **MANDATORY** | #12 |
| **17** | WebRTC for the Curious (open book) | Docs | **Ph 4** | **MANDATORY** | #12 |
| **18** | An Incentive-Compatible Mechanism for DSN | Paper | **Ph 4** | **MANDATORY** | #13 |
| **19** | BTT Whitepaper 2019 | Whitepaper | **Ph 4** | Read only | #12, #13 |
| 20 | Satoshi Nakamoto. Bitcoin: A peer-to-peer electronic cash system | paper | ph4 | Read only | #8 |
| **20** | Android WorkManager Docs | Docs | **Ph 4** | **RECOMMENDED** | #11 |
| **21** | iOS BGTaskScheduler Docs | Docs | **Ph 4** | **RECOMMENDED** | #11 |
| **22** | Vakilinia 2022 — Incentive-Compatible DSN | Paper | **Ph 5** | **MANDATORY** | #13 |
| **23** | Stripe Connect Docs | Docs | **Ph 5** | **MANDATORY** | #13 |
| **24** | Chord (Stoica 2003) | Paper | **Ph 5** | **OPTIONAL** | #1 |
| **25** | Building Microservices — Sam Newman (book) | Book | **Ph 6** | **RECOMMENDED** | #4, #6, #7 |
| **26** | Designing Data-Intensive Applications — Kleppmann | Book | **Ph 6** | **MANDATORY** | #4, #6, #8 |
| **27** | Netflix Tech Blog — Chaos Engineering | Blog | **Ph 6** | **OPTIONAL** | #6 |
