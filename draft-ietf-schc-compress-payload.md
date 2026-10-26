---
v: 3

title: "SCHC Payload Compression for Structured Formats"
abbrev: SCHC-payload
docname: draft-ietf-schc-compress-payload-latest

category: std
stream: IETF
area: "Internet"
workgroup: "Static Context Header Compression"

venue:
  mail: "schc@ietf.org"

author:
- name: Lorenzo Corneo
  org: Ericsson
  country: Finland
  email: lorenzo.corneo@ericsson.com

- name: Edgar Ramos
  org: Ericsson
  country: Finland
  email: edgar.ramos@ericsson.com

- name: Jaime Jimenez
  org: Ericsson
  country: Finland
  email: jaime.jimenez@ericsson.com

normative:
  RFC8724:
  RFC8126:

informative:
  RFC8259:

entity:
  SELF: "[RFC-XXXX]"

--- abstract

This document describes techniques to adapt the SCHC framework (RFC8724), used for header compression, to also compress and decompress payload of specific protocols. To this end, this document defines a new matching operator, `equal-template`, to check equality of field values with respect to a user-defined template, and a `payload` keyword to be used as Field IDentifier (FID) to signal the presence of payload to be compressed or decompressed. Additionally, this document defines a set of template functions and variables to optimize user-defined templates, which can be extended through the IANA registry defined herein.

--- middle

# Introduction {#introduction}

The main purpose of this document is to extend the SCHC framework {{RFC8724}} to compress and decompress structured data payloads, e.g., JSON {{RFC8259}}, reusing the same compression and decompression machinery originally designed for protocol headers.

This document defines the `equal-template` Matching Operator (MO), which enables a SCHC compressor to match a field value against a user-defined template and extract only the variable parts as compression residue. Templates may include positional variables and functions to address common patterns in structured data.

Additionally, this document defines the `payload` keyword for use in the Field IDentifier (FID) to signal that payload compression is being performed. The keyword can be used with a template, where the entire payload structure is captured in the Target Value (TV), or without a template, where individual structured data fields are specified in separate rule entries.

## Terminology {#terminology}

{::boilerplate bcp14-tagged-bcp14}

### Glossary
- Template-based compression: Payload compression utilizing a pre-shared model that only requires the values that are changing in the payload.
- Template variable: A template variable can be used as a placeholder for variable fields in a template definition. Template variables can carry semantics and other information, e.g., the position and size of a field value in the compression residue (see {{positional}}).
- Template function: A template function can be used to perform actions on a template definition, e.g., compress repeated structures in the payload (see {{repeat}}).
- Payload keyword: The keyword `payload` used in the FID to signal that payload compression is being performed.

# A Matching Operator Providing Equality over Templates {#equal-template}

This section introduces the `equal-template` Matching Operator (MO), which evaluates equality between a field value and a provided template specified in the Target Value (TV).

## Matching Procedure

The `equal-template` MO MUST perform the following steps:

1. Parse the template specified in the TV to identify the fixed portions and the positions of template variables (see {{positional}}) and functions (see {{repeat}}).

2. Compare the fixed portions of the template against the corresponding portions of the field value. The comparison MUST be performed as a byte-wise exact match on the textual representation of the field value.

3. Extract the substrings of the field value that correspond to each template variable. Each extracted substring represents a value to be included in the compression residue.

4. If all fixed portions match, the MO evaluates to true and the extracted values are passed to the Compression/Decompression Action (CDA). If any fixed portion does not match, the MO evaluates to false and the rule MUST NOT be applied.

## Residue Encoding {#residue-encoding}

<!-- LC: This section assumes prior knowledge on positional template variables, which are not yet defined. Consider if the matching operator should be defined after template variables and functions. -->

When the `equal-template` MO evaluates to true, the extracted values MUST be encoded in the compression residue in the order determined by their positional identifiers (i.e., `$1` before `$2`, and so on).

Each positional template variable MUST specify the length of the value it captures, expressed in bytes, using the notation `$n:length`. For example, `$1:10` indicates that the first positional variable captures a value occupying exactly 10 bytes in the residue. If the actual extracted value is shorter than the declared length, it MUST be right-padded with zeros to fill the declared length.

