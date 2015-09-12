BMS1: Binary Message Stream, Version 1
======================================

**BMS1** is yet another data serialization format. The main goals of its design are:

* Binary data for very fast and lossless serialization.
* Compact message size.
* Compact implementation on many devices: Machine control (PLC), embeddded systems, mobile devices, PCs, ...
* Platform independent: Different programming environments and different CPUs are able to understand the messages.
* Extensible and version tolerant: 
  Old implementations of the BMS serializer and old application versions know how to skip new/unknown parts of a message.
  A device with new BMS-serializer or new BMS-message version may be plugged into a network of old devices and the system behaves as before.
  To fulfill this requirement, the application programmer must follow some rules regarding message structure. 
* Potential to replace Json or XML serializers.


### BMS1 message frame

A BMS1 Message mainly consists of several values.
Each value starts with one tag-byte that defines the value type and the data length.
The data of a value consists of the defined amount of bytes

After the tag byte, a binary data value follows. The length of the binary value is defined by the tag.

The BMS1 message is framed by MessageStart and MessageEnd tags.
A message contains at least one block. The block is framed by BlockStart and BlockEnd tags.
The block contains an arbitrary count of values or other blocks.
Values and blocks are optionally preceeded by attributes.

By default, values and blocks are identified by their position in the message.
Before the end of a block, additional values or blocks may be appended by a new sender.
This data is skipped by an old receiver. 

