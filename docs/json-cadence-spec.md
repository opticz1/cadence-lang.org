---
title: JSON-Cadence Data Interchange Format
sidebar_label: JSON-Cadence Format
sidebar_position: 12
---

## Version

> Version 0.3.1

JSON-Cadence is a data interchange format used to represent Cadence values as language-independent JSON objects.

This format includes less type information than a complete [ABI](https://en.wikipedia.org/wiki/Application_binary_interface), and instead promotes the following tenets:

- **Human-readability** - JSON-Cadence is easy to read and comprehend, which speeds up development and debugging.
- **Compatibility** - JSON is a common format with built-in support in most high-level programming languages, making it easy to parse on a variety of platforms.
- **Portability** - JSON-Cadence is self-describing and thus can be transported and decoded without accompanying type definitions (i.e. an ABI).

# Values

---

## Void

```json
{
  "type": "Void"
}
```

### Example

```json
{
  "type": "Void"
}
```

---

## Optional

```json
{
  "type": "Optional",
  "value": null | <value>
}
```

### Example

```json
// Non-nil

{
  "type": "Optional",
  "value": {
    "type": "UInt8",
    "value": "123"
  }
}

// Nil

{
  "type": "Optional",
  "value": null
}
```

---

## Bool

```json
{
  "type": "Bool",
  "value": true | false
}
```

### Example

```json
{
  "type": "Bool",
  "value": true
}
```

---

## String

```json
{
  "type": "String",
  "value": "..."
}
```

### Example

```json
{
  "type": "String",
  "value": "Hello, world!"
}
```

---

## Address

```json
{
  "type": "Address",
  "value": "0x0" // as hex-encoded string with 0x prefix
}
```

### Example

```json
{
  "type": "Address",
  "value": "0x1234"
}
```

---

## Integers

`[U]Int`, `[U]Int8`, `[U]Int16`, `[U]Int32`,`[U]Int64`,`[U]Int128`, `[U]Int256`, `Word8`, `Word16`, `Word32`, `Word64`, `Word128` or `Word256`

Although JSON supports integer literals up to 64 bits, all integer types are encoded as strings for consistency.

While the static type is not strictly required for decoding, it is provided to inform client of potential range.

```json
{
  "type": "<type>",
  "value": "<decimal string representation of integer>"
}
```

### Example

```json
{
  "type": "UInt8",
  "value": "123"
}
```

---

## Fixed Point Numbers

`[U]Fix64`

Although fixed point numbers are implemented as integers, JSON-Cadence uses a decimal string representation for readability.

```json
{
  "type": "[U]Fix64",
  "value": "<integer>.<fractional>"
}
```

### Example

```json
{
  "type": "Fix64",
  "value": "12.3"
}
```

---

## Array

```json
{
  "type": "Array",
  "value": [
    <value at index 0>,
    <value at index 1>
    // ...
  ]
}
```

### Example

```json
{
  "type": "Array",
  "value": [
    {
      "type": "Int16",
      "value": "123"
    },
    {
      "type": "String",
      "value": "test"
    },
    {
      "type": "Bool",
      "value": true
    }
  ]
}
```

---

## Dictionary

Dictionaries are encoded as a list of key-value pairs to preserve the deterministic ordering implemented by Cadence.

```json
{
  "type": "Dictionary",
  "value": [
    {
      "key": "<key>",
      "value": <value>
    },
    ...
  ]
}
```

### Example

```json
{
  "type": "Dictionary",
  "value": [
    {
      "key": {
        "type": "UInt8",
        "value": "123"
      },
      "value": {
        "type": "String",
        "value": "test"
      }
    }
  ]
  // ...
}
```

---

## Composites (Struct, Resource, Event, Contract, Enum)

Composite fields are encoded as a list of name-value pairs in the order in which they appear in the composite type declaration.

```json
{
  "type": "Struct" | "Resource" | "Event" | "Contract" | "Enum",
  "value": {
    "id": "<fully qualified type identifier>",
    "fields": [
      {
        "name": "<field name>",
        "value": <field value>
      },
      // ...
    ]
  }
}
```

### Example

```json
{
  "type": "Resource",
  "value": {
    "id": "0x3.GreatContract.GreatNFT",
    "fields": [
      {
        "name": "power",
        "value": { "type": "Int", "value": "1" }
      }
    ]
  }
}
```

---

## Path

```json
{
  "type": "Path",
  "value": {
    "domain": "storage" | "private" | "public",
    "identifier": "..."
  }
}
```

### Example

```json
{
  "type": "Path",
  "value": {
    "domain": "storage",
    "identifier": "flowTokenVault"
  }
}
```

---

## Type Value

```json
{
  "type": "Type",
  "value": {
    "staticType": <type>
  }
}
```

### Example

```json
{
  "type": "Type",
  "value": {
    "staticType": {
      "kind": "Int"
    }
  }
}
```

---

## InclusiveRange

```json
{
  "type": "InclusiveRange",
  "value": {
    "start": <start_value>,
    "end": <end_value>,
    "step": <step_value>
  }
}
```

### Example

```json
{
  "type": "InclusiveRange",
  "value": {
    "start": {
      "type": "Int256",
      "value": "10"
    },
    "end": {
      "type": "Int256",
      "value": "20"
    },
    "step": {
      "type": "Int256",
      "value": "5"
    }
  }
}
```

---

## Capability

```json
{
  "type": "Capability",
  "value": {
    "id": <Number>,
    "address": "0x0",  // as hex-encoded string with 0x prefix
    "borrowType": <type>,
  }
}
```

### Example

```json
{
  "type": "Capability",
  "value": {
    "id": "1",
    "address": "0x1",
    "borrowType": {
      "kind": "Int"
    }
  }
}
```

---

## Functions

```json
{
  "type": "Function",
  "value": {
    "functionType": <type>
  }
}
```

Function values can only be exported, they cannot be imported.

### Example

```json
{
  "type": "Function",
  "value": {
    "functionType": {
      "kind": "Function",
      "typeID": "fun():Void",
      "parameters": [],
      "return": {
        "kind": "Void"
      }
    }
  }
}
```

---

# Types

## Simple Types

These are basic types like `Int`, `String`, or `StoragePath`.

```json
{
  "kind": <kind>
}
```

Where `kind` is one of:

- `Account`
- `AccountCapabilityController`
- `AccountKey`
- `Address`
- `AnyResource`
- `AnyResourceAttachment`
- `AnyStruct`
- `AnyStructAttachment`
- `Block`
- `Bool`
- `Capability`
- `CapabilityPath`
- `Character`
- `DeployedContract`
- `DeploymentResult`
- `Fix64`
- `FixedPoint`
- `FixedSizeUnsignedInteger`
- `HashAlgorithm`
- `HashableStruct`
- `Int`
- `Int128`
- `Int16`
- `Int256`
- `Int32`
- `Int64`
- `Int8`
- `Integer`
- `Never`
- `Number`
- `Path`
- `PrivatePath`
- `PublicKey`
- `PublicPath`
- `SignatureAlgorithm`
- `SignedFixedPoint`
- `SignedInteger`
- `SignedNumber`
- `StorageCapabilityController`
- `StoragePath`
- `String`
- `Type`
- `UFix64`
- `UInt`
- `UInt128`
- `UInt16`
- `UInt256`
- `UInt32`
- `UInt64`
- `UInt8`
- `Void`
- `Word128`
- `Word16`
- `Word256`
- `Word32`
- `Word64`
- `Word8`

### Example

```json
{
  "kind": "UInt8"
}
```

---

## Optional Types

```json
{
  "kind": "Optional",
  "type": <type>
}
```

### Example

```json
{
  "kind": "Optional",
  "type": {
    "kind": "String"
  }
}
```

---

## Variable Sized Array Types

```json
{
  "kind": "VariableSizedArray",
  "type": <type>
}
```

### Example

```json
{
  "kind": "VariableSizedArray",
  "type": {
    "kind": "String"
  }
}
```

---

## Constant Sized Array Types

```json
{
  "kind": "ConstantSizedArray",
  "type": <type>,
  "size": <length of array>,
}
```

### Example

```json
{
  "kind": "ConstantSizedArray",
  "type": {
    "kind": "String"
  },
  "size": 3
}
```

---

## Dictionary Types

```json
{
  "kind": "Dictionary",
  "key": <type>,
  "value": <type>
}
```

### Example

```json
{
  "kind": "Dictionary",
  "key": {
    "kind": "String"
  },
  "value": {
    "kind": "UInt16"
  }
}
```

---

## Composite Types

```json
{
  "kind": "Struct" | "Resource" | "Event" | "Contract" | "StructInterface" | "ResourceInterface" | "ContractInterface",
  "type": "", // this field exists only to keep parity with the enum structure below; the value must be the empty string
  "typeID": "<fully qualified type ID>",
  "initializers": [
    <initializer at index 0>,
    <initializer at index 1>
    // ...
  ],
  "fields": [
    <field at index 0>,
    <field at index 1>
    // ...
  ],
}
```

### Example

```json
{
  "kind": "Resource",
  "type": "",
  "typeID": "0x3.GreatContract.GreatNFT",
  "initializers": [
    [
      {
        "label": "foo",
        "id": "bar",
        "type": {
          "kind": "String"
        }
      }
    ]
  ],
  "fields": [
    {
      "id": "foo",
      "type": {
        "kind": "String"
      }
    }
  ]
}
```

---

## Field Types

```json
{
  "id": "<name of field>",
  "type": <type>
}
```

### Example

```json
{
  "id": "foo",
  "type": {
    "kind": "String"
  }
}
```

---

## Parameter Types

```json
{
  "label": "<label>",
  "id": "<identifier>",
  "type": <type>
}
```

### Example

```json
{
  "label": "foo",
  "id": "bar",
  "type": {
    "kind": "String"
  }
}
```

---

## Initializer Types

Initializer types are encoded a list of parameters to the initializer.

```json
[
  <parameter at index 0>,
  <parameter at index 1>,
  // ...
]
```

### Example

```json
[
  {
    "label": "foo",
    "id": "bar",
    "type": {
      "kind": "String"
    }
  }
]
```

---

## Function Types

```json
{
  "kind": "Function",
  "typeID": "<function name>",
  "parameters": [
    <parameter at index 0>,
    <parameter at index 1>,
    // ...
  ],
  "purity: "view" | undefined,
  "return": <type>
}
```

### Example

```json
{
  "kind": "Function",
  "typeID": "foo",
  "parameters": [
    {
      "label": "foo",
      "id": "bar",
      "type": {
        "kind": "String"
      }
    }
  ],
  "purity": "view",
  "return": {
    "kind": "String"
  }
}
```

---

## Reference Types

```json
{
  "kind": "Reference",
  "authorization": {
    "kind": "Unauthorized" | "EntitlementMapAuthorization" | "EntitlementConjunctionSet" | "EntitlementDisjunctionSet",
    "entitlements": [
    <entitlement at index 0>,
    <entitlement at index 1>
    // ...
    ]
    },
  "type": <type>
}
```

### Example

```json
{
  "kind": "Reference",
  "authorization": {
    {
      "kind": "EntitlementMapAuthorization",
      "entitlements": [
        {
          "kind": "EntitlementMap",
          "typeID": "foo"
        }
      ]
    }
  },
  "type": {
    "kind": "String"
  }
}
```

---

## Intersection Types

```json
{
  "kind": "Intersection",
  "typeID": "<fully qualified type ID>",
  "types": [
    <type at index 0>,
    <type at index 1>,
    //...
  ]
}
```

### Example

```json
{
  "kind": "Intersection",
  "typeID": "{0x1.FungibleToken.Receiver}",
  "types": [
    {
      "kind": "ResourceInterface",
      "typeID": "0x1.FungibleToken.Receiver",
      "fields": [
        {
          "id": "uuid",
          "type": {
            "kind": "UInt64"
          }
        }
      ],
      "initializers": [],
      "type": ""
    }
  ]
}
```

---

## Capability Types

```json
{
  "kind": "Capability",
  "type": <type>
}
```

### Example

```json
{
  "kind": "Capability",
  "type": {
    "kind": "Reference",
    "authorization": {
      "kind": "Unauthorized",
      "entitlements": null
    },
    "type": {
      "kind": "String"
    }
  }
}
```

---

## Enum Types

```json
{
  "kind": "Enum",
  "type": <type>,
  "typeID": "<fully qualified type ID>",
  "initializers":[],
  "fields": [
    {
      "id": "rawValue",
      "type": <type>
    }
  ]
}
```

### Example

```json
{
  "kind": "Enum",
  "type": {
    "kind": "String"
  },
  "typeID": "0x3.GreatContract.GreatEnum",
  "initializers": [],
  "fields": [
    {
      "id": "rawValue",
      "type": {
        "kind": "String"
      }
    }
  ]
}
```

## Repeated Types

When a composite type appears more than once within the same JSON type encoding, either because it is
recursive or because it is repeated (e.g. in a composite field), the composite is instead
represented by its type ID.

### Example

```json
{
  "type": "Type",
  "value": {
    "staticType": {
      "kind": "Resource",
      "typeID": "0x3.GreatContract.NFT",
      "fields": [
        {
          "id": "foo",
          "type": {
            "kind": "Optional",
            "type": "0x3.GreatContract.NFT" // recursive NFT resource type is instead encoded as an ID
          }
        }
      ],
      "initializers": [],
      "type": ""
    }
  }
}
```

## Inclusive Range Type

```json
{
  "kind": "InclusiveRange",
  "element":  <integer_type>
}
```

### Example

```json
{
  "kind": "InclusiveRange",
  "element": {
    "kind": "Int"
  }
}
```
