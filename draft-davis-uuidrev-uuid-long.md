---
title: "Longer Universally Unique IDentifiers (UUIDs)"
abbrev: "UUID Long"
category: std
updates: '9562'

docname: draft-davis-uuidrev-uuid-long
submissiontype: IETF 
date: {DATE}
consensus: true
v: 3
area: ART
wg: uuidrev
keyword: uuid
venue:
  group: "Revise Universally Unique Identifier Definitions (uuidrev)"
  type: "Working Group"
  mail: "uuidrev@ietf.org"
  arch: https://mailarchive.ietf.org/arch/browse/uuidrev/
  github: kyzer-davis/uuid-long/
  latest: https://github.com/kyzer-davis/uuid-long/blob/main/draft-davis-uuidrev-uuid-long.md

author:
- name: Kyzer R. Davis
  email: kydavis@cisco.com
  org: Cisco Systems

normative:
  RFC9562: RFC9562

informative:
  RFC9000: RFC9000

--- abstract

This document extends Universally Unique Identifiers (UUIDs) beyond 128 bits to facilitate enhanced collision resistance and proper room for embedding additional data within a given UUID algorithm.
These longer variable-length UUIDs ("UUID Long") leverage a previously unused variant bit "F" and feature a new sub-typing mechanisms created to ensure there is enough space to define many future UUID algorithms within this new variant of UUIDs.

This document updates {{RFC9562}}.

--- middle

# Introduction

There are a few main driving factors behind extending UUID beyond 128 bits covered by the next sections.

## The Need for Increased Entropy {#entropy}

While existing UUID formats provide sufficient entropy for most use cases;
there exists scenarios where even more entropy is required to further reduce
collision probabilities or guessability.

Further, while creating UUIDv7 during the draft phases of RFC9562, a common discussion point surrounded the number of bits allocated to entropy vs the number of bits allocated to the embedded timestamp.
The 128 bit limits on UUID created a situation where the community had to balance timestamp granularity vs entropy.
This resulted in "sliding" bits one way or other trying to find a happy medium.
While in the end a fine balance was achieved; the entire problem could have been avoided if there were more bits available to the UUID format.

With the additional length added by UUID Long; an application can generate a UUID with certainty that it is truly "unique across space and time".

## Requirements Additional Embedded Data {#data}
Some implementations require more than 128 bits to properly embed all of the application specific data they require for a given UUID algorithm.
Some examples include database metadata like entity types, checksum values, shard/partition identifiers, and even node identifiers for distributed UUID generation.

UUID Long provides ample bit space for an algorithm to properly embed all of the items required for the application logic to function.

## A better UUID sub-typing system {#versions}
128 bit UUIDs within the "OSF DCE / IETF" variant space are limited to 16 versions.
This version limit artificially inhibits innovation of new UUID algorithms (a problem partly solved by UUIDv8).

This drawback of the "OSF DCE / IETF" variant space was observed while working on {{RFC9562}}, in particular to future name-based UUID layouts that replace "UUIDv3" and "UUIDv5".
With the number of hashing algorithms available and the possibility that at any point one may be deprecated; there was little chance of getting consensus on leveraging one of the few remaining versions for such an algorithm.