Upon decompression, the decompressor MUST strip trailing zero bytes from each extracted value before reconstructing the payload. This is safe because the `equal-template` MO operates on textual representations where the null byte (0x00) is not a valid character.

The decompressor uses this fixed length to unambiguously parse the residue. This approach keeps the length information co-located with the template variable it describes and does not require changes to the SCHC rule structure defined in {{RFC8724}}.

Alternatively, the Field Length (FL) field of the rule could be extended to carry a list of lengths, one per positional variable. Such an approach would require additional standardization effort and is left for future discussion within the Working Group.

## Type Handling

The `equal-template` MO operates on the textual (serialized) representation of the field value. Template variables capture substrings as-is, preserving the original serialization. For example, given the template `{"value": $1:4}` and the field value `{"value": -11}`, the extracted value for `$1` is the string `-11` right-padded with one zero byte to fill 4 bytes. Type interpretation is left to the application layer after decompression.

## Match Failure

If the `equal-template` MO evaluates to false for a given rule, the SCHC compressor MUST proceed to evaluate the next rule in the rule set, as specified in Section 7.2 of {{RFC8724}}. If no rule matches, the packet MUST be sent uncompressed.

# Template Variables and Functions

This section specifies one variable and one function, to be used in templates, that address common patterns in structured data.

<!-- LC: Using $ as special sign to identify template variables may not be the best idea, it can be confused with JSON pointers or just regular payload. Should we consider a better alternative? -->

Template variables and functions are identified through a special character, hereby defined as the dollar sign `$`.

It is possible to define additional template variables and functions in the IANA registry defined in {{iana-reg}}.

## Positional Template Variable {#positional}

The positional template variable, `$n:length`, uses a numerical identifier, `n`, to represent the position of the field value in the compression residue, and a `length` to specify the fixed size in bytes of the captured value (see {{residue-encoding}}). That is, `$1:10` represents the first field value in the residue with a fixed length of 10 bytes, while `$2:4` represents the second one with 4 bytes.

For example, a target value (TV) using two positional template variables may look as follows:

~~~
{"id": $1:10, "value": $2:4}
~~~

Considering the above example, the MO `equal-template` will match the following payload:

~~~
{"id": "abc123", "value": -11}
~~~

As a result, the values `"abc123"` and `-11` will be encoded in the residue during compression phase, while the rest of the template will be elided. Note that since the matching is byte-wise (see {{equal-template}}), the positional variable captures exactly the substring between the surrounding fixed portions. In this example, `$1` captures `"abc123"` including the JSON quotes (8 bytes), while `$2` captures `-11` without quotes (3 bytes).

## Repeat Template Function {#repeat}

The repeat template function, `$repeat(template, separator)`, is used to repeat multiple times the template passed as argument. The function requires two arguments as input, namely, the `template` to repeat, and the `separator` between template repetitions. The example below is indicative of a common pattern found in structured data payloads.

~~~
[{"id": "abc123", "value": -11}, {"id": "abc123", "value": -13}]
~~~

Using the repeat template function, `$repeat()`, the above example may look as follows:

~~~
[$repeat('{"id": $1:10, "value": $2:4}', ", ")]
~~~

As noted above, it is possible to use the repeat template function, `$repeat()`, in conjunction with the positional template variable, `$n`.

### Repetition Count

The number of repetitions is not known a-priori from the template and varies per payload. The compressor MUST infer the number of repetitions by matching the template against the field value, and MUST encode this count as the first byte of the residue for the `$repeat()` block, as an unsigned 8-bit integer. The decompressor MUST read this count first, then extract the positional variable values for each repetition in sequence.

For the example above with two repetitions, the residue is structured as follows:

<!-- LC: Should we include a template function (or variable?) identifier to simplify matching operations in decompression phase?? Currently, the decompressor has to guess what residue is. With an ID it would be able to understand immediately what the residue is about, and may immediately fail the MO. -->

~~~
| Count (1 byte) | $1 rep.1 | $2 rep.1 | $1 rep.2 | $2 rep.2 |
| 0x02           | 10 bytes | 4 bytes  | 10 bytes | 4 bytes  |
~~~

