---
title: Run-time Types
sidebar_position: 6
---

Types can be represented at run-time. To create a type value, use the constructor function `Type<T>()`, which accepts the static type as a type argument.

This is similar to `T.self` in Swift, `T::class`/`KClass<T>` in Kotlin, and `T.class`/`Class<T>` in Java.

For example, to represent the type `Int` at run-time:

```cadence
let intType: Type = Type<Int>()
```

This works for both built-in and user-defined types. For example, to get the type value for a resource:

```cadence
resource Collectible {}

let collectibleType = Type<@Collectible>()

// `collectibleType` has type `Type`
```

Type values are comparable.

```cadence

Type<Int>() == Type<Int>()

Type<Int>() != Type<String>()
```

The method `view fun isSubtype(of: Type): Bool` can be used to compare the run-time types of values.

```cadence
Type<Int>().isSubtype(of: Type<Int>()) // true

Type<Int>().isSubtype(of: Type<String>()) // false

Type<Int>().isSubtype(of: Type<Int?>()) // true
```

To get the run-time type's fully qualified type identifier, use the `let identifier: String` field:

```cadence
let type = Type<Int>()
type.identifier  // is "Int"
```

```cadence
// in account 0x1

struct Test {}

let type = Type<Test>()
type.identifier  // is "A.0000000000000001.Test"
```

## Getting the type from a value

The method `view fun getType(): Type` can be used to get the run-time type of a value:

```cadence
let something = "hello"

let type: Type = something.getType()
// `type` is `Type<String>()`
```

This method returns the **concrete run-time type** of the object, **not** the static type.

```cadence
// Declare a variable named `something` that has the *static* type `AnyResource`
// and has a resource of type `Collectible`
//
let something: @AnyResource <- create Collectible()

// The resource's concrete run-time type is `Collectible`
//
let type: Type = something.getType()
// `type` is `Type<@Collectible>()`
```

## Constructing a run-time type

Run-time types can also be constructed from type identifier strings using built-in constructor functions:

```cadence
view fun CompositeType(_ identifier: String): Type?
view fun InterfaceType(_ identifier: String): Type?
view fun IntersectionType(types: [String]): Type?
```

Given a type identifier (or a list of identifiers for interfaces in the case of `IntersectionType`), these functions will look up nominal types and produce their run-time equivalents. If the provided identifiers do not correspond to any types, or (in the case of `IntersectionType`) the provided combination of identifiers would not type-check statically, these functions will produce `nil`:

```cadence
struct Test: I {}
struct interface I {}
let type: Type = CompositeType("A.0000000000000001.Test")
// `type` is `Type<Test>`

let type2: Type = IntersectionType(
    restrictions: ["A.0000000000000001.I"]
)
// `type2` is `Type<{I}>`
```

Other built-in functions will construct compound types from other run-types.

```cadence
view fun OptionalType(_ type: Type): Type
view fun VariableSizedArrayType(_ type: Type): Type
view fun ConstantSizedArrayType(type: Type, size: Int): Type
view fun FunctionType(parameters: [Type], return: Type): Type
// returns `nil` if `key` is not valid dictionary key type
view fun DictionaryType(key: Type, value: Type): Type?
// returns `nil` if `type` is not a reference type
view fun CapabilityType(_ type: Type): Type?
view fun ReferenceType(entitlements: [String], type: Type): Type?
```

## Asserting the type of a value

The method `view fun isInstance(_ type: Type): Bool` can be used to check if a value has a certain type, using the concrete run-time type, and considering subtyping rules:

```cadence
// Declare a variable named `collectible` that has the *static* type `Collectible`
// and has a resource of type `Collectible`
//
let collectible: @Collectible <- create Collectible()

// The resource is an instance of type `Collectible`,
// because the concrete run-time type is `Collectible`
//
collectible.isInstance(Type<@Collectible>())  // is `true`

// The resource is an instance of type `AnyResource`,
// because the concrete run-time type `Collectible` is a subtype of `AnyResource`
//
collectible.isInstance(Type<@AnyResource>())  // is `true`

// The resource is *not* an instance of type `String`,
// because the concrete run-time type `Collectible` is *not* a subtype of `String`
//
collectible.isInstance(Type<String>())  // is `false`
```

:::info

The **concrete run-time type** of the object is used, **not** the static type.

:::

```cadence
// Declare a variable named `something` that has the *static* type `AnyResource`
// and has a resource of type `Collectible`
//
let something: @AnyResource <- create Collectible()

// The resource is an instance of type `Collectible`,
// because the concrete run-time type is `Collectible`
//
something.isInstance(Type<@Collectible>())  // is `true`

// The resource is an instance of type `AnyResource`,
// because the concrete run-time type `Collectible` is a subtype of `AnyResource`
//
something.isInstance(Type<@AnyResource>())  // is `true`

// The resource is *not* an instance of type `String`,
// because the concrete run-time type `Collectible` is *not* a subtype of `String`
//
something.isInstance(Type<String>())  // is `false`
```

For example, this allows implementing a marketplace sale resource:

```cadence
access(all)
resource SimpleSale {

    /// The resource for sale.
    /// Once the resource is sold, the field becomes `nil`.
    ///
    access(all)
    var resourceForSale: @AnyResource?

    /// The price that is wanted for the purchase of the resource.
    ///
    access(all)
    let priceForResource: UFix64

    /// The type of currency that is required for the purchase.
    ///
    access(all)
    let requiredCurrency: Type
    access(all)
    let paymentReceiver: Capability<&{FungibleToken.Receiver}>

    /// `paymentReceiver` is the capability that will be borrowed
    /// once a valid purchase is made.
    /// It is expected to target a resource that allows depositing the paid amount
    /// (a vault which has the type in `requiredCurrency`).
    ///
    init(
        resourceForSale: @AnyResource,
        priceForResource: UFix64,
        requiredCurrency: Type,
        paymentReceiver: Capability<&{FungibleToken.Receiver}>
    ) {
        self.resourceForSale <- resourceForSale
        self.priceForResource = priceForResource
        self.requiredCurrency = requiredCurrency
        self.paymentReceiver = paymentReceiver
    }

    /// buyObject allows purchasing the resource for sale by providing
    /// the required funds.
    /// If the purchase succeeds, the resource for sale is returned.
    /// If the purchase fails, the program aborts.
    ///
    access(all)
    fun buyObject(with funds: @FungibleToken.Vault): @AnyResource {
        pre {
            // Ensure the resource is still up for sale
            self.resourceForSale != nil: "The resource has already been sold"
            // Ensure the paid funds have the right amount
            funds.balance >= self.priceForResource: "Payment has insufficient amount"
            // Ensure the paid currency is correct
            funds.isInstance(self.requiredCurrency): "Incorrect payment currency"
        }

        // Transfer the paid funds to the payment receiver
        // by borrowing the payment receiver capability of this sale resource
        // and depositing the payment into it

        let receiver = self.paymentReceiver.borrow()
            ?? panic("failed to borrow payment receiver capability")

        receiver.deposit(from: <-funds)
        let resourceForSale <- self.resourceForSale <- nil
        return <-resourceForSale
    }
}
```

