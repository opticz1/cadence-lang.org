---
title: Capabilities
sidebar_position: 11
---

Cadence supports [capability-based security] through the [object-capability model].

A capability in Cadence is a value that represents the right to access an object and perform certain operations on it. A capability specifies _what_ can be accessed, and _how_ it can be accessed.

Capabilities are unforgeable, transferable, and revocable.

Capabilities can be storage capabilities or account capabilities:

- **Storage capabilities** grant access to [objects in account storage] via [paths].
- **Account capabilities** grant access to [accounts].

Capabilities can be borrowed to get a [reference] to the stored object or the account it refers to.

Capabilities have the type `Capability<T: &Any>`. The type parameter specifies the kind of reference that can be obtained when borrowing the capability. The type specifies the associated set of access rights through [entitlements]: the reference type of the capability can be authorized, which grants the owner of the capability the ability to access the fields and functions of the target that require the given entitlements.

For example, a capability that has type `Capability<auth(SaveValue) &Account>` grants access to an account, and allows saving a value into the account.

Each capability has an ID, and the ID is unique **per account/address**.

Capabilities are created and managed through [capability controllers].

Capabilities are structs, so they are copyable. They can be used (i.e., borrowed) arbitrarily many times, as long as the target capability controller has not been deleted.

## `Capability`

General syntax:

```cadence
access(all)
struct Capability<T: &Any> {
    
    /// The address of the account that the capability targets:
    access(all)
    let address: Address

    /// The ID of the capability:
    access(all)
    let id: UInt64

    /// Returns a reference to the targeted object.
    ///
    /// If the capability is revoked, the function returns nil.
    ///
    /// If the capability targets an object in account storage,
    /// and no object is stored at the target storage path,
    /// the function returns nil.
    ///
    /// If the targeted object cannot be borrowed using the given type,
    /// the function panics.
    ///
    access(all)
    view fun borrow(): T?

    /// Returns true if the capability currently targets an object
    /// that satisfies the given type (i.e., could be borrowed using 
    /// the given type).
    ///
    access(all)
    view fun check(): Bool
}
```

## Capabilities in accounts

An [account] exposes its capabilities through the `capabilities` field, which has the type `Account.Capabilities`.

```cadence
access(all)
struct Capabilities {

    /// The storage capabilities of the account:
    access(mapping CapabilitiesMapping)
    let storage: Account.StorageCapabilities

    /// The account capabilities of the account:
    access(mapping CapabilitiesMapping)
    let account: Account.AccountCapabilities

    /// Returns the capability at the given public path.
    /// If the capability does not exist,
    /// or if the given type is not a supertype of the capability's 
    /// borrow type, returns an "invalid" capability with ID 0 that
    /// will always fail to `check` or `borrow`:
    access(all)
    view fun get<T: &Any>(_ path: PublicPath): Capability<T>

    /// Borrows the capability at the given public path.
    /// Returns nil if the capability does not exist, or cannot be
    /// borrowed using the given type. The function is equivalent
    /// to `get(path).borrow()`:
    access(all)
    view fun borrow<T: &Any>(_ path: PublicPath): T?

    /// Returns true if a capability exists at the given public path:
    access(all)
    view fun exists(_ path: PublicPath): Bool

    /// Publish the capability at the given public path.
    ///
    /// If there is already a capability published under the given path,
    /// the program aborts.
    ///
    /// The path must be a public path (i.e., only the domain `public`
    /// is allowed):
    access(Capabilities | PublishCapability)
    fun publish(_ capability: Capability, at: PublicPath)

    /// Unpublish the capability published at the given path.
    ///
    /// Returns the capability if one was published at the path.
    /// Returns nil if no capability was published at the path:
    access(Capabilities | UnpublishCapability)
    fun unpublish(_ path: PublicPath): Capability?
}

entitlement Capabilities

entitlement PublishCapability
entitlement UnpublishCapability
```

## Checking the existence of public capabilities

The function `capabilities.check` determines if a public capability was [published] at the given path before:

