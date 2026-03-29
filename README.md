# Vyomanaut_Research
The V1 of the project is available here 
https://github.com/masamasaowl/Vyomanaut
</br>It failed due to 
1. Lack of research in architecture
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

## Topics of study
```java
#1 Coordination Architecture: 
   Central vs DHT balance; what stays decentralised and why; Satellite
	 design

#2 Proof of Storage: 
   Audit mechanism: full PoRep vs random Merkle challenge vs
	 lightweight heartbeat

#3 Erasure Coding:
   EC scheme selection (RS/MSR/LRC); (k,m) parameters; cold vs
	 warm tradeoffs

#4 Replication / RepairProtocol: 
   Who triggers repair; peer-to-peer chunk transfer during repair; repair
	 bandwidth budget

#5 Peer Selection Algorithm
   Selection criteria: latency, uptime, geography, free space; initial
	 assignment

#6 Availability & Polling
   Polling interval vs audit gap attack surface; heartbeat failure
   detection

#7 Provider Exit State Mechanism:
	Accidental / announced / promised exit; grace period; migration
  orchestration

#8 Reliability Scoring Model
	 EigenTrust vs rolling audit rate vs hybrid; bootstrapping new
	 providers; decay function

#9 Client-Side Encryption Encryption
   scheme; key derivation per chunk vs per file; forward secrecy

#10 Key Management Strategy
   Owner key loss scenario; key rotation; delegation to service for
   republication

#11 Background OS Execution
   iOS/Android background execution limits; provider tier model 
   (desktop vs mobile)

#12 P2P Transfer Protocol 
   QUIC vs WebRTC; NAT traversal; connection migration for mobile
   providers

#13 Escrow & Payment Basis
   Per GB stored vs per audit passed vs per GB transferred; penalty
   structure

#14 Consistency Model
   Which operations need strong consistency vs eventual; invariant
   confluence framework, all nodes need to see the same data at the same time
   
#15 Encryption-Erasure Interaction
	 Encrypt-then-code vs code-then-encrypt; key management cost of
	 each choice

#16 Provider-Side Storage Engine
	 How providers store chunks locally; LSM-tree vs object store; EC
	 tiering

#17 Repair Bandwidth Optimisation
	 MSR/LRC regenerating codes; repair parallelisation; helper node
	 selection

#18 Economic Mechanism Design
	 Game-theoretic incentive compatibility; credit systems; SLA
	 enforcement

#19 Adversarial Provider Behaviour
	 Provider deletes data between audits; false audit responses;
	 collusion modelling
```
   
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
### Research paper 6
```java
Paper: High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two
        Charles Blake & Rodrigo Rodrigues, MIT Laboratory for Computer Science
        Venue: HotOS IX (USENIX), May 2003 | 6 pages
        TOPICS: #3, #4, #5, #6
 
Problem solved:   
1. Basic idea is these three things cannot happen at once, extending to the
   CAP theorem
	   - High peer churn (Partition)
	   - Data always accessible (Availability)
	   - Store large amounts of data at scale (Consistency)
	   
	 In the case of Partition one must choose between A or C
2. Idle upstream bandwidth is the limiting resource that volunteers 
   contribute, not idle disk space
3. Average bandwidth consumed per node  = 2 ((space per node) / uptime)
4. Quantifies the relation between node lifetime v/s number of nodes by 
   assuming 'k' and bandwidth
4. Provides availability data in 6 nines, the gold standard of system
   uptimes, offering downtimes of 2.6 seconds per month, making our 
   model extremely reliable 
5. Presented math of consumed bandwidth after applying erasure, delayed 
   redundancy calls and admission control

Trade-offs:
1. Assuming 6 nines of premium availability 
2. Assumes low values of bandwidth and high replication factors, but 
   they help in designing a model for adverse situations

Breaks in our case:
1. A million cable modem users to provide a month of continuous service for
   the network to hold 1000 TB would not work for us.
   (note redundancy factor of k=20 is very high and 200Kbps per node
   upload bandwidth is also very low). Research needed to implement 
   in our model
   

Decisions influenced:
1. Calculating the bandwidth of the network is more integral than the 
   combined storage network offers 
2. The volatility of the users directly decreases the storage capacity
   of the network, as bandwidth gets wasted in replication 
3. We assume that network speeds would increase at reduced rates
   according to recent ISP trends. ISPs promise symmetrical bandwidths
   upto 100Mbps for 600rupees which significantly favours the architecture 
4. Decision #4: The promised exit, where a user can announce the set time 
                he would be unavailable would be implemented to save bandwidth
                triggering replication only for true departures 
                (Failing a promise leads to direct collection of fines 
                from the linked payments account)
5. Decision #6: The polling interval is described as 't', a 24-hour
                timeout reduces maintenance bandwidth by a factor of
                30x compared to instant timeouts 
   6. Erasure coding allows 't'=25 for 'k'=15, proving 8x bandwidth savings 
   compared to replication, finalising them as the redundancy protocol
7. The architecture must move more towards stable providers rather than 
   More inclusion of nodes, as every new flaky node burdens the existing ones 
   bandwidth provided by reliable nodes

Disagreements:
1. It clearly states that common home storage suppliers cannot run the 
   architecture, but Storj and Filecoin show it works in production with
   the right coding, membership delay and admission control 

Open Questions after reading:
1. At what provider MTTF does the repair bandwidth consume
   more than 30% of the total system bandwidth?
   Ans: Using the formula, assuming each provider serves 50GB 
   At MTTF = 1 month  → BW/node = 2 × 50GB / 30 days = ~3.3 GB/day = ~310 KB/s continuous
	 At MTTF = 3 months → BW/node = 2 × 50GB / 90 days = ~1.1 GB/day = ~103 KB/s continuous
	 At MTTF = 6 months → BW/node = 2 × 50GB / 180 days = ~556 MB/day = ~52 KB/s continuous
   
   If background OS (#11) provides 100Kb/s, then MTTF must be more than 
   3 months for an average user 

2. What is the minimum provider MTTF your escrow model must enforce to 
   Keep the system economically viable?
   Ans: 
   Entry of Low MTTF is unprofitable for existing nodes 
   As if repair due to peer churn > revenue per provider 
   then the network is at a loss 
   If a provider earns 'x' but churn triggers '2x' repair costs then 
   good providers pay for bad providers 
   
3. What value of 't' should the system accept?
   Ans: 
   The paper promises 30x bandwidth savings for 24 hoour timeout 
   but we are compromising availability for partitioning in the network
   suggested period for now is 8-24 hours, but research is needed to 
   Finalise the valur of 't'
```
### Research paper 7
```java
Paper: SoK: Decentralized Storage Network
       Chuanlei Li, Minghui Xu, Jiahao Zhang, Hechuan Guo, Xiuzhen Cheng
       Shandong University | IEEE S&P

Research topics addressed: #1, #2, #5, #8, #13, #19

Problem solved: 
1. No single paper before it summarised the entire DSN landscape
2. Provides classification of major systems like Sia, Storj, Filecoin, Swarm
3. Names the failures each design choice creates
4. Points out how data updation is very complex in DSNs
5. Defines Proof of storage and consensus mechanisms
6. Pointed the biggest challenges 
   - File version control
	 - Network hijacking giving DoS (central entity risk)
	 - Privacy leaks through DHTs
	 - Illegal content cannot be traced in the network
	 - No access control, data owner cannot share access
	 - Honest Geppetto attack
	 - Bandwidth optimisations 

	 
Trade-offs:
1. Filecoin chose PoRep + PoSt over lightweight Merkle challenges
   requires 256 GB RAM + GPU with ≥11 GB VRAM, so all mobile and desktop 
   users eliminated; ruthlessly anti-sybil but only for Network Attached 
   Storage (NAS) operators
2. Storj chose reputation-gated vetting over cryptographic identity proofs
   So faster onboarding and lower hardware intensive participation
   but introduces "Honest Geppetto attack" 
3. Sia chose frequent Merkle challenges over PoS by Berlekamp-Welch; 
   lightweight for mobile users but can be bypassed by adversarial node
   having predicted the challenge timing
4. Swarm chose bandwidth-as-currency (SWAP), where payments take place to 
   settle large BW imbalances, clean for symmetric peer set and not for 
   Data owner & provider structure, it also breaks in production
5. All DSNs use cryptocurrency for payments
   

Breaks in our case:
1. Every surveyed DSN uses blockchain/cryptocurrency as the trust anchor
2. File coin uses heavy cryto challenges to prevent sybil attacks, we 
   instead use registation gating 
3. The Satellite architecture of Storj is centralised and prone to DoS attacks 
4. DHT lookups exposes CIDs

Decisions influenced:
1. Reconsider the decision of making the coordination microservice as
   the trust anchor instead of a public ledger like blockchain compromising
   on data owner trust; blockchain implements automatic verifiability. 
   We would have to create an audit trail
2. To prevent blockchain Storj finally implemented the Satellite which 
   reduced trust in the network; If we move with microservice architecture 
   then it would be a hardened service using 
3. To prevent DoS in central microservice the quorum read mechanism 
   can be implemented for consistency but introduces latency 
4. During DHT lookups via Kademlia, the content addressing must not 
   reveal the file identity 
5. Decision #2: Continuous Proof of Storage will be done using PoR 
                (Merkle challenges) as PoRep and PoSt are computationally
                heavy 
                Trasitory PoS would run at upload time when node signs 
                a receipt of the chunks stored 
                 
6. Decision #5: Storj's four subsystem reputation is adopted 
								1. Identity gate: registration with KYC/phone number 
								   limits Sybil node flood without cryptographic cost.
								2. Vetting: new nodes receive non-critical chunks under
								   high erasure redundancy to monitor and build trust								      
								3. Filtering: nodes failing audits or retrieval challenges
								   are downgraded, not immediately ejected
								4. Preference: after vetting, rank surviving nodes by 
								   throughput and latency measured during audits;
								   High-ranked nodes receive more new assignments
								   
7. Decision #19: Defence against the following attacks 
                1. Honest Geppetto attack: A single correlated provider
                   group (same subnet, same ISP AS number) must not hold 
                   multiple shards of the same file 
                2. Outsourcing attack: Use Filecoin's Seal idea 
                3. Just-in-time retrieval: Unpredictable challenge times
                

Disagreements:
1. Lakhani et al. 2022 & 2023: Do not model payment based on bandwidth
   exchange, as it failed for Swarm 
2. "Understanding Availability" (Bhagwan, Savage, Voelker, IPTPS 2003):
   Individual MTTF < nodes going offline together 

Open Questions after reading:
1. What is the lightest proof-of-storage that resists 
   just-in-time retrieval on a 5 Mbps mobile provider?
   Ans: 
		   1. Set response deadline = (chunk_size / declared_upload_speed) × 1.5
		   2. Randomise challenge timing so the provider cannot pre-cache
   
2. Does Storj's Honest Geppetto attack have a practical 
   mitigation that doesn't require centrally watching all nodes?
   Ans: Storj says
       1. Analyse the correlations between shards 
       2. If cluster C holds shards {i, j, k}
          |{i,j,k}| / n > ceiling (e.g. 20%)
          Then new assignments don't take place before major peer churn happens
       3. It's a placement constraint put during write time
       4. New networks have few shards, so implement only after 5 × n
   
3. Since we have no blockchain, what replaces its role as the 
   neutral audit trail that both data owners and providers trust? 
   Ans: Blockchain does three jobs
        1. Immutable audit log — provider submitted proof X at time T
        2. Automatic payment trigger — proof verified → escrow released
				3. Public dispute resolution — anyone can inspect the proof chain      
				We need to replicate 1. and 2. and find a substitute for 3.
				We can maintain a write-once audit log and all receipts of the 
				data owner and provider can be verified for disputes 
				Gives an auditable paper trail without blockchain
```
### Research Paper 8
```java
Paper:  Understanding Availability
        Ranjita Bhagwan, Stefan Savage, Geoffrey M. Voelker
        UC San Diego | IPTPS 2003 (Workshop on Peer-to-Peer Systems)

Research topics addressed: #5, #6, #8

Problem solved: 
1. Existing measurements and models do not capture the complex 
   time-varying nature of availability in peer-to-peer environments
2. Study availability in large P2P systems over the span of a week
3. Results affect the design of high-availability P2P services
	 
	 
Trade-offs:
1. 7-day trace only, making it risky to form concrete decisions
2. Studied Overnet and not Gnutella which is unstructured
3. Measured parameters based on HostID and not IP 

Breaks in our case:
1. The study population is voluntary file-sharers, not paid storage 
   providers so our availability is higher than 0.3 as proposed in the 
   paper
2. 6.4 joins/leaves per host per day was measured on a DHT, where 
   joining/leaving is free with zero state consequences, so financial 
   Friction would become our saviour

Decisions influenced:
1. Decision #8: The reliability score must weight recent behaviour more 
                Heavily but not discard long-term history entirely
                Maintain three rolling windows per provider — 24h, 7d, 30d 
                (period can be altered)
                weight the score as a weighted combination
                
2. Decision #6: The polling interval 't' must be set longer than the diurnal 
								absence window A provider offline for 6–8 hours (nighttime) 
								must not trigger repair. Minimum safe value of 't' is therefore 
								≥ 8 hours, ideally 12–24 hours, confirming the Blake & Rodrigues 
								suggestion
								
3. Decision #7: Follow two-component model (daily churn vs permanent departure) 
								maps directly to four exit states:
								Note: A lower bound of the threshold is maintained, crossing
								      which repair happens irrespective of 't'
							  1. Accidental/temporary absence (downtime) → no repair, wait for 't'
								   decrease reliability ()
							  2. Promised downtime → no repair, wait for the promised period
								   'p' to end, then only repair and penalise directly
							  3. Permanent silent departure → Crossing a warning limit
							     of time declared at the start of the session, failing which, we 
							     trigger repair, seize escrow earnings, node signed out
							     from network
							  4. Announced departure → Node announces his permanent exit
							     from the network; trigger repair, release pending escrow
							     Sign out the user from the network 

4. Decision #5: Independence of randomly selected hosts is confirmed
							  Random selection within a filtered pool (vetted,
							  geographically close) gives near-independent failure 
							  probabilities

Disagreements:
1. Saroiu et al. 2002 (Gnutella measurement study): their availability numbers are ~4x 
   lower than Bhagwan's because they used IP address probing  
   -> So we must be cautious when reading papers earlier than 2003 
2. Weatherspoon & Kubiatowicz 2002 (cited in paper): modelled 
   failures as independent
   -> So we must keep a small safety margin in erasure

Open Questions after reading:
1. What is the actual MTTF of a financially-incentivised 
   storage provider on a mobile device vs Bhagwan's 
   0.3-median voluntary peer?
   Ans: our own provider telemetry (no paper 
        answers this; must be measured empirically at launch)
        Just like Storj came up with 6-12 months MTTF

2. What polling interval 't' separates a sleeping provider 
   from a dead one, and what is the repair bandwidth cost 
   of getting it wrong in either direction?
   Ans: 't' is too short -> say 2hours, then overnight leaves would 
         also trigger repair 
         't' too long -> say 48hours, we may miss permanent departures
         't' sweet spot -> 12–24 hours (Blake and Rodrigues)
										       Start with 24 hours, then keep decreasing 
```
### Research paper 9
```java
Paper:  Feasibility of a Serverless Distributed File System 
        Deployed on an Existing Set of Desktop PCs
        William J. Bolosky, John R. Douceur, David Ely, Marvin Theimer
        Microsoft Research | ACM SIGMETRICS 2000

Research topics addressed: #1, #5, #6, #11

Problem solved: 
1. Measures feasibility using 51,662 machines at Microsoft over five 
   weeks
2. Clearly states it won't work on consumer desktops without optimisations
   only fine-tuned replication and downtime v/s departure separation 
   can help
3. Promises availability fails of just 0.3 to 3.0 (order of 1) per user
   per 1000 days
	 
	 
Trade-offs:
1. Simple replication over erasure for simpler design
2. Lazy update reduces write traffic but also takes away consistency 
   32MB/hr -> 7MB/hr
3. Replica placement by availability score (greedy algorithm)

Breaks in our case:
1. The paper focuses on the storage of files as done on a usual PC; we instead
   try to make a hyperscale cloud platform that relies on file updates and
   duplicate file checks
2. The machines researched are in a professional corporate environment, so
   It is less comparable to home users we are targeting, so 95% uptime
   and a 290-day lifetime is an upper bound 
3. The security ensured is much lower as the nodes are employees of the 
   same company
4. 50% free disk space was assumed in the 2000s; in today's world, it has become
   hypothetical

Decisions influenced:
1. Decision #7: Promised Downtimes and Announced departures would skyrocket
                reliability as the machines warn before leaving, providing
                time for reconstruction
                
2. Decision #7: Replication Trigger time for exits 
								Note: Time varies based on provider tier
								1. Accidental/temporary absence (downtime): 0–24h, we 
								   Wait till 72h before announcing a Permanent silent departure
								2. Promised downtime: Provide options from 0-72h, exceeding
								   declares Permanent silent departure
								3. Permanent silent departure: After 72h pass, perform 
								   replication
								4. Announced departure: instant 
       
3. Decision #11: Median CPU load of 1–2% and disk load means we can occupy 
								 5-10% of CPU for background audits and transfers without 
								 Hindering user experience 
								 
4. Decision #5: Provider tier list
								- Tier 1: NAS providers
								          MTTF: 290–380 days
								          Availability: 0.95
								          Storage: 30–70% of free disk
								          Payment: Premium       
			          - Tier 2: Standard desktop
								          MTTF: 180-290 days
								          Availability: 0.7
								          Storage: 30–40% of free disk
								          Payment: Standard
							  - Tier 3: Mobile users
													MTTF: 120-180 (minimum needed) days
								          Availability: 0.3-0.5
								          Storage: 10–15% of free disk
								          Payment: According to the service    
								          						  								        								        							       					
Disagreements:
1. Blake and Rodrigues 2003 stated that 24hr acceptable downtime, whereas Bolosky 
   suggests a bimodal figure for it 
2. Bhagwan et al. 2003 found availability to be 0.3, whereas Bolosky suggests 
   It is to be 0.9 because of the corporate atmosphere  

Open Questions after reading:
1. What is the correct repair trigger threshold for 
   desktop vs mobile providers, and what is the 
   bandwidth cost of getting it wrong?
   Ans: tier-specific thresholds are necessary
        Desktop: 72h (Bolosky)
			  Mobile:  24–36h (Bhagwan)
			  The bimodal distribution given by it was 
			  Nightly: µ=14h, σ=1.9h → 99.7% return within 20h
        Weekend: µ=64h, σ=2.1h → 99.7% return within 70h
  
2. How does the provider tier model change the erasure coding parameters
   Inherited from Storj?
   Ans: Storj used uniform parameters (k=29, n=80), assuming 
				a homogeneous NAS operator population
				We follow tier-based providers and find that 1 file must never 
				be present only with the mobile tier
				The desktop tier is the stability layer 
				Moreover we plan provider distribution based on the access 
				frequency (hot or cold storage), but the idea has not yet been 
				researched

3. Can convergent encryption be safely used for 
   deduplication in our zero-knowledge system, 
   And where does it break?	
   Ans: Convergent encryption derives the encryption key from the hash
        of the plaintext
        So two files having the same hash are duplicates 	
        This is needed for the deduplication of files in the network
        It reveals the access patterns without decrypting the content 
        compromising security
        Moreover we don't need deduplication as it's the property of the 
        data owner to store and pay for all its content 
        There is no need to catch duplicates
```
### Research paper 10
```java
Paper:  Peer-to-Peer Storage Systems: A Practical Guideline to Be Lazy
        Frédéric Giroire, Julian Monteiro, Stéphane Pérennes
        MASCOTTE joint project INRIA / I3S (CNRS, Univ. of Nice-Sophia)
        IEEE GlobeCom 2010 

Research topics addressed: #3, #4, #6, #17

Problem solved: 
1. Provides actual mathematical formulas for the entire DSN
2. Helps in deciding the delay in the trigger of repair and deciding the erasure 
   parameters 
3. Formulas: 
   a. Average BW consumption 
      = ((B·α) / (N · ln(s+r / s+r0) · τ) · (s + r − r0 − 1) · lf
   b. Data loss rate 
      = B/((s+r0+1)·ln(s+r/s+r0)·τ)·(s+r0)!/(s-1)!·(α/γ)^(r0+2)
   c. Best BW consumption for given s and r0
      = r0 − s − r + (s + r) · ln(s + rs / s + r0) = 0
   
	 
Trade-offs:
1. Lazy repair over eager repair, as BW savings grow logarithmically
   with gap in r-r0 (gap of 8 reduces BWavg by ~3× vs gap of 1) but
   the data loss probability increases
2. Markov chain model assumes independent peer failures; we use the 
   diversity cap from SoK to amke this assumption work
3. Has fixed timeout for peer loss and not the diurnal churn
4. B ≥ N² constraint requires large data volumes, for small networks 
   it gets difficult 

Breaks in our case:
1. Assumes homogenous peer distribution and not the tier-wise population
   which changes MTTF 
2. Assumes peer reenters with no data, the promised downtime allows
   it to them
3. No. of peers N and the network storage capacity D are kept constant 
   whereas they increase with time 

Decisions influenced:
1. Decision #3 : Our target is LossRate < 10^-15 per year
								 We fix 
								 initial fragments(s)=16, frament size(lf)=256KB
								 repair threshold(r0)=8, redundancy fragments=40
								 Bavg comes out to be 39 Kbps/peer
								 
2. Decision #4: We use r0 to refer to the repair trigger threshold 
								It can be derived from the formulas given in the paper 

3. Decision #17: The peak bandwidth formula Qpeek is the key input 
								  for sizing our repair bandwidth budget; For each 
								  provider failure event, the network must absorb
								  Qpeek/θ bandwidth over reconstruction window θ

Disagreements:
1. Dimakis et al. 2007: Shows that standard RS repair is not bandwidth-optimal

Open Questions after reading:
1. What is the repair bandwidth spike Qpeek when a mobile provider
   fails, and how long must repair complete to avoid data loss?
   Ans: For N=1000, B=25M, s=16, r=40, r0=8, lf=256KB
        Qpeek=793 GB per peer
        
        At 100Kbps per peer
        It takes 8hours to consume the loss of one peer 
        which is less than our 12hour reconstruction window 'θ'
        So the MTTF must not fall any more lower 
   
 2. What is the optimal lazy repair strategy for our three-tier 
    mixed network, and does it make sense to have different r0 
    per tier?
    Ans: 1. Simpler approach: use a single r0 calibrated to 
						the most volatile tier (mobile), giving desktops 
						more buffer than they need but simplifying the 
						repair state machine
				  
				 2. Sophisticated approach: separate shard pools per tier
				    A file's n=56 shards are assigned as:
					  - 20 shards to Tier 1 (desktop) with r0_desktop=12
					  - 36 shards to Tier 3 (mobile) with r0_mobile=8
					  gives optimal bandwidth per tier but requires 
					  the repair scheduler to track two independent 
					  Markov chains per block	     
```
## Exclusive Decisions made 
```java
1. Decision #5 (Peer Selection Algorithm)
   The network would not involve mobile phones in V2 and would be 
   implementing a desktop-first approach in design
   Papers which support the decision:
		   1. Storj: MTTF to be 8-12 months
		   2. Blake et al.: Low MTTF harms the existing nodes with BW usage
		   3. Bhagwan et al.
	 The background operations of Android and IOS are strict, limiting 
   The transfer is possible between devices 
		   
	 Note: Mobile phones can be included in V3 after studying the
	       performance of V2
	       
	 With this decision, the provider tier collapses completely 
	 as our 
	 NAS operators:        MTTF ~290–380 days
   Corporate desktops:   MTTF ~270–300 days
   Home desktops:        MTTF ~180–270 days
   have a close range of MTTFs so they could have the same erasure 
   parameter and repair thresholds 
   Instead of tiering, we function using a dedicated 4-6-month vetting
   period
   We still enforce the ASN/subnet diversity cap even within the 
   single tier to avoid Honest Geppetto attack
```