Since each positional variable has a fixed length declared via the `$n:length` notation (see {{residue-encoding}}), the decompressor knows that each repetition contributes a fixed number of bytes to the residue. In the example above, each repetition contributes 14 bytes (10 + 4).

A repetition count of zero is valid. In this case, the residue for the `$repeat()` block contains only the count byte (0x00), and the decompressor MUST reconstruct an empty sequence, retaining only the surrounding fixed portions of the template. However, a repetition count of zero is discouraged as it likely indicates a misconfigured rule or an unexpected payload structure.

Since the repetition count is encoded as an unsigned 8-bit integer, the maximum number of repetitions is 255. This is expected to be sufficient for constrained IoT use cases. Whether a larger count field is needed is left for discussion within the Working Group.

### Positional Variable Scope

Positional variable identifiers (`$n`) are scoped to a single repetition. That is, `$1` and `$2` refer to the first and second extracted values within each repetition, not across the entire payload. The residue contains the values for all positional variables of the first repetition, followed by those of the second repetition, and so on.

### Optimization Example

The level of payload compression that can be achieved greatly varies depending on the use case. For example, if it is known a-priori that the value of the field `id` never changes, the above payload example could be further optimized as follows:

~~~
[$repeat('{"id": "abc123", "value": $1:4}', ", ")]
~~~

The above template allows the compressor to avoid adding the value of the field `id` to the residue, unlike in the previous example.

# A Keyword for Structured Data Payload Compression

This section defines the keyword `payload` that MUST be used whenever payload compression is needed. In such cases, the keyword `payload` MUST be specified in the Field IDentifier (FID).

When payload compression is performed by individually specifying one structured field at a time, the FID MUST include the keyword `payload` as prefix, followed by a colon `:` and the name of the structured data field, e.g., `payload:name` for the structured data field called `name`. An extended FID notation that includes additional context such as media type and field index is illustrated in {{no-template-example}}. Whether such extended notation should be normative is left for discussion within the Working Group.

When payload compression is provided through a template, thus all the structured data fields are included in the TV, the FID MUST include only the keyword `payload`.

# An Example of Structured Data Payload with Template

This section demonstrates the use of the `equal-template` matching operator (MO) in conjunction with payload compression templates and the `payload` keyword.

Considering the example below

~~~
[{"id": "abc123", "value": -11}, {"id": "abc123", "value": -13}]
~~~

a SCHC rule (with only relevant fields) MAY be defined using the new `payload` keyword, the `equal-template` MO, and the payload compression templates (functions and variables). Such a rule may look as follows:

<!-- LC: Assuming CDA `send` because there MUST be some residual to be sent from the template -->

~~~
FID      TV          MO              CDA
payload  <template>  equal-template  send
~~~

where the `<template>` is omitted from the above for brevity, but is defined as follows:

~~~
[$repeat('{"id": "abc123", "value": $1:4}', ", ")]
~~~

When a SCHC node computes the rule above, the `payload` keyword in the FID signals the node that application data needs to be compressed. Specifically, the `equal-template` MO triggers the SCHC node to check for equality between the current field value and the template provided in the TV. Upon matching, the residue is sent (or retrieved) over the network. For the payload above, the residue contains the repetition count (0x02, 1 byte) followed by the values `-11` and `-13`, each padded to 4 bytes.

Please notice that the algorithm used to perform template matching is implementation-specific, provided that the result is equivalent to the matching procedure defined in {{equal-template}}.

# An Example of Structured Data Payload without Template {#no-template-example}

<!-- LC: Mention that how the payload keyword is interpreted by a SCHC node is not normative and left to implementation -->

This section demonstrates how to make use of payload compression without a template as defined in this document. The example below will be the target of the payload compression.

~~~
[{"id": "abc123", "value": -11}, {"id": "abc123", "value": -13}]
~~~

The snippet below shows how a SCHC rule (only relevant fields are used) MAY be defined using the new `payload` keyword.

<!-- LC: We need a function that tells the TV to read the value? Probably not as decompressor reads the residual anyway -->

