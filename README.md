BMS1: Binary Message Stream, Version 1
======================================

**BMS1** is yet another data serialization format. The goals of its design are:

* Binary data for fast and lossless serialization. Minimized execution time of serialization.
* Compact implementation on many devices: Machine control (PLC), embedded systems, mobile devices, PCs, ...
* Platform independent: Different programming environments and different CPUs are able to understand the messages.
* Extensible and version tolerant:
  Old implementations of the BMS serializer and old application versions know how to skip new/unknown parts of a message.
  A device with new BMS-serializer or new BMS-message version may be plugged into a network of old devices and the system behaves as before.
  To fulfill this requirement, the application programmer must follow some rules regarding message structure.
  See e.g. [Versioning/Compatibility](http://diwakergupta.github.io/thrift-missing-guide/).
* Partially implementable: An implementation must not support all attributes and data types.
  The partial implementation is able to correctly read all known data of a structure. Then it is able to skip all unknown,
  additional or later defined data.
* Potential to replace Json or XML serializers.


**BMS1 Implementations**

* [Remact.Net.Bms1Serializer](https://github.com/steforster/Remact.Net.Bms1Serializer), C# for Windows and Mono platforms


**Other binary data serialization formats**

* [Wikipedia](https://en.wikipedia.org/wiki/Comparison_of_data_serialization_formats)
* [BSON](https://en.wikipedia.org/wiki/BSON)
* [MessagePack](https://en.wikipedia.org/wiki/MessagePack)
* [Protocol Buffers](https://en.wikipedia.org/wiki/Protocol_Buffers)
* [AMQP](https://paolopatierno.wordpress.com/2015/08/30/amqp-isnt-so-scary-if-you-know-how-to-start/)

To my knowledge no other serialization format (maybe except AMQP) is easily implementable and would perform efficient on PLCs.


### BMS1 message frame

The BMS1 message is framed by MessageStart and MessageEnd tags.
The MessageStart tag allows to check whether the receiver has to reverse the byte order of
integer values e.g from [little to big endian](https://en.wikipedia.org/wiki/Endianness).

A message contains one block. The block is framed by BlockStart and BlockEnd tags.
The block contains an arbitrary count of values or other blocks.
Values and blocks are optionally preceded by attributes.

Values and blocks are identified by their position in the message.
Before the end of a block, additional values or blocks may be appended by a new sender.
This data is skipped by an old receiver.

All elements of a BMS1 message are preceded by a tag-byte.
The tag-byte defines the type of the element and the length of binary data that follows the tag-byte.

Example message structure:

    MessageStart
       Attribute
       BlockStart
           Attribute
           Attribute
           Value
           ...
           Value
           ...
           BlockStart
               Value
               Value
           BlockEnd
       BlockEnd
    MessageEnd


In Extended Backus-Naur Form [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_Form),
the BMS1 frame is defined as follows:

    <Message>      ::= MessageStart [{<Attribute>}] <Block> MessageEnd

    <Block>        ::= BlockStart {<BlockOrValue>} BlockEnd

    <BlockOrValue> ::= [{<Attribute>}] <Block> | <Value>

    <Value>        ::= <ValueType> <Length> [Data]

    <Attribute>    ::= <AttributeType> <Length> [Data]

    <AttributeType>::= Name | Type | BaseType | CollectionSize | NameValue | Namespace

    <ValueType>    ::= Null | Bool | Byte | UInt16 | UInt32 | Int16 | Int32 | Int64
                     | Enumeration | Bitset | Decimal | Float | Double
                     | Date | Time | DateTime | Char | String

    <Length>       ::= 0 | 1 | 2 | 4 | 8 | 16 | Utf8ZeroTerminated
                     | additional 1 byte array length field
                     | additional 4 byte array length field



### Length specification of BMS1 tag-bytes

A tag is represented in one byte. One byte can hold decimal values from 0 to 255.
The least significant decimal digit is used as a length specifier for the data that follows the tag.

    Tag ID    Length of data that follows the tag byte
    ------    ----------------------------------------
    xx0       No data follows, all contained data is zero (0), 
              is empty, is obsolete or is undefined for some data types.
    xx1       1 byte   (  8 bit) data follows
    xx2       2 bytes  ( 16 bit) data follows
    xx3       yet undefined, invalid length specifier
    xx4       4 bytes  ( 32 bit) data follows
    xx5       An arbitrary amount of data follows. It is an UTF8 encoded,
              single zero terminated character string (the last byte is '0').
    xx6       1 byte follows that defines the byte-length of additional data (0...255 bytes)
    xx7       4 byte follow  that define  the byte-length of additional data (0...4'294'967'295 bytes)
              The receiving side may restrict data length to a maximum number of bytes.
    xx8       8 bytes  ( 64 bit) data follows
    xx9       16 bytes (128 bit) data follows


Note A: The tags 00x, 23x, 24x and 25x do not have length specifiers according to the above rules.

Note B: The tags 00x are single byte tags, there is no data to be skipped after an unknown tag in this range is received.

Note C: The tags 23x, 24x and 25x have individually defined additional data length between 0 and 4 bytes.
        The same length applies for the alternate tag sets 1...3 (tag 241...243).



### BMS1 value tags

The following single byte value-tags also define a data value:

    Tag ID    Description
    ------    -----------

    000       Not allowed, invalid tag
    001...006 yet undefined value tags without additional data

    007       Null value, null block or null array,
              also used for obsolete blocks/values (unused elements of newer message definition)

    008       Bool value = false
    009       Bool value = true


The following value tags define data type and length of the next bytes in the data stream (according to length specification).
Wherever possible, the length is defined by the data value not by the data type.
For example, the 64 bit integer value '-100' is stored in 8 memory bytes but is streamed in 2 bytes
(Tag-byte '071' + Value-byte '-100').

    Tag ID  Description
    ------  -----------

    010     Unsigned byte,    8 bit,         [0]
    011     Unsigned byte,    8 bit, range = [0...255]
    016     Unsigned byte,    short array or ASCII character string when attribute 240 is present.
    017     Unsigned byte,    long  array or ASCII character string when attribute 240 is present.

    020     Unsigned UInt16 bit,             [0]
    021     Unsigned UInt16 bit,     range = [0...255]
    022     Unsigned UInt16 bit,     range = [0...65'535]
    026     Unsigned UInt16, short array or Unicode wide character string
                             when attribute 240 is present, (size = byte length/2).
    027     Unsigned UInt16, long  array or Unicode wide character string when attribute 240 is present.

    030     Signed Int16 bit,                [0]
    031     Signed Int16 bit,        range = [-128...+127]
    032     Signed Int16 bit,        range = [-32'768...+32'767]
    036     Signed Int16, short array, (size = byte length/2).
    037     Signed Int16, long  array.

    040     Unsigned UInt32 bit,             [0]
    041     Unsigned UInt32 bit,     range = [0...255]
    042     Unsigned UInt32 bit,     range = [0...65'535]
    044     Unsigned UInt32 bit,     range = [0...4'294'967'295]
    046     Unsigned UInt32 short array, (size = byte length/4).
    047     Unsigned UInt32 long  array.

    050     Signed Int32 bit,                [0]
    051     Signed Int32 bit,        range = [-128...+127]
    052     Signed Int32 bit,        range = [-32'768...+32'767]
    054     Signed Int32 bit,        range = [-2'147'483'648...2'147'483'647]
    056     Signed Int32, short array, (size = byte length/4).
    057     Signed Int32, long  array.

    060     Signed Int64 bit,                [0]
    061     Signed Int64 bit,        range = [-128...+127]
    062     Signed Int64 bit,        range = [-32'768...+32'767]
    064     Signed Int64 bit,        range = [-2'147'483'648...2'147'483'647]
    066     Signed Int64, short array, (size = byte length/8).
    067     Signed Int64, long  array.
    068     Signed Int64 bit,        range = [-/+ full 64 bit range] also used for GUID.

    070     Enumeration value                [0]
    071     Enumeration  8 bit       range = [-128...+127]
    072     Enumeration 16 bit       range = [-32'768...+32'767]
    074     Enumeration 32 bit       range = [-2'147'483'648...2'147'483'647]
    075     Enumeration value in string representation
    076     Enumeration 16 bit short array, (size = byte length/2).
    077     Enumeration 16 bit long  array, (size = byte length/2).
    078     Enumeration 64 bit       range = [-/+ full 64 bit range]

    080     Bitset, all bits = 0
    081     Bitset            8 bit
    082     Bitset           16 bit
    084     Bitset           32 bit
    086     Bitset           16 bit short array, (size = byte length/2).
    087     Bitset           16 bit long  array, (size = byte length/2).
    088     Bitset           64 bit
    089     Bitset          128 bit

    090     Decimal 0
    095     Decimal in string representation
    096     Decimal         128 bit short array, (size = byte length/16).
    097     Decimal         128 bit long  array, (size = byte length/16).
    099     Decimal         128 bit, [Microsoft .NET decimal representation](1)

    100     Float not a number (NaN)
    106     Float            32 bit short array, (size = byte length/4).
    107     Float            32 bit long  array, (size = byte length/4).
    104     Float            32 bit, [Microsoft.NET System.Short representation](2)

    110     Double not a number (NaN)
    115     Double in string representation
    116     Double           64 bit short array, (size = byte length/8).
    117     Double           64 bit long  array, (size = byte length/8).
    118     Double           64 bit, [Microsoft.NET System.Double representation](3)

    120     Date undefined.
    124     Date: 1. and 2. byte = year in the Gregorian calendar (signed short int),
                  3. byte = month,
                  4. byte = day of month

    125     DateTime string representation using the [Microsoft roundtrip format 'O'](4)
    126     Date             32 bit short array, (size = byte length/4).
    127     Date             32 bit long  array, (size = byte length/4).
    128     DateTime:        64 bit, [Microsoft.NET System.DateTime representation](5)
   
    130     Time undefined.
    134     Time: 1. byte = hour,
                  abs(2. byte) = minute,
                  3. and 4. byte = millisecond (unsigned short int)

            A positive 2. byte indicates UTC time. A negative 2. byte indicates local time.
 
            TimeSpan is normally defined by the application as an integer value counting
            seconds or milliseconds. The Time value (134) can also be used to represent
            a time span of up to +/-128 hours.

    136     Time             32 bit short array, (size = byte length/4).
    137     Time             32 bit long  array, (size = byte length/4).

    140     EmptyString of any encoding
    141     ASCII character    1 byte
    142     Unicode character  2 byte
    145     String UTF8, zero terminated: 0-byte marks end of string.
    146     String UTF8, length = 0...255 bytes before decoding.
    147     String UTF8, length = 0...4'294'967'295 bytes before decoding.
           
            ** Character or string arrays: **
            Empty strings or empty character arrays of any encoding are transferred
            with the EmptyString (140) tag.
            Strings can be written in different encodings by using the following tags:
            - Unsigned byte  (011), the data is interpreted as ASCII character array
                                    when the char modification attribute '240' is set.
            - UInt16         (022), the data is interpreted as Unicode 2-byte character
                                    array when the char modification attribute '240' is set.
            - UTF8 string    (14x), the data is interpreted as UTF8 encoded.

   
    15x     Yet undefined value tag following the length specifier rules.

    16x     Yet undefined value tag following the length specifier rules.


Links:
1. [Microsoft .NET decimal representation](https://msdn.microsoft.com/en-us/library/system.decimal.getbits.aspx)
2. [Microsoft.NET System.Short representation](https://msdn.microsoft.com/en-us/library/system.single.aspx)
3. [Microsoft.NET System.Double representation](https://msdn.microsoft.com/en-us/library/system.double%28v=vs.110%29.aspx)
4. [Microsoft roundtrip format 'O'](https://msdn.microsoft.com/en-us/library/az4se3k1%28v=vs.110%29.aspx#Roundtrip)
5. [Microsoft.NET System.DateTime representation](https://msdn.microsoft.com/en-us/library/system.datetime%28v=vs.110%29.aspx)



### BMS1 attribute tags

Attributes optionally describe the next block or value in the data stream.
The following attribute-tag-bytes are used:

    Tag ID    Description
    ------    -----------
    175       BlockName attribute with UTF8 string that defines the name of the next block or value.

    182       BaseBlockTypeAttribute with 2 bytes block type id, range [0...65'535].
              Defines the base element type of a collection.

    185       BlockType attribute with UTF8 string defines the type of the next block.

    195       NameValue attribute with UTF8 string defines a name and its value.
              Name and value are separated by '=' character.
              This tag is used to transfer XML attributes.

    205       Namespace attribute with UTF8 string defines name and namespace.
              Name and namespace are separated by '=' character.
              This tag is used to transfer XML namespaces.

    21x       Yet undefined attribute tag following the length specifier rules.
    22x       Yet undefined attribute tag following the length specifier rules.

    230       Collection attribute for collection with 0 elements
    231       Collection attribute with 1 byte collection length [0...255]
    232       Collection attribute with 2 byte collection length [0...65'535]
    233       Collection without predefined length. It ends at the block-end tag.
    234       Collection attribute with 4 byte collection length [0...4'294'967'295]

            A collection attribute defines that the next block contains a collection of value-
            or block type elements. All repeated elements are of the same base-block type.
            A non-null collection is framed by a BlockStart- and a BlockEnd- tag.

            ** Empty collection: **
            When the collection length specifies 0 elements, no element tags must be defined
            between the BlockStart- and a BlockEnd-tag that follow the attributes.

            ** Null collection: **
            When the collection itself is not present, then a null-tag (007) is used
            instead of BlockStart- and BlockEnd-tag. The collection attribute is optional.

            ** Collection of value types: **
            All elements of the collection have the same data type.
            BaseBlockTypeAttribute tag (182) is not used.
            Attributes defined on the collection block are applied to all elements inside the
            collection block. Each element is preceded by a tag-byte to specify its individual
            length. When the value itself has an array length specification,
            then collections of arrays are transferred.
            When the value is a string, then a collection of strings is transferred.

            ** Collection of block types: **
            Inside the collection-block, BlockStart and BlockEnd tags are repeated for each
            element of the collection.
            Attributes defined on the collection block are applied to all element blocks.
            When a BaseBlockTypeAttribute tag (182) is defined on the collection, all
            element block types must be derivable from the BaseBlockType.

    235       yet undefined attribute tag with zero terminated data string
    236       yet undefined attribute tag with 1 byte data length + data
    237       yet undefined attribute tag with 4 byte data length + data
    238       yet undefined attribute tag with 8 bytes data
    239       yet undefined attribute tag with 16 bytes data

    240       Character type modification attribute:
              Unsigned byte (01x) must be interpreted as ASCII 8 bit character (default code page).
              UInt16 (02x) must be interpreted as UNICODE 2 byte character.

    241       Alternate 1 tag set attribute: The next byte is a yet not defined tag
              with length specifiers equal to the base set.
    242       Alternate 2 tag set attribute: As tag 241.
    243       Alternate 3 tag set attribute: As tag 241.
   
    244       yet undefined attribute tag without additional data


### BMS1 frame tags

The following tag-bytes are used for message and block framing:

    Tag ID    Description
    ------    -----------

    245       BMS1 MessageStart, BMS specification version 1.0.
              MessageStart is followed by 4 bytes containing the hexadecimal number 0x544D4201
              This magic number allows a receiver to synchronize to a running data stream.
              It also allows to check the BMS version and to check whether sender and receiver
              match the byte order (little or big endian).
              When the byte order is reversed, the receiver has to reverse all received
              integer values.

    246       BlockStart without block type id. Block type may be defined by an attribute.
    247       BlockStart with 2 bytes block type id, range [0...65'535].
    248       yet undefined with 2 bytes additional data.

    249       BlockEnd, no checksum
    250       BlockEnd, with 4 byte checksum (tbd)
   
    251       MessageFooter, followed by additional attributes or values
              (for future extension) before the final MessageEnd tag.

    252       MessageEnd

    253       yet undefined framing tag with 4 bytes additional data.
    254       yet undefined framing tag with 4 bytes additional data.
    255       not allowed, invalid tag



### Predefined name/value attributes for the BMS1 message block

The outermost block of a message contains some name/value attributes (tag 195). These attributes form the message header for a higher level messaging protocol. 
When the connection is set up, the first message needs attributes 'PV' and 'SID'. All messages need the attribute 'MT'. These attribute names are case sensitive.

    **'PV': Protocol Version**
    Example: "PV=1.0"
    The 'PV' attribute is needed on the first message, when the connection is set up.
    A receiver will accept messages from a newer protocol version as long as the mayor version is not newer than its own.

    **'SID': Service ID**
    Example: "SID=MyActor/Port12"
    The service id usually is the absolute path of the service URI.
    Using the service id, a receiver can find the actor and port instance to send the message to.
    For each SID a TCP channel is opened. Therefore, only the first message needs the SID.

    **'MT': Message Type**
    Example: "MT=Q"
    The following message types are possible:
        'Q': A request message. The sender awaits a response message with the same RID.
        'R': A response message to the waiting actor port.
        'N': A notification message. No response is awaited.
        'E': An error message, the type is Remact.Net.ErrorMessage.

    **'RID': Request ID**
    Example: "RID=12345"
    The request ID is a 32 bit signed integer value. Messages with 'MT=Q', 'MT=R' or 'MT=E' need the RID attribute to release a waiting actor port.
    Request IDs are incremented by the sender up to a certain maximum. In the next message, the RID will drop to the minimum.

    **'DM': Destination Method**
    Example: "DM=Remact.ActorInfo.Remact.ActorInfo"

    On the receiving side, the payload type to deserialize can be found in two ways:
        1) The 'DM' attribute is present, this specifies a namespace and method name. The type of the first parameter of this method corresponds to the payload type.
        2) A BlockType attribute (tag 185) is present on the outermost block. This specifies namespace and class name of the payload.

    After deserializing the payload, the method to handle it can be found in four ways:
        1) The 'DM' attribute is present, this specifies a namespace and method name to be called.
        2) A response is handled in the awaiting code.
        3) The first registered method for the given port, that accepts payloads of given type (or compatible base type) is called.
        4) When no matching method is found, the default message handler is called.



### A note to implementors

I do not recommend to implement a BMS1 serializer using reflection and attributes (in C# speak).
You get much better control over serialization and versioning when implementing interfaces like the ones below.
You can keep XML and/or Json serialization attributes in your data transfer objects (DTO).
But you also implement static methods or extension methods to programmatically serialize and deserialize from BMS1.
In the end this approach will be less error prone, better maintainable and understandable than the declarative approach.


    public class Bms1MessageSerializer
    {
        IBms1Reader ReadMessageStart(Stream stream);
        T ReadMessage<T>(Func<IBms1Reader,T> readMessageDto);

        void WriteMessage(Stream stream, Action<IBms1Writer> writeDtoAction);
    }

    public interface IBms1Reader
    {
        bool ReadBool();
        int  ReadInt32();
        ....
        T    ReadBlock<T>(Func<T> readDto);   
    }

    public interface IBms1Writer
    {
        void WriteBool(bool data);
        void WriteInt32(Int32 data);
        ....
        void WriteBlock(Action writeDto);
     }


I recommend to start with a partial implementation. Just support some data types and no attributes.
This partial implementation must be able to read a message starting with fields of supported data types.
Then it has to skip all the rest of a complex BMS1 message including all kind of current or future tags.
It is a primary goal of the specification to support this behavior.  
Feedback is very welcome.


### Interface specification

The messages specified in the interface must be implemented on client and on service side.
BMS1 is intended to run on PLC's with limited or very specific programming features.
Usually a machine translated interface specification would mean much more overhead and no gain in the long run.
This [example interface specification](./Example Interface Specification.md) gives some hints.
Plain text documents with some markdown are well suited for version control.


### License

BMS1 is licensed under [MIT](http://www.opensource.org/licenses/mit-license.php).
Copyright (c) 2016, Stefan Forster.