With UUID Long, as per section {{subtypes}}, there is ample room for future UUID Long Algorithms.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Notational Conventions {#notation}

"UUID Long" generally references any variable length UUID longer than 128 bits while "UUID Short" references fixed-length 128-bit UUIDs in the prose of this document.

Field and Bit Layout in this document use a custom format borrowed from {{RFC9000}} rather than those featured in {{RFC9562}}.
The purpose of
this format is to summarize, not define, protocol elements. Prose defines the
complete semantics and details of structures.

Layouts items are named and then followed by a list of fields surrounded by a
pair of matching braces. Each field in this list is separated by commas.

Individual fields include length information, plus indications about fixed
value, optionality, or repetitions. Individual fields use the following
notational conventions, with all lengths in bits:

x (A):
: Indicates that x is A bits long

x (A..B):
: Indicates that x can be any length from A to B; A can be omitted to indicate
  a minimum of zero bits, and B can be omitted to indicate no set upper limit;
  values in this format always end on a byte boundary

x (L) = C:
: Indicates that x has a fixed value of C; the length of x is described by
  L, which can use any of the length forms above

This document uses network byte order (that is, big endian) values.  Fields
are placed starting from the high-order bits of each byte.

By convention, individual fields reference a complex field by using the name of the complex field.

{{fig-ex-format}} provides an example:

~~~
Example Structure {
  One-bit Field (1),
  7-bit Field with Fixed Value (7) = 61,
  Arbitrary-Length Field (..),
  Variable-Length Field (8..24),
  Field With Minimum Length (16..),
  Field With Maximum Length (..128),
}
~~~
{: #fig-ex-format title="Example Format"}

# UUID Long Format {#format}
At the core UUID Long features the same base characteristics as {{RFC9562, Section 4}} featured in UUID Short.
UUID Long may be represented in all of the same ways as you would expect with a UUID (e.g text, integer, binary, UUID URN, etc.)

The UUID Long Encoding Block starts at bit 129 with the actual UUID Long Data starting at Bit 177.
The 48 bit UUID Long Encoding block and UUID Long Data are separated by the dash character "-" in the textual representation of UUID Long.

This separation allows at-a-glance readability around the encoding block and separation from the variable-length UUID Long Data.

The generalized layout of UUID Long is {{ex-uuid-long}} and the UUID Encoding Block is found in {{ex-ul-block}}.

~~~
UUID Long Structure {
  UUID Short Part A (64),
  UUID Variant (4) = F,
  UUID Short Part B (60),
  UUID Long Encoding Block (48),
  UUID Long Data (8..16777215),
}
~~~
{: #ex-uuid-long title="Example UUID Long Bit and Field Layout"}

~~~
UUID Long Encoding Block {
  Sub-Variant Encoding (8),
  Algorithm Encoding (16),
  UUID Long Data Length Descriptor (24)
}
~~~
{: #ex-ul-block title="Example UUID Long Encoding Block Bit and Field Layout"}

Further, the base UUID Short string format with hex and dashes is also found in the string format of UUID Long.
Including this in the base syntax ensures backwards compatibility as per {{compatibility}}.
The UUID Long string representation is defined by {{ulLayout}} and {{ulDescription}}.

~~~
xxxxxxxx-xxxx-xxxx-Fxxx-xxxxxxxxxxxx-SVAAAALLLLLL-yy...zz
~~~
{: #ulLayout title='UUID Long Field Layout in Hex'}

| ID | Description | Bits |
| x | Short UUID Bits | 124 |
| F | Frozen Variant byte (for backwards compatibility). See section {{variant}}. | 4 |
| SV | One Byte Sub-Variant with 256 possible values | 8 |
| AAAA | Two Byte Algorithm with 65,536 possible values | 16 |
| LLLLLL | Three Byte length descriptor of 16,777,215 max length for UUID Long data | 24 |
| yy...zz | Variable length UUID Long Data with length described by LLLLLL. Minimum one byte, maximum  2,097,151 bytes | Variable |
{: #ulDescription title='UUID Long String Layout Descriptors'}

A properly constructed UUID Long value will be, at a minimum, 176 bits or 22 octets. 
The maximum value for a UUID Long is computed as UUID Short Length (128) + Long Encoding Block (48) + Maximum UUID Long Data Length (16,777,215) which is 16,777,391 bits (2,097,173 octets)

Applications MUST ensure that UUID Long values leverage natural byte boundaries and pad the least significant, right-most bits where required to achieve a proper byte boundary.

## Variant Field {#variant}
This section updates {{RFC9562, Section 4.1}} to split the unused final variant of "111x" into two variants as described by the {{variantUpdate}} table.
Splitting the final variant space ensures that the "E" variant may be used by future definitions while the "F" variant is used to signal a UUID Long Variant.
These "F"rozen variant bits are set to all 1's (b1111).


| Msb0 | Msb1 | Msb2 | Msb3 | Variant | Description |
|    1 |    1 | 1    | 0    | E     | Reserved for future definition. |
|    1 |    1 | 1    | 1    | F     | The variant used by UUID Long in this document. Also includes Max UUID as per {{RFC9562, Section 5.10}}. |
{: #variantUpdate title='UUID Variant Updates'}

UUID Long algorithms featuring the frozen Variant F MUST use the sub-typing logic and encoding block described in {{subtypes}}.

## Sub-Typing Logic and Encoding Block {#subtypes}

UUID Long does not re-use the "version" nomenclature (or bit positions) from {{RFC9562}}.
This serves to helps an implementations easily distinguish 128 bit or 128+ bit UUIDs in text and provide an opportunity for defining a better sub-typing system within this new variant space.

Note that "UUIDv4" or "UUID Version 4" is usually used to reference an UUID algorithm as specified by {{RFC9562, Section 5.4}} and do not represent UUID Long algorithms in this document.

UUID Long instead moves the sub-typing logic to a new 6 byte UUID Long Encoding Block placed immediately after the 128th bit in the original UUID layout.
This move of the sub-typing bit ensures the first 64 bits of the UUID Long are uninterrupted up to the frozen Variant bits.
This move also allows UUID Long to avoid the 4 bit version space that comes with drawbacks as alluded to in {{versions}}.

The first 3 bytes of the UUID Long Encoding Block features sub-typing system with two levels of hierarchy.
The first is a "Sub-Variant" abbreviated "sv" which indicates the grouping of UUID Long algorithm types.
The second level of UUID Long sub-typing is defined as simply the "algorithm" which can be abbreviated "a".
The Sub-Variant plus Algorithm (SV+A) serve as the identity behind a particular a UUID Long value.

With this in mind "Sub-Variant 0, Algorithm 4" can be expressed as "sv0a4" or "UUIDsv0a4" throughout this document.

The final 3 bytes of the UUID Long Encoding Block includes a descriptor for the length of the variable-length UUID Long data which can be used by applications in order to understand where a UUID Long value ends.

The full 6 byte UUID Encoding block can be observed in {{ex-ul-block}} or {{ulLayout}} and described succinctly in {{ulDescription}}.

## Sub-Variants {#subvariants}
UUID Long defines four starting sub-variant groupings as defined by {{subVariant}}.

| Sub-Variant ID | Description | 
| sv0 | Experimental/Custom Algorithms | 
| sv1 | Random Based Algorithms | 
| sv2 | Time Based Algorithms |
| sv3 | Hash-based Algorithms | 
| sv4-sv255| Reserved for future algorithm groupings as required |
{: #subVariant title='UUID Long Sub-Variants'}

Future sub-variants in the space (sv4-sv255) can be allocated where a grouping of algorithms is required; but if a current sub-variant is applicable for a new algorithm, the new algorithm should be grouped under a given sub-variant.

The four starting sub-variant groupings mirror the four generic types of UUID algorithms observed in {{RFC9562}}.

# UUID Long Algorithms {#algos}
As mentioned in {{subtypes}}, UUID Long Algorithms are grouped by at the Sub-Variant level. 

UUID Long first maps the {{RFC9562}} versions to algorithms in the appropriate sub-variant algorithm space.
The sub-variant algorithm identifier has been 'smeared' for ease understanding when referencing the old values. 
For example: "UUIDv4 == UUIDsv1a4" and "UUIDv7 == UUIDsv2a7" where the final number in each abbreviation matches.

The first 16 sub-variant algorithm values (a0-a15) in each sub-variant space are reserved for matching the appropriate {{RFC9562}} versions. This ensures that a future IETF spec can define both a UUID Short Version and UUID Long sub-variant algorithm that line up nicely to each other. With 65,536 possible sub-variant algorithms in each of the 256 sub-variant spaces; 16 reserved sub-variant algorithm identifiers should be no problem.

When the time comes that all 16 {{RFC9562}} versions have allocated to their appropriate UUID Long SV+A IDs, or are no longer in need of the mapping space; outstanding sub-variant algorithm identifiers MAY be used by future UUID Long specifications.

Other UUID sub-types that existing in other variant spaces MAY leverage unused sub-variant algorithm identifiers, starting at a16, for UUID Long versions of the existing algorithms.

Generally speaking for sub-variant algorithms based on the RFC9562 versions; there are two main areas that need to be described:

{: spacing="compact"}

- How to leverage the new UUID Long bits.
- Define if the RFC9562 version bit handling.

The following sections illustrates the current sub-variant algorithm mappings for UUID Long along with the methods for generating a UUID Long value for a given sub-variant algorithm.

For all algorithms the following two statements apply, even if they are based on an RFC9562-based algorithm.

{: spacing="compact"}

- The Variant bits are always overwritten to "F" as per {{variant}}.
- The UUID Long Encoding Block is encoded with the sub-variant id, algorithm id and long data length descriptor as per {{ex-ul-block}}.

## Sub-Variant 0 {#sv0-section}

Algorithm Identifiers in this sub-variant space SHOULD be used for custom, experimental or vendor-specific use cases.
UUIDv8 has been mapped to UUIDsv0v8 in this document and is the only current algorithm in this space defined by {{sv0}}.

Vendor's are encouraged to use this space for testing and experimental algorithms before finalization into another sub-variant algorithm identifier.
At which point the Algorithm Identifier in this sub-variant can be released for continued use.

| SV ID | Algorithm ID | Name | 9562 Version (if applicable) | Algorithm Definition Link |
| sv0 | a8 | Custom | UUIDv8 | {{sv0a8-section}} |
{: #sv0 title='Sub-Variant 0 Algorithms'}

### sv0a8 {#sv0a8-section}
sv0a8 is based on UUIDv8 from {{RFC9562, Section 5.8}} with the following deltas:

{: spacing="compact"}

- UUID Long Data can be leveraged as a new "custom_d" field of arbitrary size within the UUID Long data as shown in {{sv0a8}}. The length of this new data is calculated and inserted into the UUID Long Encoding Block.
- The version behavior does not need to remain the same as {{RFC9562, Section 4.2}} and can be set to whatever an implementation desires.

~~~
UUIDsv0a8 Structure {
  custom_a (48),
  9562 Version (4),
  custom_b (12),
  UUID Variant (4) = F,
  custom_c (60),
  UUID Long Encoding Block (48),
  custom_d (8..16777215),
}
~~~
{: #sv0a8 title="Example sv0a8 Bit and Field Layout"}

Note that where possible, for experimental use cases, implementation are encouraged to apply for a sub-variant algorithm for their UUID Long Algorithm.

TODO Link to process section

## Sub-Variant 1 {#sv1-section}
Algorithm Identifiers in this sub-variant space MUST be related to random, pseudorandom, or other similar methods of generating UUID Long values.

UUIDv4 has been mapped to UUIDsv1a4 in this document and is the only current algorithm in this space defined by {{sv1}}.

| SV ID | Algorithm ID | Name | 9562 Version (if applicable) | Algorithm Definition Link |
| sv1 | a4 | Random | UUIDv4 | {{sv1a4-section}} |
{: #sv1 title='Sub-Variant 1 Algorithms'}

### sv1a4 {#sv1a4-section}
sv1a4 is based on UUIDv4 from {{RFC9562, Section 5.4}} with the following deltas:

{: spacing="compact"}

- UUID Long Data can be leveraged as a new "random_d" field of arbitrary size within the UUID Long data as shown in {{sv1a4}}. The length of this new data is calculated and inserted into the UUID Long Encoding Block.
- The version behavior does not need to remain the same as {{RFC9562, Section 4.2}} and these 4 version bits MAY also be randomized.

~~~
UUIDsv1a4 Structure {
  random_a (48),
  9562 Version (4),
  random_b (12),
  UUID Variant (4) = F,
  random_c (60),
  UUID Long Encoding Block (48),
  random_d (8..16777215),
}
~~~
{: #sv1a4 title="Example sv1a4 Bit and Field Layout"}

Examples of UUIDsv1a4 can be seen in {{sv1a4-value}}.

## Sub-Variant 2 {#sv2-section}
Algorithm Identifiers in this sub-variant space MUST be related to UUIDs which feature timestamps.

UUIDv1, UUIDv6 and UUIDv7 have been mapped to UUIDsv2a1, UUIDsv2a6, UUIDsv2a7 where required as per {{sv2}}.

| SV ID | Algorithm ID | Name | 9562 Version (if applicable) | Algorithm Definition Link |
| sv2 | a1 | Gregorian Time-based | UUIDv1 | {{sv2a1-section}} |
| sv2 | a6 | Reordered Gregorian Time-based | UUIDv6 | {{sv2a6-section}} |
| sv2 | a7 | Unix Time-based (MS) | UUIDv7 | {{sv2a7-section}} |
{: #sv2 title='Sub-Variant 2 Algorithms'}

### sv2a1 {#sv2a1-section}
sv2a1 is based on UUIDv1 from {{RFC9562, Section 5.1}} with the following deltas:

{: spacing="compact"}

- UUID Long Data can be leveraged to as an "extended_node" field within the UUID Long data as shown in {{sv2a1}}. The length of this new data is calculated and inserted into the UUID Long Encoding Block.
- The node value MAY feature IEEE 802 MAC address and random data of arbitrary size or be fully randomized using portions of the original node bits and variable-length UUID Long data. 
- The version behavior SHOULD remain the same as {{RFC9562, Section 4.2}}, the version bits MAY also be randomized since this does not effect the sortability of this algorithm.

~~~
UUIDsv2a1 Structure {
  time_low (32),
  time_mid (16),
  9562 Version (4),
  time_high (12),
  UUID Variant (4) = F,
  clock_seq (14),
  node (48),
  UUID Long Encoding Block (48),
  extended_node (8..16777215),
}
~~~
{: #sv2a1 title="Example sv2a1 Bit and Field Layout"}

### sv2a6 {#sv2a6-section}
sv2a6 is based on UUIDv6 from {{RFC9562, Section 5.6}} with the following deltas:

{: spacing="compact"}

- UUID Long Data can be leveraged to as an "extended_node" field within the UUID Long data as shown in {{sv2a6}}. The length of this new data is calculated and inserted into the UUID Long Encoding Block.
- The node value MAY feature IEEE 802 MAC address and random data of arbitrary size or be fully randomized using portions of the original node bits and variable-length UUID Long data. 
- The version behavior MUST remain the same as {{RFC9562, Section 4.2}} to ensures proper sortability which is a key feature of this UUID's algorithm.

~~~
UUIDsv2a6 Structure {
  time_high (32),
  time_mid (16),
  9562 Version (4) = 6,
  time_low (12),
  UUID Variant (4) = F,
  clock_seq (14),
  node (48),
  UUID Long Encoding Block (48),
  extended_node (8..16777215),
}
~~~
{: #sv2a6 title="Example sv2a6 Bit and Field Layout"}

### sv2a7  {#sv2a7-section}
sv2a7 is based on UUIDv7 {{RFC9562, Section 5.7}} with the following deltas:

{: spacing="compact"}

- UUID Long Data can be leveraged to as an "rand_c" field within the UUID Long data as shown in {{sv2a7}}. The length of this new data is calculated and inserted into the UUID Long Encoding Block.
- The version behavior MUST remain the same as {{RFC9562, Section 4.2}} to ensures proper sortability which is a key feature of this UUID's algorithm.

~~~
UUIDsv2a7 Structure {
  unix_ts_ms (48),
  9562 Version (4) = 7,
  rand_a (12),
  UUID Variant (4) = F,
  rand_b (60),
  UUID Long Encoding Block (48),
  rand_c (8..16777215),
}
~~~
{: #sv2a7 title="Example sv2a7 Bit and Field Layout"}

## Sub-Variant 3 {#sv3-section}
Algorithm Identifiers in this sub-variant space MUST be related to hash-based UUIDs computed using "names" and "namespaces" as defined by {{RFC9562, Section 6.5}}
UUIDv5 has been mapped to UUIDsv3a5 while new hashing protocols utilize algorithms a16 through a27. (TODO: CITE NIST Docs)


| SV ID | Algorithm ID | Name | 9562 Version (if applicable) | Algorithm Definition Link |
| sv3 | a5 | SHA-1 | UUIDv5 | {{sv3a5-section}} |
| sv3 | a16 | SHA-224 | | {{nghb-section}} |
| sv3 | a17 | SHA-256 | | {{nghb-section}} |
| sv3 | a18 | SHA-384 | | {{nghb-section}} |
| sv3 | a19 | SHA-512 | | {{nghb-section}} |
| sv3 | a20 | SHA-512/224 | | {{nghb-section}} |
| sv3 | a21 | SHA-512/256 | | {{nghb-section}} |
| sv3 | a22 | SHA3-224 | | {{nghb-section}} |
| sv3 | a23 | SHA3-256 | | {{nghb-section}} |
| sv3 | a24 | SHA3-384 | | {{nghb-section}} |
| sv3 | a25 | SHA3-512 | | {{nghb-section}} |
| sv3 | a26 | SHAKE128 | | {{nghb-section}} |
| sv3 | a27 | SHAKE256 | | {{nghb-section}} |
{: #sv3 title='Sub-Variant 3 Algorithms'}

Note that UUIDv3 has not been mapped to UUIDsv3a3 because the current MD5-based algorithm from {{RFC9562, Section 5.3}} does not have any requirements for bits past 128. Thus there is no need for a UUID Long equivalent of this algorithm.

### sv3a5 {#sv3a5-section}
sv3a5 is based on UUIDv5 from {{RFC9562, Section 5.5}} with the following deltas:

{: spacing="compact"}

- The original algorithm requires that parts of the SHA-1 hash be truncated to fit the 128 bit layout however with UUID Long these extra bits can be embedded into the UUID Long Data as "sha1_discard" seen in {{sv3a5}}. The length of this discarded data is calculated and inserted into the UUID Long Encoding Block.
- The version MUST NOT remain the same as {{RFC9562, Section 4.2}}. As a result the bits that would have been overwritten to a hard coded "5" are now left as the original portions of the hash.

~~~
UUIDsv3a5 Structure {
  sha1_high (48),
  9562 Version (4),
  sha1_mid (12),
  UUID Variant (4) = F,
  sha1_low (60),
  UUID Long Encoding Block (48),
  sha1_discard (8..16777215),
}
~~~
{: #sv3a5 title="Example sv3a5 Bit and Field Layout"}

An Example of UUIDsv3a5 can be seen in {{sv3a5-value}}.

### sv3a16 - sv3a23 {#nghb-section}
sv3a16 - sv3a23 describe Name-Based UUID generation using new hashing algorithms.
From an operational standpoint the same fields are described for all of these algorithms. This is shown in {{ulhb}}.

The algorithm and creation of these UUID Long values is the same as {{RFC9562, Section 5.5}} with the following deltas:

{: spacing="compact"}

- The desired hash algorithm is used in place of SHA-1.
- The 9562 Version is not used and those 4 bits retain their value from the hash.
- The bits beyond 128 are placed in "hash_low" with the length calculated and inserted into the UUID Long Encoding Block.

~~~
UUID Long Hash-Based Structure {
  hash_high (64),
  UUID Variant (4) = F,
  hash_middle (60),
  UUID Long Encoding Block (48),
  hash_low (8..16777215),
}
~~~
{: #ulhb title="Example UUID Long Hash-Based Bit and Field Layout"}

Example of UUIDsv3a17, using SHA-256, can be seen in {{sv3a17-value}}.

# Compatibility with 128 Bit UUIDs {#compatibility}
Since the first 128 bits are a valid UUID Short, if some device does not understand UUID Long they can read the first 128 bit and still gleam a valid 128 bit UUID value.
Though some system may have a problem reading or accepting the F Variant; this approach ensures that a given UUID Long value can be easily transposed into a smaller value where required.

Note that the version bit-space is not a requirement in UUID Long thus some UUID sub-variant algorithms may have varying data at this position.
The bits still exist so for systems that do not read the variant bit first, they may see inconsistent results if trying to read only the version or version and then variant.

# Security Considerations {#security}

UUID Long shared many of the same security considerations as {{RFC9562}}.
The main security consideration with UUID Long is the maximum length of data and possible buffer overflows which lead to other vulnerabilities.
Implementations that only expect 128 bit UUIDs should not read beyond 128 bits. 

Implementations that plan to work with UUID Long values should use the UUID Long Data Descriptor field within the UUID Long Encoding block to gleam the total length of the UUID Long Data Field. Implementations should also program safeguards as to not read more data than is available in memory. For example, setting an arbitrary maximum on the amount of UUID Long data that is parsed.

By default the UUID Long value (and UUID short) do not feature any hash/signature method. An attacker could modify the UUID Long Data Length Descriptor bits and include new data in an attempt to force some buffer overflow condition or append data that was not part of the original algorithm.
An algorithm MAY choose create a hash/digital signature on the final UUID Long value and provide this hash to a peer in order to provide some levels data integrity.
Further, where possible introspection into the UUID is discouraged as per {{RFC9562, Section 6.12}}.

# IANA Considerations {#iana}

TODO: IANA when things are finalized. Things like add sub-variant algorithms to sub-types section of UUID registry.

--- back

# Changelog {#changelog}
{:removeinrfc}

draft-00:
: * Initial Release

# Test Vectors {#test-vectors}

Due to the variable length nature of the UUID Long Data field there could be an infinite number of test vectors.
The sections below attempt to summarize the key points of the sub-variant algorithms as described by the body of this document.

## Example sv1a4 values {#sv1a4-value}

| DOC     | Type    | Random Bits   | Example |
| RFC9562 | UUIDv4  | 122           | 73e94fe0-e951-4153-aaf3-50e4e6089d9d |
| DRAFT	  | sv1a4   | 160           | 81783312-54db-bef3-f722-2515c8f3aceb-010004000022-2072a4623 |
| DRAFT	  | sv1a4   | 192           | b6debb20-db1e-cdc7-f65f-c266fd5e25e9-010004000044-b1150e08ab9e81a11 |
| DRAFT	  | sv1a4   | 256           | d0fda74d-59a7-76c9-fd30-587ea76c99e9-010004000083-63f93f97b7b47cc5c853ce09a87ded629 |

## Example sv3a5 Value {#sv3a5-value}

~~~
Namespace (DNS):  6ba7b810-9dad-11d1-80b4-00c04fd430c8
Name:             www.example.com
----------------------------------------------------------
SHA-1: 2ed6657de927468b55e12665a8aea6a22dee3e35
~~~

~~~
A: 2ed6657d-e927-468b-55e1-2665a8aea6a2-2dee3e35
B: xxxxxxxx-xxxx-xxxx-Fxxx-xxxxxxxxxxxx
C: 2ed6657d-e927-468b-f5e1-2665a8aea6a2
D:                                     -2dee3e35
E: 2ed6657d-e927-468b-f5e1-2665a8aea6a2-03000500001e
F: 2ed6657d-e927-468b-f5e1-2665a8aea6a2-03000500001e-2dee3e35
~~~

{: spacing="compact"}

- Line A details the full SHA-1 as a hexadecimal value with the dashes inserted.
- Line B details the F variant hexadecimal positions, which must be overwritten.
- Line C details the final value after the variant has been overwritten.
- Line D details the leftover values from the original SHA-1 computation (Note that these have a length of 30 bits)
- Line E details adding the UUID Long encoding block of Sub-Variant 3, and algorithm 5 (RFC 9562's version 5), and long data length of 30 bits as hex with leading 0s included.
- Line F details the leftover values appended to form the full UUID Long of form sv3a5.

## Example sv3a17 Value {#sv3a17-value}
~~~
Namespace (DNS): 6ba7b810-9dad-11d1-80b4-00c04fd430c8
Name:            www.example.com
----------------------------------------------------------------
SHA-256: 5c146b143c524afd938a375d0df1fbf6fe12a66b645f72f6158759387e51f3c8
~~~

~~~
A: 5c146b14-3c52-4afd-938a-375d0df1fbf6-fe12a66b645f72f6158759387e51f3c8
B: xxxxxxxx-xxxx-xxxx-Fxxx-xxxxxxxxxxxx
C: 5c146b14-3c52-4afd-f38a-375d0df1fbf6
D:                                     -fe12a66b645f72f6158759387e51f3c8
E: 5c146b14-3c52-4afd-f38a-375d0df1fbf6-030011000080
F: 5c146b14-3c52-4afd-f38a-375d0df1fbf6-030011000080-fe12a66b645f72f6158759387e51f3c8
~~~

{: spacing="compact"}

- Line A details the full SHA-256 as a hexadecimal value with the dashes inserted.
- Line B details the F variant hexadecimal positions, which must be overwritten.
- Line C details the final value after the variant has been overwritten.
- Line D details the leftover values from the original SHA-256 computation (Note that these have a length of 128 bits)
- Line E details adding the UUID Long encoding block of Sub-Variant 3, and algorithm 7 (SHA-256), and long data length of 128 bits as hex with leading 0s included.
- Line F details the leftover values appended to form the full UUID Long of form sv3a17.