```cadence
access(all)
view fun exists(_ path: PublicPath): Bool
```

If the account has a capability published under the given path, the function returns true; otherwise, it returns false.

## Getting public capabilities with `.get()`

The function `capabilities.get` obtains a public capability that was [published] before:

```cadence
access(all)
view fun get<T: &Any>(_ path: PublicPath): Capability<T>
```

If the account has a capability with the given type published under the given path, the function returns it.

If the account has no capability published under the given path, or if the given type is not a supertype of the capability's borrow type, the function returns an "invalid" capability with ID 0 that will always fail to `check` or `borrow`.

## Borrowing public capabilities with `.borrow()`

The convenience function `capabilities.borrow` obtains and borrows a public capability that was [published] before, in one step:

```cadence
access(all)
view fun borrow<T: &Any>(_ path: PublicPath): T?
```

If the account has a capability with the given type published under the given path, the function borrows the capability and returns the resulting reference as an optional.

If the account has no capability published under the given path, or the requested type, via the type parameter `T`, and does not match the published capability, the function returns `nil`.

## Managing capabilities

Capabilities can be storage capabilities or account capabilities:

- **Storage capabilities** grant access to [objects in account storage] via [paths]. An account allows the management of storage capabilities through the `capabilities.storage` field, which has the type `Account.StorageCapabilities`.
- **Account capabilities** grant access to [accounts]. An account allows the management of account capabilities through the `capabilities.account` field, which has the type `Account.AccountCapabilities`.

A capability, and all its copies, is managed by a capability controller:

- **Storage capabilities** are controlled by storage capability controllers. Storage capability controllers have the type `StorageCapabilityController`.
- **Account capabilities** are controlled by account capability controllers. Account capability controllers have the type `AccountCapabilityController`.

### `Account.StorageCapabilities` and `Account.AccountCapabilities`

```cadence
access(all)
struct StorageCapabilities {

    /// Issue/create a new storage capability:
    access(Capabilities | StorageCapabilities | IssueStorageCapabilityController)
    fun issue<T: &Any>(_ path: StoragePath): Capability<T>

    /// Issue/create a new storage capability:
    access(Capabilities | StorageCapabilities | IssueStorageCapabilityController)
    fun issueWithType(_ path: StoragePath, type: Type): Capability

    /// Get the storage capability controller for the capability with the 
    /// specified ID.
    ///
    /// Returns nil if the ID does not reference an existing storage
    /// capability:
    access(Capabilities | StorageCapabilities | GetStorageCapabilityController)
    view fun getController(byCapabilityID: UInt64): &StorageCapabilityController?

    /// Get all storage capability controllers for capabilities that target
    /// this storage path:
    access(Capabilities | StorageCapabilities | GetStorageCapabilityController)
    view fun getControllers(forPath: StoragePath): [&StorageCapabilityController]

    /// Iterate over all storage capability controllers for capabilities
    /// that target this storage path, passing a reference to each
    /// controller to the provided callback function.
    ///
    /// Iteration is stopped early if the callback function returns `false`.
    ///
    /// If a new storage capability controller is issued for the path,
    /// an existing storage capability controller for the path is deleted,
    /// or a storage capability controller is retargeted from or to the
    /// path, then the callback must stop iteration by returning false.
    /// Otherwise, iteration aborts:
    access(Capabilities | StorageCapabilities | GetStorageCapabilityController)
    fun forEachController(
        forPath: StoragePath,
        _ function: fun(&StorageCapabilityController): Bool
    )
}

access(all)
struct AccountCapabilities {

    /// Issue/create a new account capability:
    access(Capabilities | AccountCapabilities | IssueAccountCapabilityController)
    fun issue<T: &Account>(): Capability<T>

    /// Issue/create a new account capability:
    access(Capabilities | AccountCapabilities | IssueAccountCapabilityController)
    fun issueWithType(_ type: Type): Capability

    /// Get capability controller for capability with the specified ID.
    ///
    /// Returns nil if the ID does not reference an existing account
    /// capability:
    access(Capabilities | AccountCapabilities | GetAccountCapabilityController)
    view fun getController(byCapabilityID: UInt64): &AccountCapabilityController?

    /// Get all capability controllers for all account capabilities:
    access(Capabilities | AccountCapabilities | GetAccountCapabilityController)
    view fun getControllers(): [&AccountCapabilityController]

    /// Iterate over all account capability controllers for all account
    /// capabilities, passing a reference to each controller to the provided
    /// callback function.
    ///
    /// Iteration is stopped early if the callback function returns `false`.
    ///
    /// If a new account capability controller is issued for the account,
    /// or an existing account capability controller for the account is
    /// deleted, then the callback must stop iteration by returning false.
    /// Otherwise, iteration aborts:
    access(Capabilities | AccountCapabilities | GetAccountCapabilityController)
    fun forEachController(_ function: fun(&AccountCapabilityController): Bool)
}

entitlement StorageCapabilities
entitlement AccountCapabilities

entitlement GetStorageCapabilityController
entitlement IssueStorageCapabilityController

entitlement GetAccountCapabilityController
entitlement IssueAccountCapabilityController

entitlement mapping CapabilitiesMapping {
    include Identity

    StorageCapabilities -> GetStorageCapabilityController
    StorageCapabilities -> IssueStorageCapabilityController

    AccountCapabilities -> GetAccountCapabilityController
    AccountCapabilities -> IssueAccountCapabilityController
}
```

