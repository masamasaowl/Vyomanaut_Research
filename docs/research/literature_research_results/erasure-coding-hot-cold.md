# Search term - erasure coding hot cold data tiering

## Number 1

**Name:**
HALO: A scalable framework for hotness-aware coding and transformation-efficient placement

**Publisher:**
Future Generation Computer Systems, October 2026

**Authors:**
Junmei ChenNe Wang, Dan Xiang

**Abstract:**
Modern erasure-coded storage systems aim to balance reliability, performance, and storage efficiency under dynamic and skewed access patterns. We present HALO (Hotness-Aware Layout and Redundancy Optimization), a middleware framework that enhances HDFS by jointly adapting redundancy and block layout, based on hotness awareness and data correlation. HALO integrates three core components: (i) a lightweight hotness predictor to classify data blocks, (ii) a non-uniform locally repairable code (NU-LRC) engine that dynamically adjusts redundancy per tier, and (iii) a graph-guided layout optimizer that places correlated blocks to minimize cross-rack repair cost. We implement HALO atop a tailored HDFS and evaluate it using real-world traces (MSR, Alibaba) and extended synthetic workloads (maritime and urban traffic). Compared to state-of-the-art schemes such as Static/Dynamic NU-LRC, U-LRC, AERS, and dynamic replication, HALO achieves up to 50% lower repair latency, 40% lower reduction in cross-rack repair traffic, and 22% lower savings in storage overhead. Furthermore, it exhibits faster adaptation to workload shifts while maintaining high prediction accuracy and minimal re-encoding overhead.

---

## Number 2

**Name:**
Effeclouds: A cost-effective cloud-of-clouds framework for two-tier storage

**Publisher:**
Future Generation Computer Systems, April 2022

**Authors:**
Mingyu LiuLi PanShijun Liu

**Abstract:**
Cloud storage is attracting more and more users with its virtually infinite capacity. However, users have to confront challenges of service outages, price hikes and vendor lock-in. Data replication is a widely used approach to overcome these challenges, but it may bring reduplicated growth in costs. Tiered-storage, which springs up in recent years, offers an opportunity for cost optimization. The hot tier charges users a higher price of storage but a lower price of operations, and vice versa for the cold tier. For optimizing costs, users can transfer their data between hot and cold tiers according to the prior knowledge of data’s future access patterns which, however, is generally very hard to obtain in practice. Therefore, it is cost-risky to transfer data rashly because data would incur more costs if they are stored in inappropriate tiers. To achieve cost-effectiveness and high data availability, in this paper we propose a cost-effective framework, Effeclouds, which leverages erasure coding to stripe data across multiple clouds redundantly, in addition to an online algorithm with proved competitiveness to further optimize costs by selecting appropriate tiers for data. Effeclouds offers users a simple way to deploy and exploit, and simultaneously gives consideration to both cost-effectiveness and data-availability. It also mitigates the effect of vendor lock-in and price hikes on costs with multiple clouds as the back-end. Eventually, we validate the effectiveness of Effeclouds by extensive simulations driven by real-world traces.

---

## Number 3

**Name:**
A comprehensive repair scheme for distributed storage systems

**Publisher:**
Computer Networks, November 2023

**Authors:**
Junmei ChenZongpeng LiXianglong Li

**Abstract:**
Modern data storage systems apply erasure codes to provide data reliability efficiently. Previous studies proposed a series of techniques to weigh repair/storage costs, reduce codec complexity, minimize repair time, improve fault tolerance, and enforce system-level service level agreement. These techniques have been designed in isolation, leading to performance limitations. We explore the potential advantages of combining these techniques to meet data storage systems’ requirements better and provide superior system performance. This work proposes a comprehensive repair scheme for fault data in distributed storage systems. First, we tailor design erasure codes in the presence of heterogeneity of storage devices. The core idea is to monitor device performance (e.g., access speed, reliability), compute two coefficients for each device, and use them to select the appropriate devices to create stripes of erasure codes. Second, we leverage the system hierarchy to perform intermediary repair operations, further minimizing cross-rack repair bandwidth. Finally, we propose a new repair scheme adapted to the skew of data access. To demonstrate the effectiveness of our comprehensive repair scheme, we evaluate various erasure codes via mathematical analysis and experiments in the Ceph cluster. In the mise-en-scène of traditional re-encoding methods and more recent adaptive erasure codes, our scheme stands out with significant savings in recovery bandwidth, code-switching bandwidth, repair time, and code-switching time.