```java
2. Decision #2 (Proof of Storage)
   The audit trail would be kept using Signed Receipt Exchange 
   Every audit event produces two signed artefacts:
		  1. Provider receipt:
		     {chunk_id, audit_challenge, response_hash, timestamp}
		     with their registered private key
		     → proves "I responded to this challenge at this time"
		     It keeps receipts in a local receipt store on their machine
		  2. Microservice countersignature:
		     {provider_receipt_hash, pass/fail, timestamp}
		     → proves "we recorded this result"
		     It keeps receipts in an append-only Postgres table 
		     (INSERT only, no UPDATE/DELETE, enforced at DB level)
		
	 The Signed Receipt Exchange is cryptographically strong; the issue 
	 is that the microservice may drop an audit trail, and the provider
   has no way to rectify it
   
   In V3, we would be implementing the Transparent Merkle Log over 
   Signed Receipt Exchange
   With it 
   Any provider can run an inclusion proof against the published daily
	 root, which is the same guarantee Certificate Transparency provides 
	 for TLS
	 
	 Note: It might not convince NAS providers at once over Blockchain
	 
	 The coordination required here is very low as only once DB write is
   needed per audit 
   Flow: provider sends response → service validates → service inserts one row
         → service returns countersignature
     
    We would be facing a problem while Merkle root construction, so
    build the tree from a timestamped snapshot, not a live query
    like Kafka Compaction
    Blockchains on the other hand wait for every on-chain write 
    to propagate to a consensus before it is final
```

