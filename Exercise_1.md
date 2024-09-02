# Computer Systems and Networks

## Lecture 1 Exercises. Done by Joel Okore and Fedorov Alexey. Group: [M24-SNE-01]

### Exersice 1
```
Scenario: You are designing a distributed file system where files are stored across
multiple servers. The goal is for users to be able to access and manipulate their
files without worrying about where the files are stored, how many copies exist, or
what happens if a server fails.

Tasks:

Identify Transparency Requirements:
1. List the types of transparency that are relevant for your distributed file system.
2. Justify why each type is important for the users' experience.

Design a Solution:
1. Propose a high-level design that addresses at least two types of transparency identified.
2. Explain how your design hides the complexities of the distributed system from the users.

Consider Failure Handling:
1. Describe how your system can handle a situation where one of the file servers goes
down while maintaining transparency.
```

#### Identify Transparency Requirements

We decided that for distributed filesystem the following transparencies are most relevant:
it  
- **Access** - There should be no difference between local and remote files access methods.
- **Location** - Users should be able to access shared files data without interference between each other.
- **Migration** - User dont care about where his file stored. If we need to migrate file, User experience must not be affected.
- **Replication** - Replication of data must not concern user. It means that operations of uploading, downloading data to/from replica must not make influence of how system works.
- **Concurrency** - Users should be able edit files together.
- **Failure** - There are many reasons for system failure. The worst thing is that part of them depends on third parties. Every time system fails, it must not affect the user. If it is impossible, system downtime should be short as it can be.
- **Scaling** - System should be adaptable to growing load. All elements must be scalable.

#### Design a Solution

##### Replication

Let's task about replication transparancy design. For example we have the following high level architecture:

![image](https://github.com/user-attachments/assets/b3695203-ef63-4a1c-8596-7ba6afb60aca)

Client goes to service to access file, Service takes file from Repository (Database, S3, Filesystem) and returns it to user.

Let's add a some kind of replication for Repository. For example, we use Postresql, it have builtin streaming replication mechanism. 
It will be simple Master-Slave replication.

![image](https://github.com/user-attachments/assets/19dfcb34-57c2-4296-8ffb-edec69c11947)

Postgres Master - is Read/Write, Postgres Slave - is ReadOnly. Master always replicating it's data to slave.

This is simple replication. We can use Postgres Slave as backup database or for readonly analytics, but not as a failover solution, because if Master in downtime, we will promote Slave manually.

To make replication transparency work, we need use more reliable solutions. The following solutions are our choice:

- Master-Master replications. Old but Gold. Expensive, but reliable solution.
- RAFT. Consensus algorithm. When old master is unreachable,new master will be chosen randomly among slaves.

Let's design one of solutions above. For example Master-Master:

![image](https://github.com/user-attachments/assets/2bea32fe-db93-4cbe-ad95-1a9d8e25a3d8)


Unfortunately there no built in Master-Master solution in Postgresql, we need to use third party software, "Patroni" for example.

##### Failure

In designing a distributed file system (DFS) that address failure transparency and fault tolerance, it is crucial to integrates multiple strategies to ensure the system's reliability and availability, even in cases of hardware or software failures. Component redundancy ensures, by storing multiple copies of each piece of data, that even if one node (server) fails, the data remains accessible from other nodes. Automatic failover mechanisms can be implemented to detect server or node failures and automatically redirect operations to healthy servers, ensuring that the system continues to function without interruption.   

The system's ability to recover from failures can be further improved by Recovery Mechanisms. These mechanisms continuously monitor the system for data loss or inconsistency and automatically trigger processes to re-replicate lost data and synchronize failed nodes.    

To avoid single points of failure, Load Balancing can be incorporated. By distributing control across multiple nodes using distributed hash tables (DHT), the system reduces the risk of failure. Client-Side Resilience is also be achieved by implementing caching on the client side, the system ensures that clients can continue to operate even during temporary failures.  

#### Consider Failure Handling

We described failover solutions in previous task. Hope it is enough.

### Exercise 2
```
Which kind of transparencies are supported by the DNS?
```
 
- **Replication** - As a minimum dns records are replicated on root servers.
- **Concurrency** - DNS implements concurrent access for the same records.
- **Failure** - For failure handling the  architecture of DNS splited by layers.
![image](https://github.com/user-attachments/assets/e7ae2ef9-d5f2-49b9-aa40-fd6dfc3d72ee)
- **Scaling** - DNS architecture accepts scaling.
- **Performance** - The configuration of the DNS not be apparent to the user in terms of performance. 

