BMS1: Binary Message Stream, Version 1
======================================

**BMS1** is yet another data serialization format. The goals of its design are:

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

The BMS1 message is framed by MessageStart and MessageEnd tags.
A message contains at least one block. The block is framed by BlockStart and BlockEnd tags.
The block contains an arbitrary count of values or other blocks.

Values and blocks are optionally preceeded by attributes.

By default, values and blocks are identified by their position in the message.
Before the end of a block, additional values or blocks may be appended by a new sender.
This data is skipped by an old receiver. 

All elements of a BMS1 message start with a tag-byte.
The tag-byte defines the type of the element and the length of binary data that follows the tag-byte.

According to the [little endian, Intel convention](https://en.wikipedia.org/wiki/Endianness), 
the first byte of the binary data is the least significant byte when representing numeric data.

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

	<AttributeType>::= Name | Type | ArraySize | NameValue | Namespace

	<ValueType>    ::= Null | Bool | Byte | Integer | Enumeration | Bitset | Decimal | Float | DateTime | Char | String | ByteArray

	<Length>       ::= 0 | 1 | 2 | 4 | 8 | 16 | Utf8ZeroTerminated | additional 1 byte length field | additional 4 byte length field



### Length specification of BMS1 tag-bytes

A tag is represented in one byte. One byte can hold decimal values from 0 to 255.
The least significant decimal digit is used as a length specifier for the data that follows the tag.

	Tag ID	Length of data that follows the tag byte
	------	----------------------------------------
	xx0     No data follows, all contained data is zero (0), empty or is undefined for some data types.
	xx1		1 byte   (  8 bit) data follows
	xx2		2 bytes  ( 16 bit, little endian) data follows
	xx3		reserved
	xx4		4 bytes  ( 32 bit, little endian) data follows
	xx5		An arbitrary length of data follows. It is a UTF8 encoded, zero terminated character string (the last byte is '0').
	xx6		1 byte follows that defines the length of data in bytes (0...255 bytes)
	xx7		4 byte follow  that define  the length of data in bytes (0...4'294'967'295 bytes)
	xx8		8 bytes  ( 64 bit, little endian) data follows
	xx9		16 bytes (128 bit, little endian) data follows

Note A:
    The tags 00x, 01x, 24x and 25x do not have length specifiers according to the above rules. 

Note B: 
	The tags 00x and 01x are single byte tags, there is no data to be skipped after an unknown tag in this range is received.

Note C:
	Introducing new tags in the range 24x and 25x is a breaking change of the BMS1 specification.
  	The message parsing fails, when an unknow 24x or 25x tag is read because the data length is not specified for these tags. 


### BMS1 value tags

Value tags define data type and length of the next bytes in the data stream.
Wherever possible, the length is defined by the data value not by the data type.
For example, the 64 bit integer value '-100' is stored in 8 memory bytes but is streamed in 2 bytes 
(Tag-byte '071' + Value-byte '-100').

The following value-tag-bytes are used:

	Tag ID	Description
	------	-----------
    010     Bool = false,    Note: Length specifier does not apply to the 01x tag.
    011     Bool = true

    012     Null value, block or array = no data is available

    013     Second value tag set    : The next byte is a yet not defined value tag with length specifier.
    014     Second attribute tag set: The next byte is a yet not defined attribute tag with length specifier.
    015     Second framing tag set  : The next byte is a yet not defined framing tag with length specifier.
	
	016		Character type modification attribute: 
            Unsigned byte  (02x) must be interpreted as a ASCII 1 byte character. 
            Unsigned short (03x) must be interpreted as a UNICODE 2 byte character.

    020     Unsigned byte,    8 bit,         [0]
    021     Unsigned byte,    8 bit, range = [0...255]
    026     Unsigned byte,    short array or ASCII character string when attribute 016 is present.
    027     Unsigned byte,    long  array or ASCII character string when attribute 016 is present.

    030     Unsigned short   16 bit,         [0]
    031     Unsigned short   16 bit, range = [0...255]
    032     Unsigned short   16 bit, range = [0...65'535]
    036     Unsigned 16 bit  short array or Unicode wide character string when attribute 016 is present, (size = byte length/2).
    037     Unsigned 16 bit  long  array or Unicode wide character string when attribute 016 is present.

    040     Signed short     16 bit,         [0]
    041     Signed short     16 bit, range = [-128...+127]
    042     Signed short     16 bit, range = [-32'768...+32'767]
    046     Signed 16 bit    short array, (size = byte length/2).
    047     Signed 16 bit    long  array.

    050     Unsigned integer 32 bit,         [0]
    051     Unsigned integer 32 bit, range = [0...255]
    052     Unsigned integer 32 bit, range = [0...65'535]
    054     Unsigned integer 32 bit, range = [0...4'294'967'295]
    056     Unsigned integer short array, (size = byte length/4).
    057     Unsigned integer long  array.

    060     Signed integer   32 bit,         [0]
    061     Signed integer   32 bit, range = [-128...+127]
    062     Signed integer   32 bit, range = [-32'768...+32'767]
    064     Signed integer   32 bit, range = [-2'147'483'648...2'147'483'647]
    066     Signed integer   short array, (size = byte length/4).
    067     Signed integer   long  array.

    070     Signed long      64 bit,         [0]
    071     Signed long      64 bit, range = [-128...+127]
    072     Signed long      64 bit, range = [-32'768...+32'767]
    074     Signed long      64 bit, range = [-2'147'483'648...2'147'483'647]
    076     Signed 64 bit    short array, (size = byte length/8).
    077     Signed 64 bit    long  array.
    078     Signed long      64 bit, range = [-/+ full 64 bit range]

    080     Enumeration value                [0]
    081     Enumeration value        range = [-128...+127]
    082     Enumeration 16 bit       range = [-32'768...+32'767]
    084     Enumeration value        range = [-2'147'483'648...2'147'483'647]
    085     Enumeration value in string representation
    086     Enumeration 16 bit short array, (size = byte length/2).
    087     Enumeration 16 bit long  array, (size = byte length/2).
    088     Enumeration value        range = [-/+ full 64 bit range]

    090     Bitset, all bits = 0
    091     Bitset            8 bit
    092     Bitset           16 bit
    094     Bitset           32 bit
    096     Bitset           16 bit short array, (size = byte length/2).
    097     Bitset           16 bit long  array, (size = byte length/2).
    098     Bitset           64 bit
    099     Bitset          128 bit

    100     Decimal 0
    105     Decimal in string representation
    106     Decimal         128 bit short array, (size = byte length/16).
    107     Decimal         128 bit long  array, (size = byte length/16).
    109     Decimal         128 bit, [Microsoft.NET representation](https://msdn.microsoft.com/en-us/library/system.decimal.getbits.aspx)

    110     Float not a number (NaN)
    116     Float            32 bit short array, (size = byte length/4).
    117     Float            32 bit long  array, (size = byte length/4).
    114     Float            32 bit, [Microsoft.NET System.Short representation](https://msdn.microsoft.com/en-us/library/system.single.aspx)

    120     Double not a number (NaN)
    125     Double in string representation
    126     Double           64 bit short array, (size = byte length/8).
    127     Double           64 bit long  array, (size = byte length/8).
    128     Double           64 bit, [Microsoft.NET System.Double representation](https://msdn.microsoft.com/en-us/library/system.double%28v=vs.110%29.aspx)

    130     Date undefined.
    134     Date: 1. and 2. byte = year in the Gregorian calendar (signed short int), 3. byte = month, 4. byte = day of month
    135     DateTime in string representation using the [Microsoft roundtrip format 'O'](https://msdn.microsoft.com/en-us/library/az4se3k1%28v=vs.110%29.aspx#Roundtrip)
    136     Date             32 bit short array, (size = byte length/4).
    137     Date             32 bit long  array, (size = byte length/4).
    138     DateTime:        64 bit, [Microsoft.NET System.DateTime representation](https://msdn.microsoft.com/en-us/library/system.datetime%28v=vs.110%29.aspx)
    
    140     Time undefined.
	144     Time: 1. byte = hour, abs(2. byte) = minute, 3. and 4. byte = millisecond (unsigned short int)
            A positive 2. byte indicates UTC time. A negative 2. byte indicates local time. 
            TimeSpan is normally defined by the application as an integer value counting seconds or milliseconds.
            The Time value (144) can also be used to represent a time span of up to +/-128 hours.

    146     Time             32 bit short array, (size = byte length/4).
    147     Time             32 bit long  array, (size = byte length/4).

    150     EmptyString of any encoding
    151     ASCII character    1 byte
    152     Unicode character  2 byte
    155     String UTF8, zero terminated: First 0-byte marks end of string.
    156     String UTF8, length = 0...255 bytes before decoding.
    157     String UTF8, length = 0...4'294'967'295 bytes before decoding.



### BMS1 attribute tags

Attributes optionally describe the next block or value in the data stream.
The following attribute-tag-bytes are used:

	Tag ID	Description
	------	-----------
	175		Name attribute with UTF8 string that defines the name of the next block or value.

	185		Type attribute with UTF8 string that defines the type of the next block.

    191     ArrayLength attribute with 1 byte array length [0...255]
    192     ArrayLength attribute with 2 byte array length [0...65'535]
    194     ArrayLength attribute with 4 byte array length [0...4'294'967'295]
            The array attribute defines that the next value or block will be repeated [array length] times.
            All repeated elements belong to the same array value.

			** Empty arrays: **
			When the array length specifies 0 elements, then the first element-tag is one of the tags that are not followed by data:
            Value tag with length specifier = 0 (xx0), NullBlock (240) or BaseBlock (241).
            When the array itself is not present, then a null-tag (012) is used and the ArrayLength attribute is optional.

			** Arrays of simple value types: **
			Data type and other attributes from the first element must be equal for all array elements.
            Eeach element is preceeded by the tag-byte to specify its individual length.
			When the value itself has an array length specification, then 2 dimensional, jagged arrays are transferred.
			
			** Character or string arrays: **
			Empty strings or empty character arrays of any encoding are transferred with the EmptyString (150) tag.
			When the value itself has an array length specification, then an array of strings is transferred.
            When the array modification attribute '017', then additional rules apply:
            - The type can be mixed in a string array between unsigned byte, unsigned short and UTF8 string.
            - Unsigned byte  (021), the data is interpreted as a ASCII character array. 
            - Unsigned short (032), the data is interpreted as a Unicode 2-byte character array.
            - UTF8 string    (15x), the data is interpreted as UTF8 encoded

			** Arrays of blocks: **
			BlockStart and BlockEnd tags are repeated for each block of the array. Inner values are specified by their tag-byte.
            By default all blocks have the same data type and attributes.
            When the first block is defined with a BaseBlockDefinition tag (242), then blocks may contain their own, differing type definition.
			Each block type must be derivable from the base block type.


    205     NameValue attribute with UTF8 string that defines name and value.
            Name and value are separated by '=' character. Value is UTF8 encoded.
            This tag is used to transfer XML attributes.

    215     Namespace attribute with UTF8 string that defines name and namespace.
            Name and namespace are separated by '=' character.
            This tag is used to transfer XML namespaces.



### BMS1 frame tags

The following tag-bytes are used for message and block framing:

	Tag ID	Description
	------	-----------

	240		NullBlock with 2 bytes block type id, range [1...65'535]. No data and no BlockEnd is appended.
	241		BaseBlockDefinition with 2 bytes block type id, range [1...65'535]. No data and no BlockEnd is appended.

	242		BlockStart without block type id. Block type may be defined by an attribute.
	243		BlockStart with 2 bytes block type id, range [1...65'535].

	244		BlockEnd, no checksum
	245		BlockEnd, with 2 byte checksum (tbd)

	246		yet undefined, invalid tag
	247		yet undefined, invalid tag
	248		yet undefined, invalid tag
	249		yet undefined, invalid tag

	250		BMS1 MessageStart, BMS specification version 1.0.
            MessageStart is followed by 4 bytes containing the decimal data [01, 66, 77, 83].  
            This magic number allows a receiver to synchronize to a running data stream.
	
	251     BMS1 MessageStart, short form without synchronization data.
	252     BMS2 (next version) MessageStart, short form without synchronization data.

	253		MessageFooter, followed by additional attributes or values (for future extension) and the final MessageEnd tag.
	254		MessageEnd

	255		not allowed, invalid tag

Note: The length specifier does not apply to the 24x and 25x tag-bytes.



### A note to implementors

I do not recommend to implement a BMS1 serializer using reflection and attributes (in C# speak).
You get much better control over serialization and versioning when implementing interfaces like the ones below.
You can keep XML and/or Json serialization attributes in your data transfer objects (DTO).
But you also implement interface IBms1Block on the DTO to programmatically serialize and deserialize from BMS1. 
In the end this approach will be less error prone, better maintainable and understandable than the declarative approach.

	public interface IBms1Block
	{
		public void Bms1Read  (IBms1StreamReader stream);
		public void Bms1Write (IBms1StreamWriter stream);
	}

	public interface IBms1StreamReader
	{
		public bool EndOfMessage {get;} // before message start or after message end
		public bool EndOfBlock   {get;} // before block start or after block end

		public void ReadMessageStart (ref int messageType, ref List<Bms1Attribute> attributes);
		public void ReadMessage (ref IBms1Block message);

		public bool ReadBlock (int blockType, ref IBms1Block block); // returns false, when not read because: NoData(null), not matching type, EndOfBlock, EndOfMessage
		public bool ReadByte  (ref byte data); // returns false, when not read because: NoData(null), EndOfBlock, EndOfMessage
		public bool ReadInt   (ref int  data); // returns false, when not read because: NoData(null), EndOfBlock, EndOfMessage
        ....
	}

	public interface IBms1StreamWriter
	{
		public bool EndOfMessage {get;} // before message start or after message end
		public bool EndOfBlock   {get;} // before block start or after block end

		public void WriteMessage (int messageType, IBms1Block message, List<Bms1Attribute> attributes);

		public void WriteBlock (int  blockType, IBms1Block block); // inner block type is written by the outer block
		public void WriteByte  (byte data);
		public void WriteInt   (int  data);
        ....
	}



### License

BMS1 is licensed under [MIT](http://www.opensource.org/licenses/mit-license.php).
Copyright (c) 2015, Stefan Forster.


