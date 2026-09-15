<p align='center'>
<img src='https://img.halfrost.com/Blog/ArticleImage/132_0.png'>
</p>

# Detailed Explanation of the HTTP/2 Header Compression Algorithm — HPACK


## I. Introduction

In HTTP/1.1 (see [[RFC7230](https://tools.ietf.org/html/rfc7230)), header fields are not compressed. As the number of requests within a webpage grows to tens to hundreds, redundant header fields in these requests unnecessarily consume bandwidth, significantly increasing latency.

SPDY [[SPDY]](https://tools.ietf.org/html/rfc7541#ref-SPDY) initially addressed this redundancy issue by compressing header fields using the DEFLATE [[DEFLATE]](https://tools.ietf.org/html/rfc7541#ref-DEFLATE) format, which proved to be very efficient at representing redundant header fields. However, this approach exposed security risks, such as those demonstrated by the CRIME attack (which easily leaks compression ratio information) (see [[CRIME]](https://tools.ietf.org/html/rfc7541#ref-CRIME)).

This specification defines HPACK, a novel compression method that eliminates redundant header fields, limits vulnerabilities to known security attacks, and has limited memory requirements in constrained environments. Section 7 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E6%8E%A2%E6%B5%8B%E5%8A%A8%E6%80%81%E8%A1%A8%E7%8A%B6%E6%80%81] describes the potential security issues of HPACK.

The HPACK format is intentionally designed to be simple and inflexible. Both characteristics reduce the risk of interoperability or security issues due to implementation errors. No extension mechanism is defined; the format can only be changed by defining complete replacements.


### 1. Overview

![](https://img.halfrost.com/Blog/ArticleImage/132_1.png)

The format defined in this specification treats the header field list as an ordered set of name-value pairs, which may include duplicate pairs. Names and values ​​are considered opaque sequences of octets, and the order of header fields remains unchanged after compression and decompression.

The header field tables map header fields to index values, thus obtaining the encoding. These header field tables can be incrementally updated when encoding or decoding new header fields.


In the encoded form, header fields are represented either literally or as references to header fields in the header fields table. Therefore, a mix of references and literals can be used to encode a list of header fields.

Literal values ​​can be directly encoded or static Huffman coding can be used (maximum compression ratio 8:5).

The encoder is responsible for deciding which header fields to insert as new entries into the header field table. The decoder performs modifications to the header field table specified by the encoder, thereby rebuilding the list of header fields in the process. This keeps the decoder simple and allows it to interoperate with a variety of encoders.

[Appendix C](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#1-%E6%95%B4%E6%95%B0%E8%A1%A8%E7%A4%BA%E7%9A%84%E7%A4%BA%E4%BE%8B) provides examples of representing header fields using these different mechanisms.


Note: In HTTP/2, the definitions of request and response header fields remain unchanged, with only minor differences: all header field names are lowercase, and the request line is now split into individual :method, :scheme, :authority, and :path pseudo-header fields.
>

### 2. Agreement

The keywords “must,” “must not,” “should,” “should prohibit,” “should,” “should not,” “recommend,” “may,” and “optional” in this document are defined in RFC 2119 [[RFC2119]](https://tools.ietf.org/html/rfc2119).

All values ​​are arranged in network byte order. Unless otherwise specified, values ​​are unsigned. Literal values ​​are provided in decimal or hexadecimal where appropriate.


### 3. Terminology


The following terms are used in this article:

Header Field: A name-value pair. Both the name and value are treated as an opaque sequence of eight bytes.

Dynamic Table: A dynamic table (see Section 2.3.2 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-%E5%8A%A8%E6%80%81%E8%A1%A8)) is a table that associates stored header fields with index values. This table is dynamic and specific to the encoding or decoding context.

A static table (see Section 2.3.1 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E9%9D%99%E6%80%81%E8%A1%A8)) is a table that statically associates frequently occurring header fields with index values. This table is ordered, read-only, always accessible, and can be shared across all encoding or decoding contexts.

Header List: A header list is an ordered collection of header fields that are fused together and may contain duplicate header fields. The complete list of header fields contained in an HTTP/2 header block is the header list.

Header Field Representation: Header fields can be represented in literal or indexed form (see [Section 2.4](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#4-header-field-representation)).

Header Block: An ordered list of header field representations that, when decoded, will produce a complete header list.



## II. Overview of the Compression Process

This specification does not describe the specific algorithm of the encoder. Instead, it precisely defines how the decoder is expected to work, thus allowing the encoder to produce any encoding permitted by this definition.

### 1. Header List Ordering

HPACK preserves the order of header fields within the header list. The encoder must sort the header field representations in the header block according to their order in the original header list. The decoder must sort the header fields in the decoded header list according to their order in the header block.

### 2. Encoding and Decoding Contexts

To decompress the header block, the decoder only needs to maintain a single dynamic table (see Section 2.3.2) as the decoding context. No other dynamic state is required.

When used for bidirectional communication (e.g., in HTTP), the encoding and decoding dynamic tables maintained by the endpoints are completely independent, meaning the request and response dynamic tables are separate.

### 3. Indexing Tables

HPACK uses two tables to associate header fields with indexes. The static table (see Section 2.3.1) is predefined and contains common header fields (most of which have null values). The dynamic table (see Section 2.3.2) is dynamic and can be used by the encoder to index duplicate header fields in the encoded header list.

These two tables are merged into one address space for defining index values ​​(see [Section 2.3.3](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E7%B4%A2%E5%BC%95%E5%9C%B0%E5%9D%80%E7%A9%BA%E9%97%B4)).

### (1) Static Table

The static table consists of a predefined static list of header fields. Its entries are defined in [Appendix A](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#%E4%B8%80-%E9%9D%99%E6%80%81%E8%A1%A8%E5%AE%9A%E4%B9%89).


### (2) Dynamic Tables


Dynamic tables contain a list of header fields maintained in **first-in, first-out** order. The first and latest entries in a dynamic table are located at the lowest index, while the oldest entry is located at the highest index.


The dynamic table is initially empty. Entries are added as each header block is decompressed. The dynamic table can contain duplicate entries (i.e., entries with the same name and the same value). Therefore, the decoder must not treat duplicate entries as errors.

The encoder decides how to update the dynamic table, thus controlling how much memory the dynamic table uses. To limit the decoder's storage requirements, the size of the dynamic table is strictly limited (see [Section 4.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-maximum-table-size)).

The decoder updates the dynamic table while processing the list of header field representations (see [Section 3.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-header-field-representation-processing)).




### (3) Index address space


Static tables and dynamic tables are combined into a single index address space.

An index between 1 and the length of the static table (inclusive) refers to an element in the static table (see [Section 2.3.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E9%9D%99%E6%80%81%E8%A1%A8)).

An index that is strictly greater than the length of the static table refers to an element in the dynamic table (see Section 2.3.2 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-%E5%8A%A8%E6%80%81%E8%A1%A8)). Subtracting the length of the static table will give you the index of the dynamic table.

An index that is strictly greater than the sum of the lengths of the two tables must be considered a decoding error.

The following diagram shows the entire effective index address space for the static table size of s and the dynamic table size of k.


![](https://img.halfrost.com/Blog/ArticleImage/132_3_.png)



### 4. Header Field Representation

Encoded header fields can be represented as indexes or literals.

The indexed representation defines a header field as a reference to an entry in a static or dynamic table (see [Section 6.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E7%B4%A2%E5%BC%95-header-%E5%AD%97%E6%AE%B5%E8%A1%A8%E7%A4%BA)); the literal representation defines the header field by specifying its name and value. The header field name can be represented literally or as a reference to an entry in a static or dynamic table. The header field value is represented literally. Three different literal representations are defined:

- Add a header field at the beginning of the dynamic table as a literal representation of the new entry (see [Section 6.2.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E5%B8%A6%E5%A2%9E%E9%87%8F%E7%B4%A2%E5%BC%95%E7%9A%84%E5%AD%97%E9%9D%A2-header-%E5%AD%97%E6%AE%B5)).

- Do not add the header field to the literal representation of the dynamic table (see [Section 6.2.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-%E4%B8%8D%E5%B8%A6%E7%B4%A2%E5%BC%95%E7%9A%84%E5%AD%97%E9%9D%A2-header-%E5%AD%97%E6%AE%B5)).

- Do not add the header field to the literal representation of the dynamic table. Instead, specify that the header field should always be in literal form, especially when re-encoded by the mediator (see [Section 6.2.3](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E4%BB%8E%E4%B8%8D%E7%B4%A2%E5%BC%95%E7%9A%84%E5%AD%97%E9%9D%A2-header-%E5%AD%97%E6%AE%B5)). This indicates that header field values ​​are protected from damage after compression (see Section 7.1.3 for more details) (https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E6%B0%B8%E4%B8%8D%E7%B4%A2%E5%BC%95%E7%9A%84%E5%AD%97%E9%9D%A2)).

To protect sensitive header field values ​​(see [Section 7.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E6%8E%A2%E6%B5%8B%E5%8A%A8%E6%80%81%E8%A1%A8%E7%8A%B6%E6%80%81)), one of these literal representations can be chosen for security reasons.

The literal representation of the header field name or header field value can be directly or using static Huffman coding to encode the octet sequence (see [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)).



## III. Decoding the header block

### 1. Header Block Processing

The decoder processes the header blocks sequentially to reconstruct the original header list.

The header block is a concatenation of header field representations. Different possible header field representations are described in Section 6 (https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E7%B4%A2%E5%BC%95-header-%E5%AD%97%E6%AE%B5%E8%A1%A8%E7%A4%BA).

Once a header field is decoded and added to the reconstructed header list, it cannot be deleted. Header fields added to the header list can be safely passed to the application.

By passing the result header field to the application, the decoder requires minimal temporary memory in addition to the memory needed for the dynamic table.


### 2. Header Field Representation Processing

This section defines the process of processing the header block to obtain the header list. To ensure that decoding will successfully produce the header list, the decoder must adhere to the following rules.

All header field representations contained in the header block will be processed in the order they appear, as shown below. For details on the format of various header field representations and some other processing instructions, please see [Section 6](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E7%B4%A2%E5%BC%95-header-%E5%AD%97%E6%AE%B5%E8%A1%A8%E7%A4%BA).

The indexed representation needs to perform the following operations:

- The header field corresponding to the referenced entry in the static or dynamic table is appended to the decoded header list.

For the missing "_literal representation_" in the dynamic table, the following operations need to be performed:

- The header field is appended to the decoded header list.

Adding "_literal representation_" to the dynamic table requires the following operations:

- The header field is appended to the decoded header list.
- The header field is inserted at the beginning of the dynamic table. This insertion may result in the eviction of previous entries in the dynamic table (see [Section 4.4](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#4-entry-eviction-when-adding-new-entries)).

## IV. Dynamic Table Management

![](https://img.halfrost.com/Blog/ArticleImage/132_2.png)

To limit the storage requirements at the decoder end, the size of the dynamic table is restricted.

The dynamic dictionary is context-dependent, requiring a different dictionary to be maintained for each HTTP/2 connection.

### 1. Calculating Table Size

The size of a dynamic table is the sum of the sizes of its entries. The size of an entry is the sum of the length of its name (in octets) (as defined in [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)), the length of its value (in octets), and 32. The size of an entry is calculated using the lengths of its name and value without applying any Huffman coding.

Note: The additional 32 octets illustrate the estimated overhead associated with the entry. For example, an entry structure that uses two 64-bit pointers to reference the name and value of an entry and two 64-bit integers to count the number of references to that name and value would have an overhead of 32 octets. (64 * 2 * 2 / 8 = 32 bytes)




### 2. Maximum Table Size 

The HPACK protocol determines the maximum size that the encoder is allowed to use for dynamic tables. In HTTP/2, this value is determined by the SETTINGS_HEADER_TABLE_SIZE setting (see Section 6.5.2 of [[HTTP2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2-HTTP-Frames-Definitions.md#2-defined-settings-parameters)).

The encoder can choose to use a capacity smaller than this maximum size (see [Section 6.3](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E5%8A%A8%E6%80%81%E8%A1%A8%E5%A4%A7%E5%B0%8F%E6%9B%B4%E6%96%B0)), but the selected size must remain less than or equal to the maximum capacity set by the protocol.

Changes in the maximum size of dynamic tables are caused by updates to the dynamic table size (see Section 6.3 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E5%8A%A8%E6%80%81%E8%A1%A8%E5%A4%A7%E5%B0%8F%E6%9B%B4%E6%96%B0)). Dynamic table size updates must occur at the beginning of the first header block following a change to the dynamic table size. In HTTP/2, this follows the settings confirmation (see Section 6.5.3 of [[HTTP2]](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2-HTTP-Frames-Definitions.md#3-settings-synchronization)).

Between the transmission of two header blocks, the maximum table size may be updated multiple times. If this size changes more than once during this interval, then the smallest maximum table size that occurs during this interval must be signaled during the dynamic table size update. A final maximum size signal will always be emitted, resulting in at most two dynamic table size updates. This ensures that the decoder can perform eviction based on the decrease in dynamic table size (see [Section 4.3](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-entry-eviction-when-dynamic-table-size-changes)).

This mechanism allows you to completely remove entries from a dynamic table by setting the maximum size to 0, and then restore them.

HTTP/2 encourages the use of as few connections as possible, and header compression is one of the key reasons for this: the more requests and responses generated on the same connection, the more complete the dynamic dictionary accumulates, and the better the header compression effect.

### 3. Entry Eviction When Dynamic Table Size Changes

If the maximum size of the dynamic table is reduced, entries will be evicted from the end of the dynamic table until the size of the dynamic table is less than or equal to the maximum size.


### 4. Entry Eviction When Adding New Entries 

Before adding a new entry to the dynamic table, entries will be evict from the end of the dynamic table until the size of the dynamic table is less than or equal to (maximum size - new entry size) or until the table is empty.

If the size of a new entry is less than or equal to the maximum size, the entry will be added to the table. Attempting to add an entry larger than the maximum size is not an error; however, attempting to add an entry larger than the maximum size will cause the table to be cleared of all existing entries, resulting in an empty table.

A new entry can reference the name of entry A in the dynamic table. When the new entry is added to the dynamic table, entry A will be evicted. Note that if the referencing entry was deleted from the dynamic table before inserting the new entry, you should avoid deleting the reference name.


## V. Basic Type Representation

HPACK encoding uses two primitive types: unsigned variable-length integers and octet strings.


### 1. Integer Representation

Integers are used to represent the name index, header field index, or string length. Integer representations can begin anywhere within an octet. For optimized processing, integer representations always end at the end of the octet.

The integer is divided into two parts: a prefix that fills the current octet and a list of optional octets that are used if the integer value does not fit the prefix. The number of bits in the prefix (called N) is a parameter represented by the integer.

If the integer value is small enough, i.e. strictly less than 2^N-1, it is encoded in an N-bit prefix.

![](https://img.halfrost.com/Blog/ArticleImage/132_4.png)

In the example above, N = 5, so the largest integer that can be represented is 2^5 - 1 = 31.


If the integer value is greater than 2^N-1, all bits of the prefix are set to 1, and the value reduced by 2^N-1 is encoded using a list of one or more octets. The most significant bit of each octet is used as a contiguous flag: its value is set to 1 except for the last octet in the list. The remaining bits of the octets are used to encode the reduced value.


![](https://img.halfrost.com/Blog/ArticleImage/132_5_.png)


Decoding an integer value from a list of octets is done by reversing the order of the octets in the list. Then, for each octet, its most significant bit is removed. The remaining bits of the octets are concatenated, and the resulting value is incremented by 2^N-1 to obtain the integer value.

The prefix size N is always between 1 and 8 bits. Integers starting from an octet boundary will have an 8-bit prefix.

The pseudocode for representing the integer I is as follows:

```c
   if I < 2^N - 1, encode I on N bits
   else
       encode (2^N - 1) on N bits
       I = I - (2^N - 1)
       while I >= 128
            encode (I % 128 + 128) on 8 bits
            I = I / 128
       encode I on 8 bits
```

The pseudocode for decoding the integer I is as follows:

```c
   decode I from the next N bits
   if I < 2^N - 1, return I
   else
       M = 0
       repeat
           B = next octet
           I = I + (B & 127) * 2^M
           M = M + 7
       while B & 128 == 128
       return I
```

An example illustrating integer encoding is provided in Appendix C.1 (https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#1-%E6%95%B4%E6%95%B0%E8%A1%A8%E7%A4%BA%E7%9A%84%E7%A4%BA%E4%BE%8B).


Integer representation allows for values ​​of indeterminate size. The encoder may also send large numbers of zero values, which could waste octets and potentially cause integer overflow. Integer encodings (values ​​or octet lengths) that exceed implementation limits must be treated as decoding errors. Different limits can be set for each different use case of integers, based on implementation constraints.



### 2. String Literal Representation

The `name` field and `value` field of the `header` class can be represented as string literals. String literals can be encoded as octets directly or as octet sequences using Huffman coding (see [[HUFFMAN]](https://tools.ietf.org/html/rfc7541#ref-HUFFMAN))).


![](https://img.halfrost.com/Blog/ArticleImage/132_6.png)


The string literal representation contains the following fields:

- H：  
  A flag H indicates whether the octet of the string has been Huffman encoded.

- String Length：  
  The number of octets used to encode the string literal is encoded as an integer with a 7-bit prefix (see [Section 5.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-integer-representation)).

- String Data：
  The encoded data of the string literal. If H is '0', the encoded data is the original eight bytes of the string literal. If H is '1', the encoded data is the Huffman code of the string literal.

String literals using Huffman coding are available in [Appendix B](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#%E4%BA%8C-%E9%9C%8D%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81). Encode using the Huffman code defined in [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#4-%E6%9C%89%E9%9C%8D%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81%E8%AF%B7%E6%B1%82%E7%9A%84%E7%A4%BA%E4%BE%8B](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#4-%E6%9C%89%E9%9C%8D%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81%E8%AF%B7%E6%B1%82%E7%9A%84%E7%A4%BA%E4%BE%8B). The examples in [Appendix C.6](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_HPACK-Example.md#6-%E6%9C%89%E9%9C%8D%E5%A4%AB%E6%9B%BC%E7%BC%96%E7%A0%81%E5%93%8D%E5%BA%94%E7%9A%84%E7%A4%BA%E4%BE%8B) are shown. The encoded data is a bitwise concatenation of the code corresponding to each octet of the string literal.

Since Huffman-coded data does not always end at the boundary of an octet, padding is inserted after it until the boundary of the next octet. To avoid misinterpreting this padding as part of the string literal, the most significant bit of the code corresponding to the EOS (end-of-string) notation is used.

During decoding, incomplete codes at the end of the encoded data will be treated as padding and discarded. Padding longer than 7 bits must be considered a decoding error. Padding that does not correspond to the most significant bit of the EOS symbol code must be considered a decoding error. String literals containing Huffman-encoded EOS symbols must be considered decoding errors.


## VI. Binary Format

This section describes the detailed format of each different header field representation and the dynamic table size update command.

### 1. Index header field representation


The index header field indicates an entry that can be identified in a static or dynamic table (see Section 2.3 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-indexing-tables]).

The indexed header field indicates that the header field will be added to the list of decoded headers, as described in [Section 3.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-header-field-representation-processing).


![](https://img.halfrost.com/Blog/ArticleImage/132_7_.png)

The above scenario corresponds to the situation where both Name and Value are in the index table (including static and dynamic tables).

The index header field begins with a 1-bit pattern "1", followed by an index that matches the header field, represented as a 7-bit prefixed integer (see [Section 5.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-integer-representation)).

Index value 0 is not used. If index value 0 is found in the index header field representation, it must be considered a decoding error.



### 2. Literal header field identifier

The header field representation contains a literal header field value. The header field name is provided in literal form or by referencing an existing table entry in a static or dynamic table (see Section 2.3 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-indexing-tables](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-indexing-tables)).

This specification defines three forms of literal header field representation: with index, without index, and never indexed.



### (1). Literal header field with incremental index



A literal header field with an incremental index representation appends the header field to the decoded header list and inserts it as a new entry into the dynamic table.


![](https://img.halfrost.com/Blog/ArticleImage/132_9.png)

The above scenario corresponds to the situation where the Name is in the index table (including static and dynamic tables), and the Value needs to be encoded and passed, while also being added to the dynamic table.

![](https://img.halfrost.com/Blog/ArticleImage/132_10.png)

The above scenario corresponds to the situation where both Name and Value need to be encoded and passed, and then added to the dynamic table simultaneously.

A literal header field with an incremental index representation begins with a 2-digit "01" pattern.

If the header field name `name` matches the header field name `name` of an entry stored in a static or dynamic table, then the index of that entry can be used to represent the header field name `name`. In this case, the entry's index is represented as an integer with a 6-digit prefix (see [Section 5.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-integer-representation)). This value is generally non-zero.

Otherwise, the header field name `name` is represented as a string literal (see [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)). Use the value 0 instead of the 6-bit index, followed by the header field name `name`.

Both forms of header field name representation are followed by header field value in string literal form (see [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)).



### (2). Literal header field without index



A literal header field without an index representation allows the header field to be appended to the decoded header list without modifying the dynamic table.


![](https://img.halfrost.com/Blog/ArticleImage/132_11.png)

The above scenario corresponds to the Name being in the index table (including static and dynamic tables), while the Value needs to be encoded and passed, and is not added to the dynamic table.

![](https://img.halfrost.com/Blog/ArticleImage/132_12.png)

The above scenario corresponds to the situation where Name and Value need to be encoded and passed, and are not added to the dynamic table.

A literal header field without an index starts with the 4-digit pattern "0000".

If the header field name `name` matches the header field name `name` of an entry stored in a static or dynamic table, then the index of that entry can be used to represent the header field name `name`. In this case, the entry's index is represented as an integer with a 4-digit prefix (see [Section 5.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-integer-representation)). This value is generally non-zero.

Otherwise, the header field name `name` is represented as a string literal (see [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)). Use the value 0 instead of the 4-bit index, followed by the header field name `name`.

Both forms of header field name representation are followed by the string literal header field value (see [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)).



### (3). Literal header fields that are never indexed

The literal, never-indexed representation of the header field allows the header field to be appended to the decoded header list without modifying the dynamic table. Middleware must use the same representation to encode this header field.

![](https://img.halfrost.com/Blog/ArticleImage/132_13.png)

The above scenario corresponds to the Name being in the index table (including static and dynamic tables), while the Value needs to be encoded and passed, and is never added to the dynamic table.


![](https://img.halfrost.com/Blog/ArticleImage/132_14.png)

The above scenario corresponds to the situation where Name and Value need to be encoded and passed, and are never added to the dynamic table.

The literal representation of a header field that is never indexed begins with a 4-digit pattern of "0001".

When a header field is represented as a literal header field that is never indexed, this particular literal representation must be used for encoding. In particular, when a peer sends a received header field, and the received header is represented as a literal header field that is never indexed, it must use the same representation to forward that header field.

This is intended to protect header field values ​​by compressing them to prevent them from being put at risk (see Section 7.1 for more details) (https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-%E6%8E%A2%E6%B5%8B%E5%8A%A8%E6%80%81%E8%A1%A8%E7%8A%B6%E6%80%81)).

The encoding of this representation is the same as that of the literal header field without an index (see [Section 6.2.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-%E4%B8%8D%E5%B8%A6%E7%B4%A2%E5%BC%95%E7%9A%84%E5%AD%97%E9%9D%A2-header-%E5%AD%97%E6%AE%B5)).

### 3. Dynamic Table Size Update

The dynamic table size update means changing the size of the dynamic table.

![](https://img.halfrost.com/Blog/ArticleImage/132_8.png)

Dynamic table size updates begin with a 3-bit pattern of "001", followed by the new maximum size, represented as an integer with a 5-bit prefix (see [Section 5.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-integer-representation)).

The new maximum size must be less than or equal to the limit determined by the protocol using HPACK. Values ​​exceeding this limit must be considered decoding errors. In HTTP/2, this limit is the last value of the SETTINGS_HEADER_TABLE_SIZE parameter (see Section 6.5.2 of HTTP/2) (https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2-HTTP-Frames-Definitions.md#3-settings-synchronization) received from the decoder and confirmed by the encoder (see Section 6.5.3 of HTTP/2).

Reducing the maximum size of a dynamic table will result in the eviction of entries (first-in, first-out) (see [Section 4.3](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-entry-eviction-when-dynamic-table-size-changes)).

>
There are two ways to update the dynamic table size: one is to modify it directly in the HEADERS frame (starting with the "001" 3-digit mode), and the other is to set it in SETTINGS_HEADER_TABLE_SIZE in the SETTINGS frame.
>

## VII. Safety Precautions

This section introduces the potential security vulnerabilities of HPACK:

- Use compression as a length-based prediction to verify conjectures about encryption compressed into a shared compression context.

- Denial of service due to exhaustion of decoder processing or storage capacity.


### 1. Detecting the state of dynamic tables

HPACK reduces the length of header field encoding by leveraging the inherent redundancy of protocols such as HTTP. The ultimate goal of this is to reduce the amount of data required to send HTTP requests or responses.

An attacker can probe the compression context used to encode header fields, and can also define the header fields to be encoded and transmitted, observing the length of these fields after encoding. When an attacker can perform both operations simultaneously, they can adaptively modify the request to confirm a hypothesis about the state of a dynamic table. If the hypothesis is compressed to a shorter length, the attacker can observe the length of the encoding and infer that the hypothesis is correct.

Even with Transport Layer Security (TLS) protocols (see [[TLS12]](https://tools.ietf.org/html/rfc7541#ref-TLS12)), it is still possible to be attacked because TLS provides cryptographic protection for content, but only provides limited content length protection.

Note: Padding schemes offer only limited protection against attackers with these capabilities, potentially only forcing them to increase the number of guesses required to estimate the length associated with a given guess. Padding schemes can also directly resist compression by increasing the number of bits transmitted.


Attacks such as CRIME (https://tools.ietf.org/html/rfc7541#ref-CRIME) demonstrate the existence of these attackers. A specific attack exploits the fact that DEFLATE (https://tools.ietf.org/html/rfc7541#ref-DEFLATE) removes redundancy based on prefix matching. This allows the attacker to determine one character at a time, reducing an exponential-time attack to a linear-time attack.



### (1). Applicable to HPACK and HTTP

HPACK mitigates, but does not completely prevent, attacks modeled after CRIME by forcing a guess to match the entire header field value instead of a single character. Attackers can only know whether the guess is correct, thus simplifying the attack to brute-force guessing of header field values. Therefore, the feasibility of recovering a specific header field value depends on the entropy of the value. As a result, values ​​with high entropy are less likely to be successfully recovered. However, low-entropy values ​​remain vulnerable.

This type of attack can occur whenever two untrusted entities exchange requests or responses over a single HTTP/2 connection. If a shared HPACK compressor allows one entity to add entries to a dynamic table, and another entity accesses those entries, it can learn the table's state.

A request or response from an untrusted entity will occur when the middleware does the following:

- Sending requests from multiple clients on a single connection to the origin server.

- Retrieve responses from multiple origin servers and send them over a shared connection with the client.

Web browsers also need to assume that requests from different web sources [[ORIGIN]](https://tools.ietf.org/html/rfc7541#ref-ORIGIN) on the same connection are made by entities that do not trust each other.



### (2). Relieve


Requiring HTTP header fields to be cryptographic allows users to use values ​​with sufficient entropy to make guessing impossible. However, this is impractical as a general solution because it forces all HTTP users to take steps to mitigate attacks. It will impose new restrictions on how HTTP is used.


HPACK's implementation does not impose constraints on HTTP users, but rather on how compression is applied to limit the potential for dynamic table probing.

The ideal solution is to isolate access to dynamic tables based on the entity that is constructing the header field. The header field value added to the table will be attributed to a single entity, and only the entity that created the specific value can retrieve it.

To improve the compression performance of this option, certain entries can be marked as public. For example, a web browser might make the value of the Accept-Encoding header field available in all requests.

An encoder unfamiliar with the origin of a header field might introduce a penalty mechanism for header fields with many different values. If an attacker makes numerous attempts to guess the header field value, triggering the penalty mechanism, the header field will no longer be compared with the dynamic table entity in future messages. This effectively prevents further guessing.

Note: If an attacker has a reliable method to reinstall values, simply deleting the entry corresponding to the header field from the dynamic table may be an ineffective attack. For example, requests to load images in a web browser often contain cookie header fields (a potentially high-value target for such attacks), and websites can easily force the loading of images, thereby refreshing the entries in the dynamic table.

The response may be inversely proportional to the length of the header field value. Shorter values ​​are more likely to mark the header field as no longer used in a dynamic table, either more quickly or with a higher probability.



### (3). Literals that are never indexed

Implementations can also choose not to compress sensitive header fields, but instead encode their values ​​literally to protect them.

Refusing to generate an indexed representation of the header field is only effective if compression is avoided across all hops. A literal "never indexed" (see [Section 6.2.3](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E4%BB%8E%E4%B8%8D%E7%B4%A2%E5%BC%95%E7%9A%84%E5%AD%97%E9%9D%A2-header-%E5%AD%97%E6%AE%B5)) can be used to signal to middleware that a specific value is intentionally sent as a literal.

Middleware must not re-encode values ​​that use a literal representation that is never indexed with another representation that will be indexed. If re-encoding is performed using HPACK, then a literal representation that is never indexed must be used.

The choice to use a literal representation of the header field that is never indexed depends on several factors. Because HPACK does not prevent guessing the entire header field value, it is easier for an attacker to recover short or low-entropy values. Therefore, the encoder might choose not to index values ​​with low entropy.

The encoder may also choose not to index values ​​of header fields that are considered to be of high value or sensitive to recovery (such as cookie or authorization header fields).

Conversely, if the value is public, the encoder might prefer the index value of a header field that is small or has no value. For example, the User-Agent header field typically doesn't change between requests and is sent to any server. In this case, confirming that a specific User-Agent value has been used provides little value.

Please note that as new attacks are discovered, these standards for using literal representations that are never indexed will evolve over time.



### 2. Static Huffman Coding

There are currently no attacks targeting static Huffman coding. One study showed that using a static Huffman coding table can lead to information leakage; however, the same study concluded that attackers cannot exploit this information leakage to recover any meaningful information (see [[PETAL]](https://tools.ietf.org/html/rfc7541#ref-PETAL)).

Dynamic Huffman coding is vulnerable to attacks!


### 3. Memory Management

Attackers can attempt to exhaust the endpoint's memory. HPACK is designed to limit the peak memory allocation and state amount of the endpoint.

The amount of memory used by the compressor is limited by the maximum size defined in the dynamic tables conforming to the HPACK protocol. In HTTP/2, this value is controlled by the decoder by setting the parameter SETTINGS_HEADER_TABLE_SIZE (see Section 6.5.2 of [[HTTP2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2-HTTP-Frames-Definitions.md#2-defined-settings-parameters)). This limit takes into account both the size of the data stored in the dynamic tables and a small amount of overhead.


The decoder can limit the amount of state memory used by setting an appropriate value for the maximum size of the dynamic table. In HTTP/2, this is achieved by setting an appropriate value for the SETTINGS_HEADER_TABLE_SIZE parameter. The encoder can limit the amount of state memory it uses by signaling that the dynamic table size is smaller than the state allowed by the decoder (see Section 6.3 [https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#3-%E5%8A%A8%E6%80%81%E8%A1%A8%E5%A4%A7%E5%B0%8F%E6%9B%B4%E6%96%B0)).

The amount of temporary memory consumed by the encoder or decoder can be limited by processing header fields sequentially. Implementations are not required to retain the complete list of header fields. However, note that applications may need to retain the complete header list for other reasons. Even if HPACK does not force this, application constraints may make it necessary.


### 4. Implementation Limitations

HPACK implementers need to ensure that large integer values, long integer encodings, or long string literals do not create security vulnerabilities.

An implementation must set a limit on the integer values ​​it accepts and the length of the encoding (see [Section 5.1](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#1-integer-representation)). Similarly, it must set a limit on the length of string literals (see [Section 5.2](https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/HTTP_2_Header-Compression.md#2-string-literal-representation)).


------------------------------------------------------

Reference：
  
[RFC 7541](https://tools.ietf.org/html/rfc7541)

GitHub Repo：[Halfrost-Field](HTTPS://github.com/halfrost/Halfrost-Field)
> 
> Follow: [halfrost · GitHub](HTTPS://github.com/halfrost)
>
> Source: [https://halfrost.com/http2-header-compression/](https://halfrost.com/http2-header-compression/)