---

## Number 4

**Name:**
Online dynamic replication and placement algorithms for cost optimization of online social networks in two-tier multi-cloud

**Publisher:**
Journal of Network and Computer Applications, April 2024

**Authors:**
Ali Y. AldailamyAbdullah MuhammedWaidah Ismail

**Abstract:**
In social media, a huge number of worldwide data objects are posted every day. The contents of these data objects include text, links, images, audio, and videos which could be small, medium, or large and accessed across the world. Moving these data objects into a single cloud service provider (CSP) is risky and results in four-fold obstacles: vendor lock-in, service availability, cost-ineffective use, and increasing latency. Using multiple CSPs to replicate and distribute the data object solves such obstacles. However, replicating data objects among multiple CSPs increases the cost of creating and maintaining this replication. This study focuses on three issues of Online Social Network (OSN) which include: (1) determining the appropriate number of replicas of each data object based on its popularity on the OSN, (2) identifying the suitable datacenters that host the replicas according to latency time of different regions, and (3) deciding the suitable storage class for the data object at a specific time of its lifetime. Two algorithms are proposed to adapt the replication and placement of the data object according to its popularity in the OSN. The first algorithm is Dynamic Fixed Time (DFT) which uses fixed time periods to adapt replication and placement. The second algorithm is Dynamic Exponential Time (DET) which determines the data object replication and placement based on exponential time periods. A simulation using a synthesized workload generated based on a real Facebook statistic dataset shows that the proposed algorithms produce a monetary cost savings of more than 23% compared to the Static Replication and Local Placement (SRLP) algorithm.

---

## Number 5

**Name:**
A bi-level programming model to enhance the encoding efficiency of erasure codes

**Publisher:**
Expert Systems with Applications, 15 December 2026

**Authors:**
Jiexin ChenZhongbo HuXinyi Zhang

**Abstract:**
The encoding efficiency of erasure codes is traditionally optimized by two independent programming models designed to minimize the count of ones in the generator matrix and the XOR scheduling count. However, when the objective functions of these two programming models are inherently unable to reach the optimal state simultaneously, solving them independently fails to achieve global optimality. This article reveals, for the first time, that the minimum count of XOR scheduling operations is not always achieved at the maximum sparsity state of a bit matrix. To address this issue, this article proposes a bi-level programming model for bit-matrix sparsity and XOR scheduling count (BiLP-BX), which is designed to pursue the encoding efficiency of erasure codes. Within the BiLP-BX model, the lower-level program minimizes XOR scheduling operations subject to encoding feasibility constraints, while the upper-level program minimizes the count of ones in the corresponding bit matrix. Since a single XOR operation generally involves multiple non-zero elements, BiLP-BX strictly prioritises XOR minimisation to derive the highly efficient coding matrices. The BiLP-BX model is solved using a newly developed iterative greedy algorithm based on a penalty mechanism and tabu search (IG-PTS). Overall numerical evaluation shows that BiLP-BX has an encoding throughput improvement of up to 442.8% compared with Jerasure, and is significantly higher than the maximum gain of 377.8% achieved by the state-of-the-art Cerasure library. These results demonstrate that the proposed bi-level paradigm provides an effective approach for improving the practical encoding efficiency of erasure codes.

---

**Name:**
HCFR: A pipeline-free hybrid CPU-FPGA acceleration architecture for small-file erasure coding recovery

**Publisher:**
Journal of Systems Architecture, September 2026

**Authors:**
Fan LeiYong WangQian He

