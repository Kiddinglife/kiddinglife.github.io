---
layout: post
title:  "Geco Engine 1: Server Components"
date:   2016-05-02 19:12:03
categories: game-server
tags:  gecoengine network 
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

aoi priority list of e0:  
 - e1 10m position and direction update event list { jump, weapon change ...}
 - e2 20m position and direction update event list { jump, weapon change ...}
 - e3 30m position and direction update event list { jump, weapon change ...}
 
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

##### 5.1.5.1.1 Level of Detail
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

###### 5.1.5.1.1.1 Non-state-changing events
In the description of each entity type, there is a description of all 
messages (non-state-changing events) it can generate, along with the 
priority associated with each one.

When an avatar adds information about an entity, it only adds messages 
with a priority greater than the avatar's current interest in the entity. 
Therefore, messages will be ignored depending on the distance between 
the avatar and the entity.

For example, there might be a type of chat heard only within 50 metres, 
or a jump seen only within 100 metres.

###### 5.1.5.1.1.2 State-changing events
GECOEngine cannot ignore state-changing events that occur out of range, 
because if the entity subsequently does get close enough, the client will 
have an incorrect copy of the entity's state.

For example, the client may be interested in the type of weapon that an 
entity is carrying, but only when it is within 100 metres. If the entity 
changes its weapon outside this range, then this information is not sent 
to the client as yet. BigWorld will only send it if the entity comes within 
range.

Unlike non-state-changing events, in which the priority level can have any 
value, state-changing events have a discrete or non-contiguous number of state LODs
specified by the developer. 

These can be thought of as rings around the client. Each property of an 
entity's type can be associated with one of these rings. For an example:
```xml
    <seatType>
      <Type>         INT8          </Type>
      <Flags>        OTHER_CLIENT  </Flags>
      <Default>      -1            </Default>  <!-- See Seat.py -->
      <DetailLevel>  NEAR          </DetailLevel>
    </seatType>
```
 
For example, supposed an avatar A is surrounded by four entities of the 
same type, with LOD levels at 20m, 100m, and 500m. It will process 
each entity according to its distance, as described in the table below: 

| Entity   	|      Distance   |   	   20m|100m |500m |               
|----------	|:-------------:	    |------:	|----------	|:-------------:	    
| E1 	|  15m 	  | y	    |  y	|   y  | 
| E2 	|  90m    |   n	| y 	|  y 	| 	
| E3 	|  400m  |    n 	| n 	| y 	|    
| E4 	|  1000m|    n 	| n 	| n 	|    

When an entity with an associated witness comes within an LOD ring of 
another entity, any state associated with the ring that has changed since
they were last this close is sent to the client. 

uses timestamps to achieve to prevent very often updates. A timestamp is
stored with each property of each entity. This timestamp indicates when 
that property was last changed. Also, for each entity in a witness' AoI, 
a timestamp corresponding to when that entity last left that level is kept 
for each LOD ring.

By itself, this does not allow us to throttle this data. To achieve this, 
we only need to apply a multiplying factor to the interested level of the 
avatar when it is calculated for other entities. Multiplying by a factor 
less than one has the effect of reducing the amount of data sent. 
As a result, the bigger the ring is, the more data to be sent. the smaller the 
ring is, the less data to be sent.

### 5.1.6. Scripting and Entities
The following sub-sections describe how to use scripting for entities.

#### 5.1.6.1. Entity Classes
An entity class describes a type of entity. The list below describes its 
component parts:
 - An XML definition file    <res>/scripts/entity_defs/<entity>.def  
 - A Python cell script       <res>/scripts/cell/<entity>.py
 - A Python base script     <res>/scripts/base/<entity>.py
 - A Python client script    <res>/scripts/client/<entity>.py

The XML definition file can be considered the interface between the Cell, 
Base and Client scripts. It defines all available public methods, and all 
properties of an entity. In addition, it defines global characteristics of 
the class, such as:
 - Whether the entity requires volatile position updates.
 - How the entity is instantiated on the client.
 - The base class of the entity, if any.
 - The interfaces implemented by the entity, if any.

#### 5.1.6.2. Example Entity Definition File
The following is an example entity definition file <res>/scripts/entity_defs/Seat.def.