~~~
FID                               TV      MO      CDA
payload:application/json:1:id     abc123  equal   elide
payload:application/json:1:value          ignore  send
payload:application/json:2:id     abc123  equal   elide
payload:application/json:2:value          ignore  send
~~~

The FIDs in the rule above carry semantics that can be used by the SCHC node to perform successful compression and decompression. Considering the first field of the above rule, `application/json` signals the decompressor which media type is being reconstructed. The subsequent numerical identifier, `1`, represents the object the compressed field belongs to. Finally, the last value encoded in the FID is the name of the JSON object, `id`. Moreover, the target value (TV) corresponds to the value of the structured data field, and not to the name.

In the second field of the above rule, the TV is omitted as the `send` CDA implies that the decompressor reads the received residue. However, in this case, the decompressor knows that the residue must be assigned to the JSON field specified in the FID, `value` in this case.

Please notice that the semantics of the FID described above are not normative. Semantic-specific details of the FID are left to implementations as this document only mandates the `payload` keyword.


# Security Considerations

The security considerations of the SCHC framework {{RFC8724}} apply to this document.

<!-- Template definitions and rules are assumed to be pre-shared between the compressor and decompressor through a secure provisioning mechanism. If an attacker can modify the rules or templates, it could cause the decompressor to reconstruct payloads that differ from the original, leading to data integrity issues. Implementations MUST ensure that rule provisioning is performed over authenticated and integrity-protected channels. -->

<!-- Compressed payloads are smaller than their original form, but decompression expands them back to full size. An attacker that can craft compressed data with a high repetition count in a `$repeat()` block could cause the decompressor to allocate disproportionate resources. Implementations SHOULD enforce limits on the maximum number of repetitions and on the maximum size of decompressed output. -->

This document does not define any encryption mechanism. If payload confidentiality is required, it MUST be provided by a lower layer or by encrypting the payload before compression.

# IANA Considerations {#iana}

This section provides guidance to the Internet Assigned Numbers Authority (IANA) regarding registration of values related to this document, in accordance with {{RFC8126}}.

## SCHC Payload Compression Template Functions and Variables {#iana-reg}

IANA is requested to create a new registry called "SCHC Payload Compression Template Functions and Variables".

The registration policy for this registry is "Specification Required" as defined in Section 4.6 of {{RFC8126}}.

The registry MUST include the following fields:
- Name of function or variable
- Type (`function` or `variable`)
- Description
- Reference of the specification describing the function or variable

The designated expert(s) SHOULD verify that the template function, or variable, name is appropriate and not likely to cause confusion with existing entries.

The registrant of an existing entry may request updates to that entry, subject to the same expert review. They should verify that updates preserve backward compatibility with deployed implementations, or if breaking changes are necessary, consider whether a new registry entry is more appropriate.

The following template functions and variables are described in this document:

| Function/Variable Name |  Type    | Description                  | Reference                     |
|------------------------+----------+------------------------------+-------------------------------|
| `n`                    | variable | Positional Template Variable | This document, {{positional}} |
| `repeat`               | function | Repeat Template Function     | This document, {{repeat}}     |
{: #protmap-reg title="SCHC Payload Compression Template Functions and Variables Registry"}

--- back

# Acknowledgements
A great token of appreciation to Rajat Kandoi and Ari Keränen for their support in the inception and review of this draft. Their ideas and thinking are also reflected in the text of the draft.

# Preliminary ABNF for Template Syntax {#abnf}

This appendix provides a preliminary ABNF (RFC 5234) grammar for the template syntax defined in this document. This grammar is work in progress and is subject to discussion and agreement within the SCHC Working Group.

~~~abnf
template       = *( fixed-text / variable / function )

fixed-text     = 1*( %x00-23 / %x25-FF )  ; any character except "$"

variable       = "$" pos-id ":" length
pos-id         = 1*DIGIT
length         = 1*DIGIT

function       = "$" func-name "(" func-args ")"
func-name      = 1*ALPHA
func-args      = func-arg *( "," SP func-arg )
func-arg       = quoted-string / template

quoted-string  = "'" *( %x00-26 / %x28-FF ) "'"
~~~