All elements of a BMS1 message are identified by the starting tag-byte.
The tag-byte defines the type of the element and the length of binary data that follows the tag-byte.
According to the [little endian, Intel convention](https://en.wikipedia.org/wiki/Endianness), 
the first byte of the binary data is the least significant byte when representing numeric data.

E.g:
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

	<AttributeType>::= Name | Type | ArraySize | NameValue | Namespace

	<ValueType>    ::= Null | Bool | Byte | Integer | Enumeration | Bitset | Decimal | Float | DateTime | Char | String | ByteArray

	<Length>       ::= 0 | 1 | 2 | 4 | 8 | 16 | Utf8ZeroTerminated | additional 1 byte length field | additional 4 byte length field



### Length specification of BMS1 tag-bytes

A tag is represented in one byte. One byte can hold decimal values from 0 to 255.
The least significant decimal digit is used as a length specifier for the data that follows the tag.

	Tag ID	Length of data that follows the tag byte
	------	----------------------------------------
	xx0     No data follows
	xx1		1 byte   (  8 bit) data follows
	xx2		2 bytes  ( 16 bit, little endian) data follows
	xx3		reserved
	xx4		4 bytes  ( 32 bit, little endian) data follows
	xx5		An arbitrary length of data follows. It is a UTF8 encoded, zero terminated character string (the last byte is '0').
	xx6		1 byte follows that defines the length of data (0...255 bytes)
	xx7		4 byte follow  that define  the length of data (0...4'294'967'295 bytes)
	xx8		8 bytes  ( 64 bit, little endian) data follows
	xx9		16 bytes (128 bit, little endian) data follows

Note A:
    The tags 00x, 01x and 25x do not have length specifiers according to the above rules. 

Note B: 
	The tags 00x and 01x are single byte tags, there is no data to skip after an unknown tag in this range is received.



### BMS1 value tags

Value tags define data type and length of the next bytes in the data stream.
Wherever possible, the length is defined by the data value not by the data type.
For example, the 64 bit integer value '-100' is stored in 8 memory bytes but is streamed in 2 bytes 
(Tag-byte '071' + Value-byte '-100').

The following value-tag-bytes are used:

	Tag ID	Description
	------	-----------
    010     Bool = false,    Note: Length specifier does not apply to bool values.
    011     Bool = true,     Note: Length specifier does not apply to bool values.
    012     Null = no data,  Note: Length specifier does not apply to null values.

    021     Unsigned byte,    8 bit, range = [0...255]

    031     Unsigned integer 16 bit, range = [0...255]
    032     Unsigned integer 16 bit, range = [0...65'535]

    041     Signed integer   16 bit, range = [-128...+127]
    042     Signed integer   16 bit, range = [-32'768...+32'767]

    051     Unsigned integer 32 bit, range = [0...255]
    052     Unsigned integer 32 bit, range = [0...65'535]
    054     Unsigned integer 32 bit, range = [0...4'294'967'295]

    061     Signed integer   32 bit, range = [-128...+127]
    062     Signed integer   32 bit, range = [-32'768...+32'767]
    064     Signed integer   32 bit, range = [-2'147'483'648...2'147'483'647]

    071     Signed integer   64 bit, range = [-128...+127]
    072     Signed integer   64 bit, range = [-32'768...+32'767]
    074     Signed integer   64 bit, range = [-2'147'483'648...2'147'483'647]
    078     Signed integer   64 bit, range = [-/+ full 64 bit range]

    081     Enumeration value        range = [-128...+127]
    082     Enumeration value        range = [-32'768...+32'767]
    084     Enumeration value        range = [-2'147'483'648...2'147'483'647]
    085     Enumeration value in string representation
    088     Enumeration value        range = [-/+ full 64 bit range]

    091     Bitset            8 bit
    092     Bitset           16 bit
    094     Bitset           32 bit
    098     Bitset           64 bit
    099     Bitset          128 bit

    105     Decimal in string representation
    109     Decimal         128 bit, [.NET representation](https://msdn.microsoft.com/en-us/library/system.decimal.getbits.aspx)

    114     Float            32 bit, [.NET System.Short representation](https://msdn.microsoft.com/en-us/library/system.single.aspx)

    125     Double in string representation
    128     Double           64 bit, [.NET System.Double representation](https://msdn.microsoft.com/en-us/library/system.double%28v=vs.110%29.aspx)

    134     Date: 2 byte signed short int year in the Gregorian calendar, 1 byte month, 1 byte day of month
    134     Time: 1 byte hour, 1 byte minute, 2 byte millisecond of the day
    138     DateTime:        64 bit, [.NET System.DateTime representation](https://msdn.microsoft.com/en-us/library/system.datetime%28v=vs.110%29.aspx)

    141     Char
    145     String UTF8, zero terminated

    156     Byte array, length = 0...255 bytes.           The type attribute may describe the data format.
    157     Byte array, length = 0...4'294'967'295 bytes. The type attribute may describe the data format.



### BMS1 attribute tags

Attributes describe the next block or value in the data stream.
The following attribute-tag-bytes are used:

	Tag ID	Description
	------	-----------
	165		Name attribute with UTF8 string that defines the name of the next block or value.

	175		Type attribute with UTF8 string that defines the type of the next block.

    184     ArraySize attribute with 4 byte array length [0...4'294'967'295]
    182     ArraySize attribute with 2 byte array length [0...65'535]
            The array attribute allows to reserve memory space on the receiving side.
            It defines that the next value or block will be repeated [array length] times.
            All repeated elements belong to the same array value.

    195     NameValue attribute with UTF8 string that defines name and value.
            Name and value are separated by '=' character. Value is UTF8 encoded.
            This tag is used to transfer XML attributes.

    205     Namespace attribute with UTF8 string that defines name and namespace.
            Name and namespace are separated by '=' character. Value is UTF8 encoded.
            This tag is used to transfer XML namespaces.



### BMS1 block tags

The following tag-bytes are used for message and block framing:

	Tag ID	Description
	------	-----------

	240		BlockStart, either no block type available or block type is defined by an attribute

	242		BlockStart with 2 bytes block type id, range [0...65'535]

	244		BMS1 MessageStart, BMS specification version 1.0.
            MessageStart is followed by 4 bytes containing the decimal data [01, 66, 77, 83].  
            This magic number allows a receiver to synchronize to a running data stream.
	
	250		BlockEnd,   no checksum
	251		BlockEnd,   with 2 byte checksum (tbd)

	252		MessageEnd, no checksum
	253		MessageEnd, before the next message start, additional attributes or values are sent (for future extension).

	254		not allowed, invalid tag

Note: Length specifier does not apply to the 25x tag-bytes.



### License

BMS1 is licensed under [MIT](http://www.opensource.org/licenses/mit-license.php).
Copyright (c) 2015, Stefan Forster.