**Abstract:**
Distributed storage systems face significant challenges in recovering massive amounts of small files, where the high start-up latency of traditional deep-pipeline hardware and the context-switching overhead of pure software solutions severely constrain real-time performance. To address this issue, this paper proposes HCFR, a hardware-software co-designed hybrid architecture. By constructing a “pipeline-free” FPGA decoder based on pure combinational logic to eliminate pipeline fill and drain overheads, and integrating CPU-based binary decomposition for matrix inversion with a double-buffering latency hiding mechanism, the proposed architecture achieves pipeline-level parallelism. Experimental results show that, for extremely small objects such as 32 Bytes, HCFR achieves up to 18.75× core decoding speedup over Jerasure and up to 14.30× over Intel ISA-L. In terms of end-to-end recovery throughput, HCFR achieves up to 14.52× improvement over Jerasure and up to 7.68× over Intel ISA-L under the evaluated RS configurations. Furthermore, it achieves approximately 9× higher energy efficiency over the evaluated CPU-based software implementation, effectively breaking the performance barrier associated with small data block recovery. This study presents an efficient and viable technical pathway for mitigating tail latency in heterogeneous storage systems, offering broad application prospects for next-generation low-latency cloud storage and edge computing infrastructures.

---

## Number 6

**Name:**
ZJC: Constructing fully local repair in erasure codes for distributed cloud storage

**Publisher:**
Journal of Systems Architecture, May 2026

**Authors:**
Zijian ZhouXiaoheng DengKaiping Xue

**Abstract:**
Achieving efficient repair with minimal redundancy remains a long-standing challenge for distributed cloud storage. Existing erasure codes such as Reed–Solomon (RS) and Local Reconstruction Codes (LRC) rely on global repair, which causes heavy repair bandwidth consumption and high node construction overhead. This paper presents ZJ Codes (ZJC), a novel erasure coding scheme that achieves fully local repair while maintaining strong availability. ZJC introduces an interleaved local-group structure in which each data block participates in two local parity groups, thus providing two independent recovery paths without any global parity. A new storage overhead model is further proposed, incorporating node construction cost into redundancy evaluation. We analytically prove that ZJC achieves lower storage overhead and constant repair bandwidth with only two helper blocks per repair. Both RS-based and XOR-based implementations are developed and experimentally deployed in a distributed system. Results show that ZJC improves encoding efficiency by up to 87.5% and recovery throughput by 96.5% compared with Reed–Solomon codes at equivalent redundancy. Compared to LRC, ZJC achieves encoding improvements of 62.3% and 86.5%, respectively. RS-based and Xor-based ZJC show recovery improvements of 53.9% and 96.5% compare with RS, respectively. Additionally, Xor-based ZJC outperforms LRC and ZJC RS-based by 77.5% and 90.8%, respectively. These results demonstrate that ZJC effectively reconciles the long-standing trade-off between low redundancy and high repair efficiency in distributed cloud storage.

---

## Number 7

**Name:**
Cognitive Erasure-Coded Data Update and Repair for Mitigating I/O Overhead

**Publisher:**
Computers, Materials and Continua, 9 December 2025

**Authors:**

**Abstract:**
In erasure-coded storage systems, updating data requires parity maintenance, which often leads to significant I/O amplification due to “write-after-read” operations. Furthermore, scattered parity placement increases disk seek overhead during repair, resulting in degraded system performance. To address these challenges, this paper proposes a Cognitive Update and Repair Method (CURM) that leverages machine learning to classify files into write-only, read-only, and read-write categories, enabling tailored update and repair strategies. For write-only and read-write files, CURM employs a data-difference mechanism combined with fine-grained I/O scheduling to minimize redundant read operations and mitigate I/O amplification. For read-write files, CURM further reserves adjacent disk space near parity blocks, supporting parallel reads and reducing disk seek overhead during repair. We implement CURM in a prototype system, Cognitive Update and Repair File System (CURFS), and conduct extensive experiments using real-world Network File System (NFS) and Microsoft Research (MSR) workloads on a 25-node cluster. Experimental results demonstrate that CURM improves data update throughput by up to 82.52%, reduces recovery time by up to 47.47%, and decreases long-term storage overhead by more than 15% compared to state-of-the-art methods including Full Logging (FL), Parity Logging (PL), Parity Logging with Reserved space (PLR), and PARIX. These results validate the effectiveness of CURM in enhancing both update and repair performance, providing a scalable and efficient solution for large-scale erasure-coded storage systems.

---

## Number 8

**Name:**
On the data persistency of replicated erasure codes in distributed storage systems

**Publisher:**
Information and Computation, May 2025

**Authors:**
Roy FriedmanRafał KapelkoKarol Marchwicki