```java
3. Decision #2 (Proof of Storage)
The Receipt Schema
It is an architectural decision that cannot be altered once instated
Structure:
 {
  schema_version:   1
  chunk_id:         SHA256(plaintext_chunk)  // content address
  file_id:          pseudonymous file handle  // not plaintext CID
  provider_id:      registered account UUID
  challenge_nonce:  HMAC(server_secret, chunk_id + server_ts)
  server_timestamp: unix_ms from challenge issuance
  response_hash:    SHA256(chunk_data || challenge_nonce)
  response_latency: ms between challenge sent and response received
  provider_sig:     Ed25519(SHA256(all above fields))
  service_sig:      Ed25519(SHA256(provider_sig + service_ts))
  service_timestamp: unix_ms of countersignature
}
 
The current proposed structure is identical to AWS CloudTrail, 
Google Certificate Transparency logs
Problems 
1. Clock skew attacks
	   - Provider alters local clock
	   - Fix: By OCSP (RFC 2560)
	          Nonce must include a server-generated 
						timestamp component that the provider cannot predict 
						before the challenge is issued
						challenge_nonce = HMAC(server_secret, 
		                         chunk_id + server_timestamp)
		                         
2. Receipt replay
      - Provider replays receipts for two distinct chunks
      - Fix: By TLS 1.3
             SHA256(chunk_id || challenge_nonce || 
                    response_hash || timestamp)
						 Not just response_hash alone
						 
3. Service countersignature timing gap
		  - Provider claims he has responded, whereas the microservice had 
		    crashed
		  - Fix: By Kafka
		         Service writes to an append-only log, 
             then sends countersignature
             Provider's receipt is only considered valid 
					   after receiving the service countersignature   
```

