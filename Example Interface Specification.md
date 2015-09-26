Example interface specification Hobbit - Gollum
===============================================
This interface specification is a free form, human readable text document. It is under version control.
The document fully describes semantics and syntax of an interface between two actors.

**Actors**  
Client : PC application 'Gollum', C# TCP client  
Service: PLC application 'Hobbit', Siemens S7, TCP port 9100.

**Message exchange**  
Messages are transfered over a TCP/IP connection.  
The 'Hobbit' hostname is configured on 'Gollum'.  

'Hobbit' accepts connections from several 'Gollum' instances.
But only one of these instances is allowed to actively execute commands.

When a client or a service has not received a message for 10 seconds, it closes the connection locally.

The service sends responses to requests from the client.
The service also may send asynchronous notifications to the client.

The message serialization format is [BMS1](https://github.com/steforster/bms1-binary-message-stream-format).


History
-------

Version |    Date    | Name | Modifications
:------:|:----------:|:----:|:-------------
   1    | 2015-01-01 | SFo  | Created for project Xyz
   2    | 2015-09-01 | SFo  | Added feature Z for project Uvw

Please note: Interface version numbers are for information and logging.
Client and service are designed to be back- and forward compatible regarding this interface definition.
This means, that a new actor must behave as the old version when receiving a message from an old partner.
The old partner will skip unknown parts of the newer message version.
Special care has to be taken when extending enumeration definitions:
Never send a new enumeration definition to an old partner.


Overview
--------

### Included interfaces
The two actors also support messages from base interface definitions.
All interfaces share the same TCP connection.
An offset is added to message- and block IDs of included interfaces.
The IDs are mapped as follows:
* 1001...1999: UserManagement interface
* 2001...2999: Logging interface
* 3001...3999: DriveConfiguration interface

### Message type IDs
The 'Hobbit - Gollum' interface supports the following message types:
* 1: Idle
* 2: Identification

### Inner block type IDs
The messages of the 'Hobbit - Gollum' interface contain the following block types:
* 100: VersionBase
  * 101: VersionDotNet
  * 102: VersionPLC

### Basic data types
Not all datatypes of the BMS1 specification are supported by the PLC 'Hobbit'.
The recommended datatypes are:  
	Bool, Byte, Int16, Int32, Char[]

Due to restrictions on the 'Hobbit' PLC, strings are transferred as UTF8 encoded byte arrays.
The definition Char[20] means that the PLC will limit the string to 20 bytes when reading the message.


Message definition
------------------

### Message type 1: Idle
This is an empty message without any data member.
The client sends it, when it has not sent another message for 3 seconds.
When a service receives an idle message, it replys an idle message too.

	
### Message type 2: Identification
When a service receives an identification message from a client, it replys the same message type with the service identification.

	ApplicationName: Char[30];
	InterfaceVersion: Int16;
	ApplicationVersion: Block 'VersionBase';
		Here we demonstrate how to define inherited block types in messages:
		Block 'VersionBase' may be either a 'VersionDotNet' or a 'VersionPLC' block.
 

#### Block type 100: VersionBase
VersionBase contains no members, it is the base of type VersionDotNet and VersionPLC.


#### Block type 101: VersionDotNet : VersionBase
	Major: Int16;
	Minor: Int16;
	Revision: Int16;
	Build: Int16;


#### Block type 102: VersionPLC : VersionBase
	Version: Char[30];

	**Version 2**
	CpuType: Enum CpuType;
		Here we demonstrate how to define an extended message member in version 2 of the message.
		Starting with version 2, the VersionPLC-block includes the CpuType enumeration.
	    The added member will simply be skipped by an old receiver.


Enumeration definition
----------------------

### Enum CpuType
	Unknown = 0,	
	IntelAtom = 1,
	ArmCortexA5 = 2,
	ArmCortexA9 = 3

