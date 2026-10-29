# **Design Document: A Self-Adapting Storage System Inspired by Biology** 

## **1. Overview and Main Idea** 

The project applies ideas from cellular biology to explore how a software storage system can adapt to changing workloads and manage its resources autonomously. 

- Hypertrophy (Cell Growth): Cells can increase in size. In the storage system, this inspires increasing a storage instance's resources before creating additional instances. 

- Mitosis (Cell Division): A cell divides to produce two daughter cells. Similarly, a storage instance splits its assigned data range and creates a new instance to distribute the workload. 

- G₀ Phase (Resting State): Cells can enter a resting state rather than continuing to divide. The storage system uses this idea to pause before splitting, compact its data, and check whether additional capacity is still needed. 

- Cellular Homeostasis (Internal Balance): Cells regulate their internal conditions as their environment changes. This inspires the storage system to monitor resource pressure and adjust its structure in response to demand. 

- Telomeres and the Hayflick Limit (Bounded Division): Biological mechanisms limit repeated cell division. The project adapts this general idea through a counter inherited by new storage instances, limiting how many times they can split. In cells this is called a molecular clock which restricts normal cells to 50 to 70 divisions only. 

- Tissue Remodeling and Regression (Scale-In): Biological structures can change as conditions evolve. This inspires a planned feature that merges storage instances when demand decreases, reducing resource usage. This directly relates to lowering computing cost in distributed cloud computing setups. 

Overall idea: The project explores how principles of cellular growth, division, rest, and regulation can inspire a self-adapting storage system that scales up or down as workloads change. 

Note that these are biological inspirations, not exact equivalents. In particular, the inherited division counter is an engineering mechanism inspired by bounded cell division, not a literal implementation of telomere biology. 

In a traditional distributed database, a central controller often manages decisions about scaling and data placement. In this design, each storage instance monitors its own resource usage and can decide when it needs more resources or when it should split into smaller partitions. 

The system has two main parts: 

- Router (control plane): Directs client requests to the correct storage instance. 

- Storage instances (data plane): Store data, monitor their own resource usage, and manage their own partitions. 

The main idea is to reduce the need for a central controller to manage every scaling decision. 

## **2. Main Components** 

### **2.1 Router** 

The router receives requests to store or retrieve data and decides which storage instance should handle each request. 

- It uses a fast hash function to convert each key into a number between 0 and 264 −1 . 

- Each storage instance is responsible for a specific range of these numbers. 

- The router maintains a directory of the active instances and their assigned ranges. 

- It does not continuously monitor storage usage or decide when instances should split. 

- When a storage instance splits, it sends the router a notification so that the router can update its directory. 

### **2.2 Storage Instance** 

Each storage instance manages its own data and resources. 

- It uses RocksDB to store data persistently on disk. 

- It monitors its storage usage and compares it with predefined limits. 

- It can adjust its resources or split its assigned range when needed. 

## **3. Two Levels of Adaptation Inspired by Cells** 

The system uses two main strategies to respond to increasing workloads. 

### **3.1 Level One: Hypertrophy (Increasing Local Resources)** 

In biology, hypertrophy means that a cell grows larger. In this design, it means increasing the resources available to a storage instance before creating another instance. 

When it happens: A storage instance experiences a sudden increase in writes or resource usage. 

What it does: 

- Increases its in-memory write buffer from 4 MB to 16 MB. 

- Increases its background processing threads from 2 to 6. 

- Makes these adjustments without restarting the process, if supported by the relevant RocksDB settings. 

Goal: Handle temporary increases in workload without creating unnecessary storage instances. 

These resource changes are proposed design settings and should be verified in the implementation. 

### **3.2 Level Two: Mitosis (Splitting into Two Instances)** 

In biology, mitosis is the process through which one cell divides into two. Here, a storage instance splits its assigned range and creates a new storage instance. 

When it happens: The workload remains too high even after the first level of adaptation. 

What it does: 

1. Finds the midpoint of its assigned range of hash values. 

2. Divides the range into two smaller ranges. 

3. Creates a new storage instance to manage one of the ranges. 

4. Updates its own range assignment. 

5. Notifies the router so it can direct requests to the correct instance. 

Goal: Distribute the workload across additional storage instances instead of continually increasing the resources of one instance. 

### **3.3 G** ₀ **State (A Temporary Pause Before Splitting)** 

In biology, G₀ is a resting state in which a cell temporarily stops progressing through the division cycle. The storage system uses this concept as a checkpoint before splitting. 

What it does: 

1. Pauses the decision to split. 

2. Runs RocksDB compaction to reorganize stored data and remove obsolete records. 3. Checks resource usage again. 

There are two possible outcomes: 

- Pressure decreases: The instance returns to normal operation without splitting. 

- Pressure remains high: The instance proceeds with the split. 

Goal: Avoid creating new instances when a temporary workload spike or accumulated obsolete data can be handled locally. 

### **3.4 Limiting Further Splits** 

Repeated splitting could create too many small storage instances. To reduce this risk, each instance carries a counter that limits how many further divisions are allowed along its branch. 