```xml
<root>
  <Properties>
    <seatType>
      <Type>         INT8          </Type>
      <Flags>        OTHER_CLIENT  </Flags>
      <Default>      -1            </Default>  <!-- See Seat.py -->
      <DetailLevel>  NEAR          </DetailLevel>
    </seatType>
    <ownerID>
      <Type>         INT32         </Type>
      <Flags>        OTHER_CLIENT  </Flags>
      <Default>      0             </Default>
    </ownerID>
    <channel>
      <Type>         INT32         </Type>
      <Flags>        PRIVATE       </Flags>
    </channel>
  </Properties>

  <ClientMethods>
    <clientMethod1>
      <Arg>            STRING  </Arg>  <!-- msg -->
      <DetailDistance> 30      </DetailDistance>
    </clientMethod1>
  </ClientMethods>

  <CellMethods>
    <sitDownRequest>
      <Exposed/>
      <Arg>      OBJECT_ID  </Arg>  <!-- WHO -->
    </sitDownRequest>

    <getUpRequest>
      <Exposed/>
      <Arg>      OBJECT_ID  </Arg>  <!-- WHO -->
    </getUpRequest>

    <ownerNotSeated>
    </ownerNotSeated>

    <tableChat>
      <Arg>      STRING    </Arg>  <!-- msg -->
    </tableChat>
  </CellMethods>
  
  <LODLevels>
    <level>  20   <hyst>   4  </hyst> <label> NEAR   </label>  </level>
    <level>  100  <hyst>  10  </hyst> <label> MEDIUM </label>  </level>
    <level>  250  <hyst>  20  </hyst> <label> FAR    </label>  </level>
  </LODLevels>

</root>
```

#### 5.1.6.3. Properties
All properties are defined in the XML file, 
 - along with their data type, 
 - an optional default value, 
 - flags to indicate how they should be replicated, 
 - and an optional tag DetailLevel for LOD processing.
 
All cell entity properties need to be defined, even if they are private to 
the class. This is because CellApps need to offload entities to other 
CellApps, and having the data types of all properties defined allows 
the data to be transported as efficiently as possible.

__Note:__ For security and efficiency reasons, when a client modifies a 
shared property, the change is not propagated to the server. All 
communication from the client to the server must be done using method 
calls. 

This is because only when original data  is changed or updated, 
propagation of the new value on copy data can be done. 
And the only way a client can change the original data is via rpc.

When a Python script modifies the value of a property, the server does
the work of propagating this property change to the correct places.

In other words, when a property changed from the entity's own client via 
rpc or command msg, the server uses:
 - the flags OWN_CLIENT and ALL_CLIENTS
to determine if an update should be sent to the entity's own client,    
 - the flags OTHER_CLIENTS and ALL_CLIENTS to determine if an update 
should be sent to the other clients.

Flag is consisted of SourceProcess + Accessibility +PropagationProcess.

SourceProcess must not be same to PropagationProcess. 

SourceProcess is among CELL, BASE or CLIENT, 

Accessibility is among of PUBLIC, PRIVATE OR PROTECTED. 

PropagationProcess is among of ALL_CLIENT, OWN_CLIENT, 
OTHER_CLIENT, BASE or NONE if only stored in SourceProcess no 
propagation.

IE., ALL_CLIENT can be treated as CELL__PUBLIC__ALL_CLIENT.

The following flags are used in the XML definition to determine how each 
property should be replicated.
  - ALL_CLIENT    
    - Implies the CELL_PUBLIC flag  
    - Only relevant for entities with an associated client, 
    - IE., a player entity Properties that are visible to all nearby clients, 
   including the owner. 
    - Corresponds to setting both the OWN_CLIENT and OTHER_CLIENT flags.
 - BASE  
   - Properties that are used on bases.
 - CELL_PRIVATE  
   - Properties that contain state information that is internal to an entity.  
   - They are available on to the owner, and only on the cell. 
   - They will not be ghosted, and as such, entities on other cells will not be 
   able to see them.
 - CELL_PUBLIC
   - Properties that are visible to other entities on the server. 
   - They will be ghosted, and any other nearby entity will be able to read 
   them, whether they are in the same cell or not (as long as they are 
   within the AoI distance).

__NOTE:__ Property definitions can also include the tag DetailLevel. This is used for 
data LOD processing. For details on LOD processing, see the document 
_Server Programming Guide Proxies and Players → Entity Control_.