```java
4. Decision #18 (Economic Mechanism Design)
	 The network would be exclusively for India at launch
	 and would use UPI via Razorpay or Cashfree
	 This would 
			 - Ensure geographic proximity, reducing latency 
			 - The need for currency conversion drops
			 - UPI has no per-transaction fee for the merchant
			 - UPI settles in seconds; Stripe Connect could take days
			 - Stripe account or PayPal creates friction in onboarding
			   whereas UPI is readily available with all providers 
			   
		But we would be designing a payment extraction layer from the start
		PaymentProvider interface:
			  initiateEscrow(data_owner_id, amount, contract_id)
			  releaseEscrow(provider_id, amount, audit_period)
			  penalise(provider_id, amount, reason)
			  getBalance(entity_id)
```

```java
5. Decision #5 (Peer Selection Algorithm)
	 Segregate providers as Hot and Cold based on Storage capability 
		 - An explicit consent would be taken from the providers at the 
		   time of joining
		 - We perform the vetting according to the consent, validating 
		   the claim
		 - Similarly, the Data owner too specifies his requirement of either
		   a hot of cold service and is charged accordingly 
		 - Hot Storage band:
			   - Low storage 
			   - High Bandwidth low latency nodes
			   - Lower space overhead in erasure (smaller chunks)
			   - Payment basis: per GB stored + per retrieval
		-- Cold Storage band:
				 - High storage capacity
				 - High MTTF
				 - Low Bandwidth acceptable 
				 - High space overhead in erasure (big chunks for safety)
				 - Payment basis: per GB stored per month
			
		 - For V2 we can try implementing just the Cold storage architecture
		   and add the Hot storage layer in V3
```