### `AccountCapabilityController` and `StorageCapabilityController`

```cadence
access(all)
struct AccountCapabilityController {

    /// The capability that is controlled by this controller:
    access(all)
    let capability: Capability

    /// An arbitrary "tag" for the controller.
    /// For example, it could be used to describe the purpose of
    /// the capability.
    /// Empty by default:
    access(all)
    var tag: String

    /// Updates this controller's tag to the provided string:
    access(all)
    fun setTag(_ tag: String)

    /// The type of the controlled capability (i.e., the T
    /// in `Capability<T>`):
    access(all)
    let borrowType: Type

    /// The identifier of the controlled capability.
    /// All copies of a capability have the same ID:
    access(all)
    let capabilityID: UInt64

    /// Delete this capability controller,
    /// and disable the controlled capability and its copies.
    ///
    /// The controller will be deleted from storage,
    /// but the controlled capability and its copies remain.
    ///
    /// Once this function returns, the controller is no longer usable,
    /// all further operations on the controller will panic.
    ///
    /// Borrowing from the controlled capability or its copies will
    /// return nil:
    ///
    access(all)
    fun delete()
}
```

```cadence
access(all)
struct StorageCapabilityController {

    /// The capability that is controlled by this controller:
    access(all)
    let capability: Capability

    /// An arbitrary "tag" for the controller.
    /// For example, it could be used to describe the purpose of
    /// the capability.
    /// Empty by default:
    access(all)
    var tag: String

    /// Updates this controller's tag to the provided string:
    access(all)
    fun setTag(_ tag: String)

    /// The type of the controlled capability (i.e., the T
    /// in `Capability<T>`):
    access(all)
    let borrowType: Type

    /// The identifier of the controlled capability.
    /// All copies of a capability have the same ID.
    access(all)
    let capabilityID: UInt64

    /// Delete this capability controller,
    /// and disable the controlled capability and its copies.
    ///
    /// The controller will be deleted from storage,
    /// but the controlled capability and its copies remain.
    ///
    /// Once this function returns, the controller is no longer usable,
    /// all further operations on the controller will panic.
    ///
    /// Borrowing from the controlled capability or its copies
    /// will return nil:
    ///
    access(all)
    fun delete()

    /// Returns the targeted storage path of the controlled capability:
    access(all)
    fun target(): StoragePath

    /// Retarget the controlled capability to the given storage path.
    /// The path may be different or the same as the current path:
    access(all)
    fun retarget(_ target: StoragePath)
}
```

### Issuing capabilities

Capabilities are created by _issuing_ them in the target account.

**Issuing storage capabilities**