**Abstract:**
This paper studies the fundamental problem of data persistency for a general family of redundancy schemes, called replicated erasure codes. In replicated erasure codes each document is divided into p chunks and then encoded into p+q chunks. Then, each of the p+q chunks is replicated into r replicas. We analyze two strategies of replicated erasure codes distribution: random (all chunks are spread randomly among storage nodes) and sequential (the chunks are sequentially placed into storage nodes). For both strategies we derive closed-form expression and asymptotic bounds for expected data persistency of replicated erasure codes when the storage nodes leave the storage system and erase their locally stored data. We observe that the maximal expected data persistency of replicated erasure codes for both placement strategies is attained for parameter p=1 and give formulas in terms of the beta function in this case.

---

## Number 8

**Name:**
In-network aggregation enabled multiple sub-blocks parallel repair in erasure-coded storage system

**Publisher:**
Computer Networks, October 2025

**Authors:**

**Abstract:**
Erasure coding has gained widespread adoption in large-scale distributed storage systems since it can significantly reduce storage overhead while ensuring high reliability. However, repairing failed data in erasure-coded systems requires retrieving data from multiple nodes, which generates substantial network traffic, and often leads to incast congestion and degraded repair performance. Existing solutions alleviate requester-side congestion by offloading aggregation operations to helpers (nodes that provide repair data), but they inevitable increase inter-helper traffic and still struggle to fully utilize global network resources. To this end, we propose lnaPR (In-network Aggregation Enabled Parallel Repair for Multiple Sub-Blocks), a framework that leverages programmable switches to perform in-network aggregation during data repair. InaPR decomposes a data repair task into multiple tree-structured pipelines, enabling data repair to collect source data from more helpers beyond the fixed k-nodes requirement. Then, the bandwidth allocation for each pipeline is optimized through a two-stage methodology: (1) a heuristic helper allocation strategy that assigns high-bandwidth helpers across multiple pipelines while distributing low-capacity ones among distinct pipelines; (2) a throughput-maximizing bandwidth allocation formulated as a linear programming model. Furthermore, we also extend the architecture to cross-rack scenarios through virtual node decomposition. Finally, we prototype lnaPR using a P4-programmable switch and validate its performance in real-world evaluations and multi-rack simulations. Experimental results demonstrate that InaPR achieves 6.74% higher repair throughput than state-of-the-art methods in single-rack prototype tests and an 11.03% improvement in cross-rack simulations.

---

## Number 9

**Name:**
Separation and optimization of encryption and erasure coding in decentralized storage systems

**Publisher:**
Future Generation Computer Systems, June 2025

**Authors:**
Marcell SzabóÁkos RecseMarkosz Maliosz

**Abstract:**
Entering the cloud storage market requires a high upfront investment, thus it is dominated by a few players with existing capacity. Decentralized cloud storage solutions can disrupt the status quo by allowing businesses and individuals to sell their unused storage capacity, reducing the need for large upfront investments in service infrastructure. We show that network operators providing such service can significantly decrease the traffic volume carried on the transport network, which is essential when serving mobile users, while maintaining high data security by implementing our proposed solution, of leveraging controlled replication inside the core network. Upon data uploads encryption and erasure encoding are separated, with the latter moved inside the network, enabling the arbitrary replication of storable data pieces without straining the access network. We present simulation results, showing that the proposed method reduces traffic by 20% compared to the out-of-the-box solution. Moreover, we elaborate on optimal multi-proxy placements and even optimal storage node choosings in complex ISP networks, where deep data penetration is desired, by giving ILP optimization methods and results, achieving minimal overall network load and maximum data security.

---

## Number 10

**Name:**
Zebra: Demand-aware erasure coding for distributed storage systems

**Publisher:**
2016 IEEE/ACM 24th International Symposium on Quality of Service (IWQoS), 20-21 June 2016

**Authors:**
Jun Li, Baochun L

