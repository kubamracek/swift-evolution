# Low-level operations for volatile memory accesses

* Proposal: [SE-NNNN](NNNN-filename.md)
* Authors: [Kuba Mracek](https://github.com/kubamracek), [Rauhul Varma](https://github.com/rauhul)
* Status: Pitch #2
* Implementation: [apple/swift#70944](https://github.com/apple/swift/pull/70944), available on `main` with `-enable-experimental-feature Volatile`.
* Discussion threads:
  * Pitch #1: https://forums.swift.org/t/pitch-low-level-operations-for-volatile-memory-accesses/69483
  * Pitch #2: TBD

## Introduction

Volatile operations are needed to program MMIO hardware registers in environments that have direct access to those, such as microcontroller firmware or kernel device drivers. This proposal adds APIs for the most basic load and store volatile operations, on 8, 16, 32 and 64 bit integers, and uses volatile semantics defined by Clang and LLVM.

## Motivation

Volatile memory operations are very common and fundamental in embedded programming for configuring hardware devices and peripherals. Whether this is done directly in the embedded application code, or indirectly via a library such as [Swift MMIO](https://github.com/apple/swift-mmio), these memory reads and writes must be marked as having volatile semantics and propagate as such through the compiler pipeline. Currently, this is impossible to express in Swift, and [many](https://github.com/apple/swift-mmio/blob/main/Sources/MMIOVolatile/MMIOVolatile.h) [projects](https://github.com/ole/swift-rp-pico-bare/blob/main/Sources/MMIOVolatile/MMIOVolatile.h) [workaround](https://github.com/apple/swift-embedded-examples/blob/main/stm32-blink/BridgingHeader.h) [this](https://github.com/apple/swift-embedded-examples/blob/main/pico-blink/Sources/Support/include/Support.h) [by using](https://github.com/kkebo/swift_os/blob/main/Sources/Volatile/Volatile.h) awkward and boilerplate helper C modules or bridging headers, just to be able to use the "volatile" keyword available in C:

```c
// In a bridging header
static inline uint8_t volatile_load_uint8_t(volatile uint8_t* pointer) {
  return *pointer;
}
static inline void volatile_store_uint8_t(volatile uint8_t* pointer, uint8_t value) {
  *pointer = value;
}
```

```swift
// In Swift
let controlRegisterPointer = UnsafeMutablePointer(bitPattern: 0xc000d800)
let control = volatile_load_uint8_t(controlRegisterPointer)
```

Alternatively, if a register is [not accessed using volatile semantics](https://github.com/ole/pico-embedded-swift/blob/main/SwiftLib/SwiftLib.swift), the program will inevitably break when compiler optimizations are enabled.

This proposal is aiming to resolve this by providing a way to express individual loads and stores with volatile semantics in Swift. However, it's not trying to accomplish anything else beyond that. Namely, it's not trying to be a complete user-facing high-level solution for volatile operations and/or for working with hardware registers, quite the opposite: It only makes the most primitive low-level volatile operations available, and assumes that safe(r) layers are built on top of those. [Swift MMIO](https://github.com/apple/swift-mmio/tree/main) is a prime example of such a layer, but users are free to build other layers, or where appropriate, use the primitives directly.

In C and C++, volatile is often misused for inter-thread synchronization (for which using volatile is almost always wrong and triggers undefined behavior) and other situations where volatile is not appropriate. The intended usage of the solutions from this proposal is in embedded software to program hardware registers, and it's desirable to prevent accidental usage and misuse.

## Proposed solution

The proposal is to create a new library/module called Volatile that will ship with the toolchain (given how close the logic is to the compiler implementation) and be available on all platforms and all environments. It will define a new type, `VolatilePointer<Pointee>` that will provide a volatile load and volatile store operation for UInt8, UInt16, UInt32 and UInt64 pointees (with UInt64 support only available on 64-bit CPUs).

With this API present, the code snippet presented in the Motivation section (which needed a bridging header) can now be written fully in Swift, without any helper modules and bridging headers:

```swift
import Volatile

let controlRegisterPointer = VolatilePointer<UInt8>(unsafeBitPattern: 0xc000d800)
let control = controlRegisterPointer.load()
```

## Detailed design

### Separate module

The set of valid use cases for using volatile operations is very narrow (programming hardware registers), but there are a lot of misconceptions and misuse patterns of volatile in C, in particular for synchronization across threads. To discourage accidental discovery of the volatile APIs in Swift, they will be located in a separate `Volatile` module that users must explicitly import to access the APIs.

### API design

The following API will be provided by the `Volatile` module:

```swift
/// A pointer for accessing "volatile" memory, e.g. memory-mapped I/O registers.
///
/// Do not use for inter-thread synchronization. This is only meaningful for
/// low-level operations on special memory addresses performed from OS kernels,
/// embedded firmware, and similar environments.
///
/// The semantics of volatile load and volatile store operations match the LLVM
/// volatile semantics. Notably, a volatile operation cannot be added, removed,
/// or reordered with other volatile operations by the compiler. They may be
/// reordered with non-volatile operations. For details, see
/// <https://llvm.org/docs/LangRef.html#volatile-memory-accesses>.
@frozen
public struct VolatilePointer<Pointee> {
  public init(unsafeBitPattern: UInt)
}

extension VolatilePointer where Pointee == UInt8 {
  /// Perform an 8-bit volatile load operation from the target pointer.
  ///
  /// Do not use for inter-thread synchronization.
  public func load() -> Pointee

  /// Perform an 8-bit volatile store operation on the target pointer.
  ///
  /// Do not use for inter-thread synchronization.
  public func store(_ value: Pointee)
}

extension VolatilePointer where Pointee == UInt16 {
  /// Perform a 16-bit volatile load operation from the target pointer.
  /// This operation is only valid when the pointer is at least 2-byte aligned.
  ///
  /// Do not use for inter-thread synchronization.
  public func load() -> Pointee

  /// Perform a 16-bit volatile store operation on the target pointer.
  /// This operation is only valid when the pointer is at least 2-byte aligned.
  ///
  /// Do not use for inter-thread synchronization.
  public func store(_ value: Pointee)
}

extension VolatilePointer where Pointee == UInt32 {
  /// ...
  public func load() -> Pointee

  /// ...
  public func store(_ value: Pointee)
}

#if _pointerBitWidth(_64)
extension VolatilePointer where Pointee == UInt64 {
  /// ...
  public func load() -> Pointee

  /// ...
  public func store(_ value: Pointee)
}
#endif
```

The load and store APIs are only defined for UInt8, UInt16, UInt32 and if the target is 64-bit then for UInt64, too. They are not defined for any other data types, and they are not defined for signed integer types.

The volatile semantics are defined by Clang and LLVM, see exact definition in the [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html#volatile-memory-accesses). The pointer must be at least naturally aligned for the load and store operations to be valid.

Users of `VolatilePointer` can expect that the load and store function calls are inlined to the call sites, and the actual volatile load and store operations are performed directly from user's code without a function call overhead (when applicable; e.g. inlining won't be possible if the load method is used as a function value and invoked indirectly).

### Volatile semantics

The semantics of the load and store operations on `VolatilePointer` are equivalent to the semantics of load/store operations on volatile pointers of the respective scalar types in C/C++:

```swift
// VolatilePointer's load() and store() in Swift...
let volatilePointer = VolatilePointer<UInt8>(...)
let loadedValue = p.load()
p.store(newValue)
```

```c
// ...is equivalent to volatile pointer accesses in C/C++:
volatile uint8_t *volatile_pointer = ...;
uint8_t loaded_value = *volatile_pointer;
*volatile_pointer = new_value;
```

Defining the semantics of volatile operations in Swift by referring to the semantics in C/C++ is practically the only available choice: For interoperability reasons, we need Swift volatile operations to be compatible with C and C++, and any different semantics are not going to be supported by LLVM. Differing from C/C++ would also cause unnecessary confusion to anyone already familiar them.

The exact behavior and effect of the volatile keyword in C/C++ in all situations is complex, but we are only bringing a small subset into Swift: Individual volatile operations on pointers of hardware-native scalar integer types. The exact semantics of those are relatively simple, and they're described in detail in the [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html#volatile-memory-accesses). In summary:

- Volatile operations cannot be added, removed, merged, or split by the compiler. For natively supported integer types, they will be emitted as a single machine instruction.
- The order of volatile operations relative to other volatile operations must be preserved by the compiler.
- Volatile operations are assumed to have platform-specific side-effects, but they cannot change content of other regular (non-volatile) memory.

It's worth explicitly calling out that volatile operations do not perform any inter-thread synchronization, and do not establish a happens-before relationship across threads. This is, among others, a major difference from what volatile means in Java and C#.

### Primitive operations only

This proposal is intentionally only providing the most basic primitives for performing volatile memory accesses and doesn't try to provide any safety or structured access, as that is left for libraries built on top of the low-level APIs.

The proposal doesn't try to make it possible to mark types, struct members or variables as volatile (like C allows). Only a pointer to eligible integer types can be volatile and a load/store using such a pointer is volatile. In other words, volatility is not a property of a type, a member, a value, or a variable. It's only a property of concrete load/store operations.

The proposal doesn't try to provide the ability for making more pointer types volatile beyond UInt8, UInt16, UInt32, UInt64. MMIO register access is inherently tied to the hardware native data types. Allowing custom types to become volatile would encourage misuse for inter-thread synchronization. Any specialized use cases where volatility on larger and/or different data types is needed, are encouraged to build library solutions for those.

`VolatilePointer` is intentionally not providing any conversion APIs from/to `UnsafePointer`, nor any pointer arithmetic or subscripting operations.

See "Alternatives Considered" section below for details.

## Alternatives considered

### Different naming of type and initializers

Several alternative naming options were considered around the type name and what initializers should be available:

```swift
// (0) what this proposal suggests: VolatilePointer, "unsafe" initializer
struct VolatilePointer<Pointee> {
  init(unsafeBitPattern: UInt)
}

// (1) alternative - type is "unsafe", initializer is "safe"
struct UnsafeVolatilePointer<Pointee> {
  init(bitPattern: UInt)
}

// (2) alternative - avoid the term "volatile"
struct UnsafeMMIOPointer<Pointee> { ... }
struct MMIOPointer<Pointee> { ... }

// (3) alternative - avoid the term "pointer", use a name that suggests the intended use case
struct VolatileRegister { ... }
struct VolatileMappedRegister { ... }
```

This proposal suggests `VolatilePointer` for multiple reasons (which the other options don't satisfy): The "pointer" part of the name makes it clear that the type is a memory address, hiding that seems contra-productive. Not using "register" means we don't unnecessarily restrict the use of the type, there are valid use cases of volatile memory operations that are not about accessing a hardware register. Keeping the term "volatile" in the name is highly desirable because it's prevalent in the embedded programming world already and the semantics match the volatile semantics from C. The only available initializer, `init(unsafeBitPattern:)`, conveys well that constructing the type is potentially not a memory safe operation. However, if the type is constructed correctly, subsequent use of the type (to perform loads and stores) cannot violate memory safety.

### Top-level functions, extensions on UnsafePointer

An alternative API set technically achieving the same functionality could just be top-level functions:

```swift
func volatileLoad(from: UnsafeMutablePointer<UInt8>) -> UInt8 // repeat for 16, 32, 64
func volatileStore(to: UnsafeMutablePointer<UInt8>, value: UInt8) // repeat for 16, 32, 64
```

Or extensions on Unsafe(Mutable)Pointer:

```swift
extension UnsafeMutablePointer<UInt8> { // repeat for 16, 32, 64
  func volatileLoad() -> UInt8
  func volatileStore(_ value: UInt8)
}
```

However, these alternatives were rejected on the grounds that they would make it too easy to start using volatile operations on "regular" pointers outside of the valid use cases (MMIO device access). Situations where one needs to use a volatile operation justify the verbosity and explicitness of creating an `VolatilePointer`. The initializer `init(unsafeBitPattern:)` also implies that the user needs to be extra careful about which address is passed in.

Note that nothing prevents the user from using the primitives described in this proposal to define their own shortcut APIs as project-specific helpers, if that's appropritate for the particular project.

### Rich(er) API surface for arithmetics, conversions, structured access

This proposal avoids adding any pointer arithmetic facilities to `VolatilePointer`, any conversion APIs between UnsafePointer and `VolatilePointer,` or any structured or offset-based access APIs. While it would be technically possible to implement, we want to avoid thinking about or using instances of `VolatilePointer` as pointers. Forcing users to be explicit when using volatile memory accesses is an intentional goal of this proposal:

```swift
let baseRegister = VolatilePointer(...)
let r1 = baseRegister.advanced(by: 4) // error, no such API
let r2 = baseRegister.pointer(to: \MyType.myField) // error, no such API
```

If it's necessary to compute the MMIO register's address, and/or by using offsets to members of types, this is left to the user (or ideally, a library) to build on top of the primitives provided by this proposal.

### Generalized support for different pointee types

In this proposal, only UInt8, UInt16, UInt32 and UInt64 become supported pointee types for `VolatilePointer`.

This is intentional and contrary to how e.g. the `Atomic` type is designed. `Atomic<T>` in the standard library support is not restricted to a fixed set of `T` types, instead any type that is `AtomicReprentable` is supported. The standard library makes a lot of types `AtomicReprentable`: booleans, floating-point types, signed and unsigned integers, optionals, pointers. Users can make their own types conform to `AtomicReprentable`, too. 

However, the use cases for using volatile operations to program MMIO registers break the analogy with `Atomic<T>`. For example, a "volatile (pointer to a) boolean" does not make sense: The CPU typically cannot address a single bit, and there's no reasonable way of handling operations of such a type: Automatically promoting the type to have an 8-bit "storage" would be counter-intuitive at best, and emitting bitfield manipulation instructions will not allow modifying multiple bits on a single register at once.

Same argument applies to other non-CPU-native data types, and also to allowing users to make their own types eligible for `VolatilePointer`. It seems best to only provide primitive operations on the types that directly match the CPU's native integer types, and defer the entire problem domain of providing rich, structured high-level access with semantically matching data types to a library (which will use the primitives as the underlaying implementation).

This is exactly how [Swift MMIO](https://github.com/apple/swift-mmio) is designed. Notably, it can surface single-bit fields in a register as Bools, and it provides a `.modify { ... }` closure-based API to allow collapsing several field modifications into a single volatile operation. For example:

```swift
import MMIO

func turnLEDOnOrOff(enable: Bool) {
  GPIO.cr.modify { $0.enable = enable }
  // the accessors compute the correct pointee/offset and perform a volatile store
}
```

### Volatile types, volatile variables

In C, "volatile" is a type qualifier, and it's specifically allowed even on non-pointer types and on aggregate types. This proposal chooses an opposite approach to specifically avoid all the ambiguity and non-intuitive behavior that becomes possible when allowing types/variables/members to be volatile. In particular, for embedded software it's extremely important that the user is in control of which exact (volatile) operations happen on what addresses/registers. Thus it's important that it's obvious from the code what exact volatile operations happen, of what size, and in which order. There are many counter-examples of this in C:

```c
// (1)
volatile int volatile_int;
volatile_int++; // is this a load + store? or an single-instruction increment (e.g. on x86)?

// (2)
volatile int *reg_ptr = ...;
*reg_ptr = 1; // differently sized store based on the size of "int"

// (3)
volatile struct S { uint32_t a; uint32_t b; };
S *s = ...;
uint32_t loaded_value = s->a; // this actually isn't a volatile load (!)

// (4)
struct S { volatile uint32_t a; volatile uint32_t b; };
S *s = ...;
struct S local_copy = *s; // is this a memcpy? two volatile loads? something else?

// (5)
struct S { volatile uint64_t x; volatile uint64_t y; };
struct T { volatile struct S inner; };
struct T *t = ...;
struct S local_copy = t->inner; // is this two volatile loads?
uint64_t loaded_value = a.x; // is *this* a volatile load? from the local copy?
```

What embedded software developer who use C end up doing in practice, is that they simply stay away from any of these ambiguous or non-intuitive usages of volatile. They instead restrict themselves to only use volatile types in specific controlled situations. One such situation is using pointers to volatile types of CPU-native integers:

```c
uintptr_t device_base = MY_DEVICE_BASE;
uint16_t offset = MY_DEVICE_CTRL_REG_OFFSET;
*(volatile uint8_t *)(base + offset) = new_value;
```

This use of volatile is very clear and intuitive, which is why this proposal suggests to bring this ability into Swift.

Another common situation in C is to use a structure with precisely defined and aligned volatile fields representing the individual registers of a device:

```c
struct my_device_t {
	volatile uint8_t CTRL; // offset 0x0
	volatile uint8_t FEAT; // offset 0x1
  ...
};
struct my_device_t *device_ptr = (struct my_device_t *)MY_DEVICE_BASE;
device_ptr->FEAT = 1;
```

As long as the individual fields are well-aligned natively-supported integer types, this usage is also very clear, but a user of such a type must refrain from using e.g. increment/decrement operators, and a little bit of ambiguity exists even when manipulating bits inside the register, which is extremely common in embedded programming:

```c
device_ptr->CTRL |= DEVICE_CTRL_ENABLE_BIT; // is this a volatile load + volatile store?
```

While the above is generally accepted as a valid pattern in C (which will result in one volatile load and one volatile store), it suffers from suboptimal type safety. This proposal encourages users to use library solutions, such as [Swift MMIO](https://github.com/apple/swift-mmio), which can both solve the type safety problem, and provide APIs that have result in intuitive volatile operations on the hardware layer.

### Separate read-only, read-write and write-only types

Hardware registers often have restrictions on how they can be used, for example read-only and write-only registers. Given that the proposed APIs are introducing a type that allows both reads and writes, it's easy to imagine also adding sibling types that are read-only and write-only. However, this is against the design approach of this proposal, which aims to provide the smallest useful set of primitives. Instead, such restrictions should be added as a library built on top of the primitives. [Swift MMIO](https://github.com/apple/swift-mmio) specifically solves read-only/write-only safety, alongside providing many more facilities for structured and safe access to hardware registers.