The `capabilities.storage.issue` function issues a new storage capability that targets the given storage path and can be borrowed with the given type:

```cadence
access(Capabilities | StorageCapabilities | IssueStorageCapabilityController)
fun issue<T: &Any>(_ path: StoragePath): Capability<T>
```

Calling the `issue` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` or `StorageCapabilities` entitlements (`aut (Capabilities) &Account` or `auth(StorageCapabilities) &Account`), or the fine-grained `IssueStorageCapabilityController` entitlement (`auth(IssueStorageCapabilityController) &Account`).

The path must be a storage path, and it must have the domain `storage`.

For example, the following transaction issues a new storage capability, which grants the ability to withdraw from the stored vault by authorizing the capability to be borrowed with the necessary `Withdraw` entitlement:

```cadence
transaction {
    prepare(signer: auth(IssueStorageCapabilityController) &Account) {
        let capability = signer.capabilities.storage.issue<auth(Withdraw) &Vault>(/storage/vault)
        // ...
    }
}
```

**Issuing account capabilities**

The `capabilities.account.issue` function issues a new account capability that targets the account and can be borrowed with the given type:

```cadence
access(Capabilities | AccountCapabilities | IssueAccountCapabilityController)
fun issue<T: &Account>(): Capability<T>
```

Calling the `issue` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` or `AccountCapabilities` entitlements (`auth(Capabilities) &Account` or `auth(AccountCapabilities) &Account`), or the fine-grained `IssueAccountCapabilityController` entitlement (`auth(IssueAccountCapabilityController) &Account`).

For example, the following transaction issues a new account capability, which grants the ability to save objects into the account by authorizing the capability to be borrowed with the necessary [`SaveValue` entitlement]:

```cadence
transaction {
    prepare(signer: auth(IssueAccountCapabilityController) &Account) {
        let capability = signer.capabilities.account.issue<auth(SaveValue) &Account>()
        // ...
    }
}
```

### Publishing capabilities

Capabilities can be made available publicly by _publishing_ them.

The `capabilities.publish` function publishes a capability under a given public path:

```cadence
access(Capabilities | PublishCapability)
fun publish(_ capability: Capability, at: PublicPath)
```

Calling the `publish` function requires access to an account via a reference that is authorized with the coarse-grained `Capabilities` entitlement (`auth(Capabilities)  Account`), or the fine-grained `PublishCapability` entitlement (`auth(PublishCapability) &Account`).

For example, the following transaction issues a new storage capability and then publishes it under the path `/public/vault`, allowing anyone to access and borrow the capability and gain access to the stored vault. Note that the reference type is unauthorized, so when the capability is borrowed, only publicly accessible (`access(all)`) fields and functions of the object can be accessed:

```cadence
transaction {
    prepare(signer: auth(Capabilities) &Account) {
        let capability = signer.capabilities.storage.issue<&Vault>(/storage/vault)
        signer.capabilities.publish(capability, at: /public/vault)
    }
}
```

### Unpublishing capabilities

The `capabilities.unpublish` function unpublishes a capability from a given public path:

```cadence
access(Capabilities | UnpublishCapability)
fun unpublish(_ path: PublicPath): Capability?
```

Calling the `unpublish` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` entitlement (`auth(Capabilities)  Account`), or the fine-grained `UnpublishCapability` entitlement (`auth(UnpublishCapability) &Account`).

If there is a capability published under the path, the function removes it from the path and returns it. If there is no capability published under the path, the function returns `nil`.

For example, the following transaction unpublishes a capability that was previously published under the path `/public/vault`:

```cadence
transaction {
    prepare(signer: auth(Capabilities) &Account) {
        signer.capabilities.unpublish(/public/vault)
    }
}
```

### Tagging capabilities

Capabilities can be associated with a tag, which is an arbitrary string. The tag can be used for various purposes, such as recording the purpose of the capability. It is empty by default and is stored in the capability controller.

Both storage capability controllers (`StorageCapabilityController`) and account capability controllers (`AccountCapabilityController`) have a `tag` field and a `setTag` function, which can be used to get and set the tag:

```cadence
access(all)
var tag: String