```java
6. Decision #13 (Escrow & Payment Basis)
	 Providers would be paid for passing audits and not for a successful 
	 retrieval to the data owner 
	 This ensures
	   - High MTTF 
	   - The payment mechanism runs via the microservice 
		   Whereas the storage is maintained by pure P2P transfers 
		   These two must not collide, else it would create a credit 
		   liability as soon as the microservice goes down
	
	 - The payment microservice must use the idempotency key used by 
	   Razorpay/Stripe to avoid the double-payment race condition
	 - We must maintain a minimum escrow floor in case of an unexpected
	   increase in repair costs; ensuring payment to the providers in all
	   scenarios
	 - There remains an authentication gap in the pure P2P 
	   transfers that need to be researched extensively 
```

## Research papers to continue reading
### Phase 0
    
Motive: 

Get a fair idea about developments in the P2P file transfer & distributed storage networks domain

| # | Title | Type | Priority | Topics | Key Extraction Target |
| --- | --- | --- | --- | --- | --- |
| 1 | Incentives Build Robustness in BitTorrent, Bram Cohen, May 22, 2003 | Paper | MANDATORY | #1 | •  |
| 2 | Kademlia: A P2P information system based on XOR metric, Peter Maymounkov & David Mazieres, New York University | Paper | MANDATORY | #1 | •  |
| 3 | S/Kademlia: A Practicable Approach Towards Secure Key-Based Routing, Ingmar Baumgart and Sebastian Mies, Institute of Telematics Universitat Karlsruhe, Germany | Paper | MANDATORY | #1 | •  |
| 4 | IPFS - Content Addressed, Versioned, P2P File System, Juan Benet, 2015 | Paper | MANDATORY | #1,#2 | •  |
| 5 | Storj V3 whitepaper  | Whitepaper | MANDATORY | #1,#2,#13 | •  |
    