#### 5.1.6.4. Built-in Properties
In addition to the properties defined in the XML file, cell entity scripts also 
have access to several built-in properties provided by the cell scripting 
environment. These include:
 - id (Read-only) Integer ID of the entity.
 - spaceID (Read-only) ID of the space that the entity is in.
 - vehicle (Read-only) Some other entity (or None) that the entity is riding upon.
 - position (Read/write) Position of the entity, as a tuple of 3 floats.
 - direction (Read/write) Direction of the entity, as a tuple of roll, pitch, and yaw.
 - isOnGround (Read/write) Integer flag, containing 1 if the entity is on the ground, or 0 otherwise.
 
It is possible to move an entity by changing its position property. However,
for continuous movement, the method moveToPoint is recommended.
The spaceID and vehicle properties are influenced by the teleport and 
boardVehicle methods, respectively.

#### 5.1.6.5. Methods
There are different sections for client, cell, and base methods, and each 
method contains its name and its list of arguments.

Cell and base methods can also have an optional <Exposed/> tag. This 
indicates whether the client can call this method. Exposed methods on the
cell have an implicit argument, source, which is the ID of the entity
associated with the client that called the method.

Method definitions can also have a <DetailDistance> attribute which is 
used in relation to Level of Detail filtering.

#### 5.1.6.6. Calling Client methods
A client has references to all entities that are within its AoI. 
A cell component of an entity can call methods on other client entities 
(including its own) that exist within this AoI.

Four properties are available to the cell script, to facilitate calling client 
methods. These are:
 - ownClient (or client)
 - otherClients
 - allClients
 - clientEntity
 
A method called on one of these objects will be delivered to the indicated 
clients.

_For example, to call the chat method on the client script for this object, 
for all nearby clients:_ `self.allClients.chat( "hi!" )`.

#### 5.1.6.7. Cell Built-in Methods
The cell script has access to numerous built-in script methods, 
some of which are described in the list below:
 - _destroy_ — Destroys the entity.
 - _addTimer_ — Adds an asynchronous timer callback.
 - _moveToPoint_ — Moves the entity in a straight line towards a point.
 - _moveToEntity_ — Moves the entity towards another entity.
 - _navigate_ — Moves the entity towards a point, avoiding obstacles.
 - _cancel_ — Cancels a controller (timers, movement, etc...).
 - _entitiesInRange_ — Finds entities within a given distance of the entity.
 
#### 5.1.6.8. Controllers
A controller is a C++ object associated with a cell entity. It normally 
performs a task that would be inefficient or difficult to do from script. 
It also generally has state information, which must be sent across the 
network if the entity is offloaded to another cell.

Examples of currently implemented controllers are:
 - TimerController — Provides asynchronous timed callbacks.
 - MoveToPointController — Moves the entity to a position, at a given velocity.
 
Since controllers normally perform operations asynchronously, they notify 
the script object of completion by calling a named script callback. 
For example, the TimerController always calls the onTimer event handler, 
and the MoveToPointController always calls the onMove event handler.

5.1.6.10. Example Script File
Cell entity script — <res>/scripts/cell/Seat.py
The following is an example cell entity script file for a Seat object:
```python
"This module implements the Seat entity."
import BigWorld
import Avatar
# -----------------------------------------------------------------------
# Section: class Seat
# -----------------------------------------------------------------------
class Seat( BigWorld.Entity ):
  "A Seat entity."
  def __init__( self ):
    BigWorld.Entity.__init__( self )

  def sitDownRequest( self, source, entityID ):
    if self.ownerID == 0 and source == entityID:
      self.ownerID = entityID
      BigWorld.entities[ self.ownerID ].enterMode(
        Avatar.Avatar.MODE_SEATED, self.id, 0 )
      if self.channel:
        try:
          channel = BigWorld.entities[ self.channel ]
          channel.register( self.ownerID )
        except:
          pass

  def getUpRequest( self, source, entityID ):
    if self.ownerID == entityID and source == entityID:
      if self.channel:
        try:
          channel = BigWorld.entities[ self.channel ]
          channel.deregister( entityID )
        except:
          pass
      BigWorld.entities[ self.ownerID ].cancelMode()
      # The Avatar's cancelMode() method will in-turn call this
      # Seat's ownerNotSeated() method to release itself.

  def ownerNotSeated( self ):
    self.ownerID = 0

  def tableChat( self, msg ):
    if self.channel:
      channel = BigWorld.entities[ self.channel ]
      assert( channel )
      assert( self.ownerID )
      channel.tellOthers( self.ownerID, "[local] " + msg )
```