access(all)
fun setTag(_ tag: String)
```

### Retargeting storage capabilities

Storage capabilities (`StorageCapabilityController`) can be retargeted to another storage path after they have been issued.

The `target` function returns the storage path of the controlled capability, and the `retarget` function sets a new storage path:

```cadence
access(all)
fun target(): StoragePath

access(all)
fun retarget(_ target: StoragePath)
```

### Revoking capabilities

A capability and all of its copies can be revoked by deleting the capability's controller.

The `delete` function deletes a controller (`StorageCapabilityController` or `AccountCapabilityController`):

```cadence
access(all)
fun delete()
```

### Getting capability controllers

The capability management types `StorageCapabilities` and `AccountCapabilities` allow obtaining the controller for a capability, as well as iterating over all existing controllers.

**Getting a storage capability controller**

The `capabilities.storage.getController` function gets the storage capability controller for the capability with the given capability ID:

```cadence
access(Capabilities | StorageCapabilities | GetStorageCapabilityController)
view fun getController(byCapabilityID: UInt64): &StorageCapabilityController?
```

Calling the `getController` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` or `StorageCapabilities` entitlements (`auth(Capabilities) &Account` or `auth(StorageCapabilities) &Account`), or the fine-grained `GetStorageCapabilityController` entitlement (`auth(GetStorageCapabilityController) &Account`).

If a storage capability controller for the capability with the given ID exists, the function returns a reference to it, as an optional. If there is no storage capability controller with the given capability ID, the function returns `nil`.

**Getting an account capability controller**

The `capabilities.account.getController` function gets the account capability controller for the capability with the given capability ID:

```cadence
access(Capabilities | AccountCapabilities | GetAccountCapabilityController)
view fun getController(byCapabilityID: UInt64): &AccountCapabilityController?
```

Calling the `getController` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` or `AccountCapabilities` entitlements (`auth(Capabilities) &Account` or `auth(AccountCapabilities) &Account`), or the fine-grained `GetAccountCapabilityController` entitlement (`auth(GetAccountCapabilityController) &Account`).

If an account capability controller for the capability with the given ID exists, the function returns a reference to it, as an optional. If there is no account capability controller with the given capability ID, the function returns `nil`.

**Iterating over storage capability controllers**

The functions `getControllers` and `forEachController` allow iterating over all storage capability controllers of a storage path:

```cadence
access(Capabilities | StorageCapabilities | GetStorageCapabilityController)
view fun getControllers(forPath: StoragePath): [&StorageCapabilityController]

access(Capabilities | StorageCapabilities | GetStorageCapabilityController)
fun forEachController(
    forPath: StoragePath,
    _ function: fun(&StorageCapabilityController): Bool
)
```

Calling the `getControllers` and `forEachController` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` or `StorageCapabilities` entitlements (`auth(Capabilities) &Account` or `auth(StorageCapabilities) &Account`), or the fine-grained `GetStorageCapabilityController` entitlement (`auth(GetStorageCapabilityController) &Account`).

The `getControllers` function returns a new array of references to all storage capability controllers.

The `forEachController` function calls the given callback function for each storage capability controller and passes a reference to the function. Iteration stops when the callback function returns `false`.

**Iterating over account capability controllers**

The functions `getControllers` and `forEachController` allow iterating over all account capability controllers of the account:

```cadence
access(Capabilities | AccountCapabilities | GetAccountCapabilityController)
view fun getControllers(): [&AccountCapabilityController]

access(Capabilities | AccountCapabilities | GetAccountCapabilityController)
fun forEachController(_ function: fun(&AccountCapabilityController): Bool)
```

Calling the `getControllers` and `forEachController` function requires access to an account via a reference, which is authorized with the coarse-grained `Capabilities` or `AccountCapabilities` entitlements (`auth(Capabilities) &Account` or `auth(AccountCapabilities) &Account`), or the fine-grained `GetAccountCapabilityController` entitlement (`auth(GetAccountCapabilityController) &Account`).