How it works: 

1. Each instance starts with a division counter. 

2. When it splits, it decreases the counter and passes the updated value to the new instance. 

3. When the counter reaches zero, that instance can no longer split. 

It can continue serving read and write requests. 

Goal: Place a limit on repeated splitting and reduce excessive partition fragmentation. 



<!-- Start of picture text -->
COMPARATIVE MITOSIS VS. STORAGE SPLIT<br>(ra CELL MITOSIS STORAGE SYSTEM SPLIT<br>emanuie 1. BIOLOGICAL SPLIT Colurteves 1, SINGLE INSTANCE & G, PHASECHECK fea]<br>REMAINING<br>(SPLIT LIMIT, showing 5) 0<br>DIVISIONS © F oN ‘IvisioNs<br>fol — ( ' (2) fa)<br>& AE ASS —— &<br>fA 4, TELOPHASE/CYTOKINESIS rt Ge! J 4. TWO INDEPENDENT INSTANCES fA<br>é \W\. 4 DAUGHTER<br>@3) (© o seaes<br>ALF RANGES ALF RANGES<br><!-- End of picture text -->

## **4. Communication Between Storage Instances** 

### **4.1 In-Memory Communication Proxy** 

The storage instances communicate through an interface that separates their internal logic from the communication method. 

In the current prototype, an in-memory proxy uses direct references and pointers to connect instances within the same process. 

This allows the core storage and scaling logic to be tested before adding network communication. The interface is intended to make a future gRPC or socket-based implementation easier, although additional engineering will be needed for a production deployment. 

### **4.2 Retrieving Data from a Parent Instance** 

After a split, some records may still exist only in the original instance. 

The proposed design handles this as follows: 

1. A request reaches the new instance. 

2. If the requested record is not present locally, the new instance asks its parent for it. 

3. The new instance copies the record into its own storage. 

4. Once ownership has been safely transferred, the parent can remove its duplicate copy. 

This allows records to move gradually instead of requiring all data to be transferred immediately during a split. 

The current in-memory communication design is a prototype; network-based retrieval remains a separate implementation step. Safe deletion, concurrent updates, and failures during migration will also need explicit handling. 

## **5. Future Work: Merging Storage Instances** 

The next major research goal is to explore the reverse of mitosis: combining storage instances when demand decreases. 

This process is called scale-in in distributed systems. 

When it happens: Two neighboring storage partitions remain small enough that maintaining separate instances may no longer be worthwhile. 

Proposed process: 

1. One instance determines that its partition can be merged with its neighbor. 

2. It prepares its data in a format that can be transferred efficiently. 

3. The receiving instance imports the data. 

4. The system ensures that the merged data is available and consistent. 

5. The redundant instance shuts down. 

6. The router updates its directory to reflect the new partition arrangement. 

The proposed implementation uses RocksDB's SST file-writing and external-file-ingestion capabilities to transfer data efficiently. 

The example threshold of 2 MB is a design parameter to investigate, not an established optimal value. 

Goal: Reduce resource consumption when workloads decline, while preserving data correctness and availability. 

## **Project Summary: Biomimetic Autonomous Storage System** 

This project explores how principles of cellular biology can inspire a self-adapting storage system that responds to changing workloads with less reliance on centralized control. The goal is to design a system in which individual storage instances monitor their own resource usage and make decisions about when to expand, split, or eventually merge. 

The project draws inspiration from several biological concepts: 

- Hypertrophy (Cell Growth): A cell grows larger to meet its needs. Similarly, a storage instance increases its local resources to handle rising workloads before creating additional instances. 

- Mitosis (Cell Division): A cell divides into two daughter cells. In the storage system, an instance splits its assigned data range and creates a new instance to distribute the workload. 

- G₀ Phase (Resting State): Cells can temporarily stop progressing toward division. The storage system uses this idea to pause before splitting, reorganize stored data through compaction, and determine whether the increased workload still requires another instance. 

- Limited Cell Division: Biological mechanisms can limit repeated cell division. Inspired by this concept, the system passes a division counter from a parent instance to its newly created instance, limiting repeated splitting and helping prevent excessive fragmentation. 

- Cellular Homeostasis (Maintaining Balance): Cells regulate their internal conditions as their environment changes. Similarly, the storage system monitors resource pressure and adjusts its resources and structure in response to changing workloads. 

- Scale-In (Inspired by Biological Remodeling): Just as biological structures can change as conditions evolve, the proposed system would merge storage instances when demand decreases, reducing unnecessary resource consumption. 

The central research question is whether these biology-inspired mechanisms can help create a storage system that adapts to changing workloads more autonomously, efficiently, and with less centralized coordination. 

The project is currently at the prototype stage. The immediate focus is on developing the core storage architecture and progressively implementing and testing the proposed adaptation mechanisms. Future work includes evaluating system behavior under different workloads, introducing network communication between storage instances, and developing automatic scale-in. Through this process, the project aims to explore how ideas from biology can be translated into practical computing systems and whether they offer useful approaches to autonomous resource management and scalability. 