### Phase 1
    
Coordination and storage systems

Decide #1 Coordination Architecture & #2 Proof of Storage

| # | Title | Type | Priority | Topics | Key Extraction Target |
| --- | --- | --- | --- | --- | --- |
| 1 | High Availability, Scalable Storage, Dynamic Peer Networks: Pick Two (Blake & Rodrigues, HotOS 2003) | Paper | MANDATORY | #1 | • The P2P storage trilemma — pick two of three properties • Read before Storj: makes every Storj design choice legible • Identify which two properties your V2 picks and document why |
| 2 | SoK: Decentralised Storage Network (Li et al. — IEEE S&P / CCS) | Paper | MANDATORY | #1–3,#5,#13 | • Landscape map: all major DSN systems compared on one framework • Extract the failure taxonomy — why each system collapsed • Map your architecture against their comparison axes |
| 3 | "Understanding Availability in Peer-to-Peer Applications"
Bhagwan, Savage, Voelker — IPTPS 2003 | paper | MANDATORY | #6, #5, #8 | • correlated failure model; availability ≠ individual uptime |
| 4 | Feasibility of a Serverless Distributed File System Deployed
on an Existing Set of Desktop PCs"
Rodrigues, Blake, Liskov — ACM SIGMETRICS 2000 | paper | RECOMMENDED | #1, #6, #11 | voluntary participation floor; admission control necessity |
| 5 | P2P Storage Systems: A Practical Guideline to Be Lazy (Giroire et al., GlobeCom 2010) | Paper | MANDATORY | #1,#4,#6 | • 'Lazy' repair = defer repair until redundancy falls below threshold • Quantifies repair cost vs redundancy decay tradeoff • Sets your repair trigger threshold decision |
| 5 | Coordination Avoidance in Database Systems (VLDB 2015) | Paper | MANDATORY | #14 | • Crucial to study the requirement for blockchain
Invariant confluence framework for deciding strong vs eventual consistency |
| 6 | Dynamo: Amazon's Highly Available Key-Value Store (SOSP 2007) | Paper | MANDATORY | #4,#6 | • N/R/W quorum model • Gossip-based failure detection |
| 7 | libp2p Specification & Architecture | Docs | MANDATORY | #1,#12 | • Kademlia implementation + NAT traversal |
| 8 | QUIC: A UDP-Based Multiplexed and Secure Transport (RFC 9000) | RFC | MANDATORY | #12 | • Connection migration and multiplexed streams |
| 9 | Separation and Optimization of Encryption + Erasure Coding (Szabó et al., FGCS Feb 2025) | Paper | MANDATORY | #3,#9,#15 | • BLOCKING: must decide encrypt-then-code vs code-then-encrypt before chunk design • Quantifies key management overhead of each choice • Directly targeted paper for Topic #15 decision |
| 10 | Filecoin Whitepaper (Protocol Labs, 2017) | Whitepaper | MANDATORY | #1,#2,#13 | • Sections 3.1–3.2 only: PoRep and PoSt mechanisms • Skip all token/blockchain sections • Audit gap closure mechanism — extract the proof structure, not the crypto |