The `getControllers` function returns a new array of references to all account capability controllers.

The `forEachController` function calls the given callback function for each account capability controller and passes a reference to the function. Iteration stops when the callback function returns `false`.

## Examples

**Entitlement Increment**

1. Declare a resource named `Counter` that has a field `count` and a function `increment`, which requires the `Increment` entitlement:

   ```cadence
   access(all)
   resource Counter {

       access(all)
       var count: UInt

       access(all)
       init(count: UInt) {
           self.count = count
       }

       access(Increment)
       fun increment(by amount: UInt) {
           self.count = self.count + amount
       }
   }
   ```

   In this example, an account reference is available through the constant `account`, which has the type `auth(Storage, Capabilities) &Account` (i.e., the reference is authorized to perform storage and capability operations).

1. Create a new instance of the resource type `Counter` and save it in the storage of the account. The path `/storage/counter` is used to refer to the stored value. Its identifier `counter` was chosen freely and could be something else:

   ```cadence
   account.storage.save(
       <-create Counter(count: 42),
       to: /storage/counter
   )
   ```

1. Issue a new storage capability that allows access to the stored counter resource:

   ```cadence
   let capability = account.capabilities.storage.issue<&Counter>(/storage/counter)
   ```

1. Publish the capability under the path `/public/counter`, so that anyone can access the counter resource. Its identifier `counter` was chosen freely and could be something else:

   ```cadence
   account.capabilities.publish(capability, at: /public/counter)
   ```

Imagine that the next example is in a different context, such as a new script or transaction:

1. Get a reference to the account that stores the counter:

   ```cadence
   let account = getAccount(0x1)
   ```

1. Borrow the capability for the counter that is made publicly accessible through the path `/public/counter`. The `borrow` call returns an optional reference `&Counter?`. The borrow succeeds, and the result is not `nil`; it is a valid reference because:

   - The path `/public/counter` stores a capability.
   - The capability allows it to be borrowed as `&Counter`, as it has the type `Capability<&Counter>`.
   - The target of the storage capability, the *path* `/storage/counter`, stores an object and it has a type that is a subtype of the borrowed type (type equality is also considered a subtype relationship).

1. Force-unwrap the optional reference. After the call, the declared constant `counterRef` has type `&Counter`:

   ```cadence
   let counterRef = account.capabilities.borrow<&Counter>(/public/counter)!
   ```

1. Read the field `count` of the `Counter`. The field can be accessed because it has the access modifier `access(all)`.

   Even though it is a variable field (`var`), it cannot be assigned to from outside of the object.

   ```cadence
   counterRef.count  // is `42`

   // Invalid: The `increment` function is not accessible for the
   // reference, because the reference has the type `&Counter`,
   // which does not authorize the entitlement `Increment`,
   // which is required by the `increment` function
   // (it has the access modifier ` access(Increment)`).

   counterRef.increment(by: 5)
   ```

1. Attempt to borrow the capability again for the counter, but use the type `auth(Increment) &Counter` to re-attempt the call to `increment`.

   Getting the capability fails, because the capability was issued using the type `&Counter`. After the call, `counterRef2` is `nil`.

   ```cadence
   let counterRef2 = account.capabilities.borrow<auth(Increment) &Counter>(/public/counter)
   ```

<!-- Relative links. Will not render on the page -->

[capability-based security]: https://en.wikipedia.org/wiki/Capability-based_security
[object-capability model]: https://en.wikipedia.org/wiki/Object-capability_model
[objects in account storage]: ./accounts/storage.mdx
[paths]: ./accounts/paths.mdx
[account]: ./accounts/index.mdx
[accounts]:  ./accounts/index.mdx
[reference]: ./references.mdx
[entitlements]: ./access-control.md
[capability controllers]: #capabilities-in-accounts
[published]: #publishing-capabilities
[objects in account storage]: ./accounts/storage.mdx
[`SaveValue` entitlement]: ./accounts/storage.mdx#saving-objects