**Abstract:**
Erasure coding has been increasingly replacing replication in distributed storage systems, thanks to its lower storage overhead with the same level of failure tolerance. However, with lower storage overhead, the reconstruction overhead of erasure codes can increase significantly as well. Under the ever-changing workload, in which the data access can be highly skewed, it is difficult to achieve a well trade-off between the storage overhead and the reconstruction overhead. In this paper, we propose Zebra, a framework that encodes data into multiple tiers by their demand. Given the overall storage overhead and the number of failures to tolerate, Zebra determines the parameters of erasure coding in each tier by solving a geometric programming problem. Based on the demand of data, Zebra can dynamically assign data into the corresponding tiers to minimize the overall reconstruction overhead, and achieve a flexible tradeoff between the storage overhead and the reconstruction overhead in multiple tiers, such that hot data can enjoy less overhead of reconstruction and cold data can be stored with lower storage overhead. When demand changes, Zebra can adjust itself accordingly with a marginal amount of network transfer.

---

## Number 11

**Name:**
Localitycache: Toward efficient straggler tolerance in LRC-coded storage via caching local parity blocks

**Publisher:**
High-Confidence Computing, March 2026

**Authors:**
Ximeng ChenSi WuYinlong Xu

**Abstract:**
Modern distributed storage systems increasingly employ Locally Repairable Codes (LRCs) to provide reliable, low-cost data storage with high repair efficiency. However, the presence of stragglers, i.e., nodes that unpredictably slow down, can significantly impact access latency. Traditional approaches for handling stragglers, such as detection, blacklisting, or speculative execution, are often insufficient for efficient straggler tolerance. In this paper, we show how an in-memory caching strategy coupled with LRCs can bypass stragglers without relying on precise straggler detection. We propose LocalityCache, a novel in-memory caching mechanism designed for LRC-coded distributed storage systems, which effectively mitigates the impact of stragglers by caching local parity blocks. We provide theoretical guarantees for LocalityCache and show that caching local parity blocks minimizes the likelihood of encountering stragglers. Additionally, we devise optimized workflows for write, read, and repair operations under LocalityCache to ensure system efficiency. We implement LocalityCache in a distributed key–value store prototype atop Redis. Our extensive testbed evaluations show that LocalityCache can significantly reduce read latency of the baselines by up to 73.6% in the presence of stragglers.

---

## Number 12

**Name:**
POCache: Toward robust and configurable straggler tolerance with parity-only caching

**Publisher:**
Journal of Parallel and Distributed Computing, September 2022

**Authors:**
Mi ZhangQiuping Wang, Patrick P. C. Lee

**Abstract:**
Stragglers (i.e., nodes with slow performance) are prevalent and incur performance instability in large-scale storage systems, yet it is challenging to detect stragglers in practice. We make a case by showing how erasure-coded caching provides robust straggler tolerance without relying on timely and accurate straggler detection, while incurring limited redundancy overhead in caching. We first analytically motivate that caching only parity blocks can achieve effective straggler tolerance. To this end, we present POCache, a parity-only caching design that provides robust straggler tolerance. To limit the erasure coding overhead, POCache slices blocks into smaller subblocks and parallelizes the coding operations at the subblock level. It further adopts a configurable straggler-aware cache algorithm (CSAC) that takes into account both file access popularity and straggler estimation to decide which parity blocks should be cached. CSAC enables POCache to configure various cache admission and eviction algorithms with straggler awareness and supports cache prefetching. We implement a POCache prototype atop Hadoop 3.1 HDFS, while preserving the performance and functionalities of normal HDFS operations. Extensive experiments on both local and Amazon EC2 clusters show that in the presence of stragglers, POCache can reduce the read latency by up to 87.9% compared to vanilla HDFS.

---

## Number 13

**Name:**
LESS is More for I/O-Efficient Repairs in Erasure-Coded Storage

**Publisher:**
24th USENIX Conference on File and Storage Technologies, February 24–26, 2026 .

**Authors:**
Keyun Cheng1, Guodong Li2*, Xiaolu Li3, Sihuang Hu2, and Patrick P. C. Lee1

**Abstract:**
I/O efficiency is critical for erasure-coded repair performance
in modern distributed storage. We propose LESS, a family of
repair-friendly erasure code constructions that reduces both
the amount of data accessed and the number of I/O seeks
in single-block repairs, while ensuring balanced reductions
across blocks. LESS layers multiple extended sub-stripes
formed by widely deployed Reed-Solomon coding, and is
configurable to balance the trade-off between the amount of
data accessed and I/O seeks. Evaluation shows that LESS on
HDFS reduces both single-block repair and full-node recovery
times compared to state-of-the-art I/O-optimal erasure codes.

---