### Phase 2A
    
Finalise the Erasure Coding mechanism for hot and cold storages

Decide #3 Erasure Coding and #15 Encryption-Erasure Interaction (if possible)

| # | Title | Type | Priority | Topics | Key Extraction Target |
| --- | --- | --- | --- | --- | --- |
| 11 | Coral DSHT (Freedman, Mazières, NSDI 2004) | Paper | MANDATORY | #1,#5 | • Sloppy DHT: store pointers not values — your exact architecture • Geographic clustering for provider selection • Section 3 (hierarchical clustering) is the key extraction target |
| 12 | Design & Evaluation of IPFS (Trautwein et al., SIGCOMM 2022) | Paper | MANDATORY | #1,#5,#6 | • Real production measurements: lookup latency, content decay, churn rates • Sets empirical floor for your redundancy and polling interval decisions • Section 5 (availability decay) is the most important section |
| 13 | Measurement Study of P2P File Sharing (Saroiu et al., 2002) | Paper | MANDATORY | #3,#5,#6 | • Median node session < 1 hour — empirical basis for redundancy factor k • Table 2–4: actual uplink/downlink distributions for consumer devices • Grounds your provider tier model in measured reality |
| 14 | Survey of the Past/Present/Future of Erasure Coding (Shen, Lee et al., ACM ToS Dec 2024) | Paper | MANDATORY | #3 | • Read this first — landscape map for all EC schemes • RS vs LRC vs MSR vs MLEC comparison table • Tells you which of the 6 repair papers below are worth your time |
| 15 | Minimum Storage Regenerating Codes (Dimakis et al., 2010) | Paper | MANDATORY | #3,#17 | • FOUNDATIONAL: proves the information-theoretic lower bound on repair bandwidth • MSR codes: minimum storage AND minimum repair bandwidth simultaneously • All subsequent repair papers build on this result |
| 16 | Erasure Codes for Cold Data in Distributed Storage Systems (Chao Yin et al.) | Paper | MANDATORY | #3 | • Consumer device storage = cold data model • Storage overhead vs reconstruction speed tradeoff • Compare their optimal (k,m) against your Saroiu-derived redundancy requirement |

### Phase 2B
    
Repair Bandwidth optimisation

Decide #4 Replication / RepairProtocol & #17 Repair Bandwidth Optimisation

| # | Title | Type | Priority | Topics | Key Extraction Target |
| --- | --- | --- | --- | --- | --- |
| 17 | EigenTrust: Managing Trust in P2P Networks | Paper | MANDATORY | #8 | • Global reputation from local interactions |
| 18 | TrustDSN: Reputation Without Blockchain for Distributed Storage | Paper | MANDATORY | #8,#19 | • Reputation system without blockchain |
| 19 | Handling Churn in a DHT | Paper | MANDATORY | #1,#6 | • DHT churn handling strategies |
| 20 | IRON File Systems (SOSP 2005) | Paper | MANDATORY | #19 | • Adversarial provider simulation: partial writes, silent corruption |