### 5.1.7. Directed Messages
Not all events are propagated via the event history. If an event is destined 
for a specific player, or a small number of players, then it can be delivered
almost immediately in its next packet.

### 5.1.8. Forwarding From Ghosts
Each ghost knows its real cell, so that it can forward messages to its 
real version to be processed.

Most messages are of the _pull_ type, as in when another player jumps — 
it is up to the avatar (via the priority queue) to decide if and when to send 
this to its client.

Minority of the messages is of the push variety, as in when player A 
wants to shake hands with player B. To do this, client A sends a handshake
request to its cell, which in turns  makes a request to avatar B on the same cell. 

Avatar B may be a ghost, in which case it forwards the request to its 
real representation. This communication happens automatically and 
transparently.

### 5.1.9. Offloading Entities
Approximately once a second each cell checks whether to offload any 
entities to other cells. Each entity is considered against the boundary of 
the current cell. 

The boundary is artificially increased (by about 10 metres) 
to avoid hysteresis. If an entity is found to be outside the boundary of the 
current cell, it is offloaded to the most appropriate one.

When an entity is offloaded, the real entity object is transformed into a 
ghost entity, and vice-versa when an entity is loaded.

### 5.1.10. Adding and Removing Cells
When a cell is added to a large multi-cellular space during runtime, it is 
done by gradually increasing its area, in order to avoid a large spike in 
entities trying to change cells. Similarly for removing a cell, its area slowly 
decreases until all entities are gone.

### 5.1.11. Load Balancing
Cell boundaries are moved in order to adjust the proportion of the server
load that an individual cell is responsible for.
 
The basic (simplified) idea is that if a cell is overloaded in comparison to 
other cells, then its area is reduced. Conversely, if it is underloaded,
 then its area is increased.

### 5.1.12. Physics
Because of factors such as latency inherent in traversing the Internet, 
large number of desired players, and bandwidth limits, it is difficult (if not 
impossible) to achieve a convincing game where full collision handling is 
attempted.

For more details, see the document _Server Programming Guide's section 
Proxies and Players → Entity Control._

### 5.1.13. Navigation System
The key features of the navigation system are:
 - Navigation of both indoor and outdoor chunks.
 - Seamless transition between indoor and outdoor regions.
 - Dynamic loading of navpoly graphs.
 - Paths caching, for efficiency.

For details, see the document _Server Programming Guide, 
section Entities and the Universe → Navigation System._

### 5.1.14. Range Triggers and Range Queries
There is often the need to trigger an event when an entity (of a specific 
type) gets close enough to another. These are called Range Triggers.

There is also the need to find all entities that are in a given area. This is 
called a Range Query.

To perform a range query, the software searches just one of these lists to 
the distance of the query range, and then selects the entities geometrically 
in range (can be achieved by set intersection that must <z and <y).


To perform a range trigger, the entity that is interested in a particular 
range inserts high and low triggers in both the X and Z lists. When the 
entity moves, the lists and the triggers are updated. When other entities
 move, the lists are updated. If an entity crosses a trigger, or a trigger 
 crosses an entity, then the software checks the other dimension to test 
 whether it is actually a trigger event.
 
 For the majority of situations this algorithm has been found to be very effective.

### 5.1.15. Fault Tolerance
The CellApp is designed to be fault tolerant. A CellApp process can stop 
suddenly, or its machine can become disconnected from the network, 
with little noticeable impact on the server.

## 5.2. CellAppMgr
The CellAppMgr is responsible for:
 - Maintaining the correct adjacencies between cells.
 - Adding new avatars to the correct cell.
 - Acting as a global entity ID broker.

### 5.2.1. CellApp Registration
When a new CellApp is started, it finds the location of the CellAppMgr by 
broadcasting a message to all BWMachined daemons.

If a BWMachined process has a CellAppMgr on its machine, then its 
address is returned to the CellApp. Once the CellApp has the location 
of the CellAppMgr, the CellApp registers itself with it. Next time the 
CellAppMgr needs to create more cells, the new CellApp will be considered 
as a potential host for it.

Similarly, when the CellApp stops, it deregisters itself with the CellAppMgr, 
after having all its cells gradually removed.






 

 



 

