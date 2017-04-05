---
layout: post
title:  "Geco Engine 1: Server Components"
date:   2016-05-02 19:12:03
categories: Geco Engine
tags:  game-back-end server network 
---

* content
{:toc}

## 5.1 CellApp
### 5.1.1 Cell Application and Cells
A Cell Application, or CellApp, is the actual process or executable running 
on a CellApp machine.

A CellApp manages various cells, or instances of the Cell class. A cell is a 
geometric part of a large space, or the entirety of a small space.

BigWorld divides large spaces into multiple cells in order to share the load 
evenly. It typically allocates one large cell to each CellApp. When handling 
smaller spaces, such as dynamically created mission ones, BigWorld 
typically allocates several cells to each CellApp.

In BigWorld, the CellApp machines are intended to run only BWMachined 
and one CellApp per CPU, and nothing else.

### 5.1.2 Entities
CellApps are concerned with a generic type, the entity. 
Each CellApp maintains a list of the real and ghost entities within its reach.
The CellApp also maintains a list of any real entities that need to be updated 
periodically (e.g., 10 times per second), and those that maintain an AoI.

Each entity understands the messages that are associated with its type. 
The CellApp delivers entity messages to the appropriate entity, and it is up
to it to interpret them. The way this is implemented is by having every 
entity include a pointer to an entity type object. This object has information 
related to the entity's type, such as the messages that the type can handle
and the script associated with it.

Every entity type has it own script file. Scripts execute in the context of 
a particular entity (i.e., they have a this or self), and have access to the 
data that other entities choose to make available (server and client data), 
as well as the basic members of every entity (id, position, …).

Scripts are responsible for handling messages sent to an entity.
When any server or client data is changed, the changes are automatically 
forwarded to interested entities.

### 5.1.3. Real and Ghost Entities
In order to efficiently update its client, each avatar keeps a list of entities 
in its AoI, which is typically a circle with a radius of 500m. One issue that 
arises is that not all entities might be controlled by the same cell.

However, it would be very inefficient to have each entity individually 
determine which entities on adjacent cells should be ghosted. The solution
is to have cells generically manage ghost entities. If an entity is within the 
ghost distance of a cell boundary, the cell creates a ghost of the entity on
that nearby cell.

The term real entity is introduced to distinguish between the master 
representations and the (imported) ghosts. The figure below shows all 
entities maintained by cell 1:

Once a second, a cell goes through its real entities to check if it should 
add or delete ghosts for them on neighbouring cells. It sends messages 
to the neighbour to add and remove the ghosts.

### 5.1.4. Transitioning Between Spaces
The simplest method to transition between different spaces is simply to 
'pop', i.e., to remove the entity from the old space and recreate it in the 
new one. This is simple on both client and server.

### 5.1.5. Witness Priority List
Whenever an entity has a client attached to it, a sub-object known as a 
witness is associated to the entity, in order to maintain the list of entities in
its AoI. The witness builds update packets to send to its client, and it 
must send the most important information as a priority. 

The client requires the position and other information of other entities that
are closest to it to be the most accurate and current. Closer entities should
be updated more frequently. To achieve this, the witness keeps the list of 
entities in its AoI as a priority queue.

To construct a packet to be sent to the client, relevant information about 
the entities on the top of the list is added to the packet until the desired 
size is reached. The information can include position and direction, as well
as events that the entity has processed since the last update. For more 
details, see Event History.

Closer entities receive more frequent updates, and get a greater share of 
the available bandwidth.

BigWorld throttles downstream bandwidth according to both the size of 
update packets, and their frequency. These parameters are specified in 
the configuration file <res>/server/bw.xml.

#### 5.1.5.1. Event History
Each entity keeps a history of its recent events. The history includes both 
events that change its state (such as weapon change), as well as actions
that do not (such as jumping and shooting). Frequent actions, such as 
movement, which affect volatile data, are not kept in the history list.

When an entity consumes its priority queue to construct an update packet,
it searches through new events of the top entities (i.e., the ones closes to it), 
and adds all messages it is interested in. This requires keeping a timestamp
(actually a sequence number) for the last update of each entity in the 
priority queue.

The history of these types of events is fairly short (around 60 seconds) 
since we expect that each entity in the priority queue will be considered 
(roughly) once every 30 seconds in the worst case scenario.

#### 5.1.5.2 Level of Detail
The priority queue handles the data throttling for continuous information. 
For events, we still need to handle the case where there are too many 
events filling up available downstream bandwidth to the client.

When there are many entities in an entity's AoI, information about each 
one is sent less often, but the total number of events remains the same.

To address this, BigWorld uses a Level of Detail (LOD) system for handling
event-based changes. Its principle is that the closer an entity is to the avatar, 
the more detail it should send to its client. 

For example, when a player initially comes within the avatar's AoI, 
the avatar does not need to know all details about the other player. 
The visible inventory may only be required when within, say, 100 metres. 
Finer details, such as a badge on clothing, might only be needed within 
20 metres.

##### 5.1.5.2.1 Non-state-changing events
In the description of each entity type, there is a description of all 
messages (non-state-changing events) it can generate, along with the 
priority associated with each one.

When an avatar adds information about an entity, it only adds messages 
with a priority greater than the avatar's current interest in the entity. 
Therefore, messages will be ignored depending on the distance between 
the avatar and the entity.

For example, there might be a type of chat heard only within 50 metres, 
or a jump seen only within 100 metres.

##### 5.1.5.2.2 State-changing events





 

 



 

