# Low-level linkage control

* Proposal: [SE-NNNN](NNNN-filename.md)
* Status: Pitch #2
* Discussion threads:
  * Pitch #1: https://forums.swift.org/t/pitch-low-level-linkage-control-attributes-used-and-section/65877
  * Pitch #2: https://forums.swift.org/t/pitch-2-low-level-linkage-control/69752

## Introduction

This proposal adds a new attribute `@linkage` into the language that can be used to directly control link-time properties of symbols that represent global variables and functions, overriding the compiler defaults. The goal is to enable certain systems and embedded programming use cases (e.g. allowing code or data to be placed into a custom section) and to serve as low-level building blocks for higher-level features (e.g. “linker sets”). The intention is that this attribute is to be used rarely by specific use cases; high-level application code should not need to use it directly and instead should rely on libraries, macros and other abstractions over the low-level attributes.

## Motivation

The [SE-0385 (Custom Reflection Metadata) proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0385-custom-reflection-metadata.md) explains the need for testing frameworks to allow their clients to annotate their types and other declarations to make them discoverable and enumerable at runtime, as well as to let the annotations attach user-specified data to those declarations. However, [SE-0385 was returned for revision](https://forums.swift.org/t/returned-for-revision-se-0385-custom-reflection-metadata/63758) with the argument (among others) that static (compile-time) generation of the metadata should be preferred:

>*The runtime discovery mechanism relies on executing generator functions to produce run-time values for custom attribute metadata. This design precludes static extraction of custom metadata, such as a tool that extracts the set of available tests from a binary. The design should consider whether the generated metadata could be made more amenable to static extraction, as well as more efficient queries, by considering the interaction with constant initialization.*

This proposal is motivated by the same needs as SE-0385 (see the [Motivation section in SE-0385](https://github.com/apple/swift-evolution/blob/main/proposals/0385-custom-reflection-metadata.md#motivation)), and is taking an approach that addresses the above-mentioned rejection reason. However, this proposal isn’t trying to be a direct replacement of SE-0385, as it has a limited scope of the additions to only provide the lowest-level building blocks to produce custom runtime-queryable compile-time information, and also enables static / offline / out-of-process extraction.

We can think about the needs of SE-0385 (attaching custom runtime-discoverable data on declarations) as two separate parts: (1) We allow users to define a global let or var with a fixed-layout type that is constant-initialized and goes into a specific, named section. (2) We provide an API for iterating through the records within a specific, named section. Together these form a “linker set” feature — the linker is the entity that assembles all the records together, and makes them reasonably discoverable at runtime.

*Note: The "linker set" mechanism is is an approach that Swift is already using: Nearly any kind of compiler-emitted metadata is put into a specifically-named section in the binary and given a fixed-layout record. Then when we want to do a lookup for some information — say, to find the protocol conformances in the binary — we ask the loader (dyld on Darwin) to give us the start/end address of that section in each of the loaded images, and then iterate through all of the records in those sections. It's also possible to extract some of that metadata from outside the process (via OS provided cross-process memory reading APIs), or from the binary itself. This is done today with the existing reflection library in, e.g., swift-inspect and swift-reflection-dump tools.*

At the same time, embedded and systems programming use cases have similar needs that can be solved by the same low-level building blocks: Directly controlling the sections and other link-time properties of data and code (e.g. to have a statically linked program gather a set of initialization functions, or to have different firmware components declare their properties without any runtime calls) is a commonly needed language feature which is available in C/C++. Enabling such a control in Swift will provide a more intuitive and safer implementation option, and users of Swift for embedded devices won’t need to reach for C/C++ as a workaround. Perhaps even more importantly, in order to interoperate with existing C-based embedded software, libraries and build systems might *require* that Swift components produce data structures in custom sections to satisfy a pre-existing contract.

One more concrete motivating example of needing a custom section control can be seen in the [Debug Description macro pitch](https://forums.swift.org/t/pitch-debug-description-macro/67711), where a prospective `@DebugDescription` macro generates a “summary string” for LLDB that summarizes the contents of the fields of a type *without the need for runtime evaluation*, but rather as a composition of the fields that LLDB assembles. To make those summary strings discoverable by LLDB, placing them into a custom section is a clean solution allowing LLDB to consume them in the case where LLDB has access to the binary on disk, or even without that. Such a debugging feature is nowadays even more relevant to embedded programming where runtime evaluation from a debugger is commonly not possible at all.

```swift
@DebugDescription struct Student { 
  var name: String
  var id: Int

  /* synthesized by the @DebugDescription macro, made discoverable by LLDB */
  let __Student_lldb_summary = ("PupilKit.Student", "${var.id}: ${var.name}")
}
```

## Proposed Solution

This proposal adds the ability into the Swift language to express the first part of the “linker set” mechanism: Placing fixed-layout records into specifically-named sections.

Concretely, the proposal is to add a new attribute `@linkage` with arguments that will allow annotating functions and global variables with desired low-level symbol settings (custom section, no-dead-stripping aka "attribute used"), and can be easily extended in the future, e.g. to add control over symbol visibility, alignment, exact linker-level symbol name, and more — see the Future Directions section for details.

```swift
// place entry into a section, mark as "do not dead strip"
// also implicitly make the global guaranteed to be statically initialized
@linkage(section: "__DATA,mysection", used)
let myLinkerSetEntry: Int = 42

// initializer expressions that cannot be constant-folded trigger an error
@linkage(section: "__DATA,mysection", used)
let myLinkerSetEntry: Int = Int.random(in: 0 ..< 10) // error

// code for the function is placed into the custom section
@linkage(section: "__TEXT,boot")
func firmwareBootEntrypoint() { ... }

// attribute syntax allows for future extensions (not part of this proposal):
// @linkage(visibility: .external, alignment: 64, weak, ...)
```

On top of specifying a custom section name, it’s often necessary to also mark the global as “used” — this is needed on declarations that would otherwise be removable by the compiler, e.g. inside non-public types or when the annotation is used on a non-public global which otherwise has no compile-time users. Linker set entries are typically not going to have users at compile time, and at the same time they should not be exposed in the public interface of libraries (not be made public), thus the need for the “used” marker.

## Detailed design

### Attribute @linkage

Attribute `@linkage` can be applied to variable and function declarations. It can have multiple parameters (delimited with a comma), but must have at least one parameter, and the attribute cannot be used more than once on the same declaration. Allowed parameters:

* `section: <string literal>` ... places the global/static variable or function into a custom section
* `used` ... marks the global/static variable or function as “do not dead strip”

See Future Directions and Alternatives Considered below for possible future additions and alternative syntax and usage.

### Attribute @linkage on global and static variables

Attribute @linkage can be used on variable declarations under these circumstances:

* the variable can be either a `let` or a `var`
* the variable must be a global variable or a static member variable (no local variables, no non-static member variables)
* the variable must not be declared inside a generic context (either directly in generic type or nested in a generic type)
* the variable must be a stored property (not be a computed property)
* the variable must not be a global declared as part of top-level executable code (i.e. main.swift)
* the initial expression assigned to the variable must be statically evaluable (see below)

*Note: These restrictions limit the `@linkage` attribute on variables to only be applicable to variables that can be expected to be represented as exactly one statically-initialized global storage symbol (in the linker sense) for the variable’s content. This is generally true for all global and static variables in C, but in Swift global variables might have zero global storage symbols (e.g. a computed property), or need non-trivial storage (e.g. lazily initialized variables with runtime code in the initialization expression).*

```swift
@linkage(section: "__DATA,mysection", used)
let global = 42 // ok, will force the global symbol to be emitted
                // by the compiler, will be placed into the section and marked "do not dead-strip"

@linkage(section: "__DATA,mysection", used)
var global = 42 // ok

@linkage(section: "__DATA,mysection", used)
var computed: Int { return 42 } // ERROR: @linkage cannot be used on computed properties

struct MyStruct {
  @linkage(section: "__DATA,mysection", used)
  static let staticMember = 42 // ok

  @linkage(section: "__DATA,mysection", used)
  static var staticMember = 42 // ok
  
  @linkage(section: "__DATA,mysection", used)
  let member = 42 // ERROR: @linkage cannot be used on non-static members

  @linkage(section: "__DATA,mysection", used)
  var member = 42 // ERROR: @linkage cannot be used on non-static members
}

struct MyGenericStruct<T> {
  @linkage(section: "__DATA,mysection", used)
  static let staticMember = 42 // ERROR: @linkage cannot be used in a generic context

  @linkage(section: "__DATA,mysection", used)
  static var staticMember = 42 // ERROR: @linkage cannot be used in a generic context
}

// when in main.swift:
@linkage(section: "__DATA,mysection", used)
var global = 42 // ERROR: @linkage cannot be used in top-level executable code
```

When allowed, the `@linkage(section: ...)` attribute on a variable declaration has two effects:

1. The variable’s initializer expression is going to be constant folded at compile-time (see below), and assigned as the initial value to the storage symbol for the variable. This implies that the variable’s value will not be lazily computed at runtime, and it will not use the one-time initialization helper code and token.
2. The storage symbol for the variable gets the individual properties from the @linkage annotation (i.e. the section name, the “used” marker, etc).

The properties applied to the storage symbols don’t affect optimizations and other transformations in the compiler. For example, the compiler is still allowed to propagate and copy the value of an annotated global `let` to code that uses the value, so there’s no guarantee that a value stored into global with a custom section will not be propagated and therefore “leak” outside of the section.

The `@linkage(used)` annotation, however, does inform the optimizer that such a variable cannot be removed, even when it doesn’t have any observed users or even if it’s inaccessible due to language rules (e.g. it’s a fileprivate static member on an otherwise empty type).

### Attribute @linkage on functions

Attribute `@linkage` can be used on function declarations under these circumstances:

* the function can be declared anywhere (a top-level function, a method on a type, a static method, nested function in another function body)
* the function must not be declared inside a generic context (either directly in generic type or nested in a generic type)
* the function must not be generic

These restrictions limit the `@linkage` attribute on functions to only be applicable to functions where we can expect the function to be compiled down to exactly one primary symbol (in the linker sense) for the function body’s content. If the function requires any sort of thunks to be generated by the compiler, e.g. reabstraction thunks, `@objc` thunks, resilient dispatch thunks, or distributed actor method thunks, those thunks are not affected by the `@linkage` attribute. If the function uses any language constructs that result in the generation of more auxiliary functions, e.g. default argument expressions, or closures, then those auxiliary functions are also not affected by the `@linkage` attribute. Nested functions inside a `@linkage`-annotated function do not inherit the attribute.

```swift
@linkage(section: "__TEXT,boot", used)
func entryPoint() { ... } // ok, will be placed into the custom section
                          // and marked "do not dead-strip"

@linkage(section: "__TEXT,boot", used)
func generic<T>(...) -> T { ... } // ERROR: @linkage cannot be used on generic functions

struct MyStruct {
  @linkage(section: "__TEXT,boot", used)
  func memberFunction() { ... } // ok

  @linkage(section: "__TEXT,boot", used)
  static func staticFunction() { ... } // ok
}

struct MyGenericStruct<T> {
  @linkage(section: "__TEXT,boot", used)
  func memberFunction() { ... } // ERROR: @linkage cannot be used in a generic context
}

```

The `@linkage(section: ...)` attribute on a function declaration has the effect of placing the symbol that contains the code of the function body into the specific section in the object file.

Similarly to specifying a section on globals/statics, the attribute doesn’t affect optimizations and other transformations in the compiler. For example, the compiler is still allowed to inline the function into its callers, so there’s no guarantee that the code from the function body cannot “leak” outside of the section.

The `@linkage(used)` annotation, however, does inform the optimizer that such a function cannot be removed, even when it doesn’t have any observed users or even if it’s inaccessible due to language rules (e.g. it’s a fileprivate function in an otherwise empty type).

### Guaranteed static initialization

As mentioned above, using attribute @`linkage` on a global/static variable will require the initial expression to be compile-time evaluable and the resulting global variable symbol in the binary will be guaranteed to be statically initialized with the resulting value. This is a crucial property, without guaranteed static initialization we wouldn’t be able achieve the offline / static extraction of the data as described in the Motivation section.

This proposal is intentionally trying to introduce only the minimal possible amount of change to satisfy the needs for guaranteed static initialization on *most commonly needed types*. This should be contrasted with how the [SE-0359 (Build-Time Constant Values) proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0359-build-time-constant-values.md) tried to introduce an attribute/qualifier into the language to build a rich system of compile-time knowable values. Some of the reasons for the rejection of SE-0359 were:


>Concerns about the virality of @const: [...] @const will end up being propagated throughout large swaths of Swift code, possible unnecessarily. Experience with related features of other languages (e..g, C++ constexpr) has shown this to be a problem. [...]

>Concerns that this proposal is too small a step toward constant evaluation: [...] by itself it provides only the guarantee that the argument, initializer, or witness uses a restricted form of expressions (based on literals) and types. [...]

>Whether @const properties are guaranteed to be emitted as constant objects in the binary: [...]


This proposal is addressing these concerns and also at the same time not locking down any concrete design for a full future constant / compile-time system in the language (see discussion below in Alternatives Considered), by avoiding any new keywords, attributes or qualifiers. Instead, only the expressions and declaration that have a *requirement for being guaranteed to be constant-folded*, like the presence of `@linkage` attribute, will require it. It’s relatively easy to imagine more reasons why expressions might be required to be constant-folded, for example a `#static_assert(expression)` feature for compile-time assertions (see Future Directions below).

Constant-folding already happens today in practice on a wide range of initializer expressions of global/static variables in optimized builds, and to some degree even in debug builds. The approach in this proposal is to lean on the mandatory optimizations pipeline in the SIL optimizer to guarantee that certain expressions and their compositions are constant-foldable *in all build types* (even non-optimized debug builds), because the use cases for the `@linkage` attribute require the resulting symbol to be statically-initialized for correctness, not just as an optimization.

Following is a breakdown of what expressions are going to be guaranteed to be constant-foldable, but the mandatory optimizations pipeline might be able to constant-fold more expressions and compositions of expressions.

Basic integer, boolean, floating-point and static string types and their literals are going to be always constant-foldable. Note that even though literals syntactically seem trivial to constant-fold, they actually end up using standard library types that wrap compiler builtins and use constructors with Swift code inside them, all of which must still be constant-folded by the mandatory optimizations.

```swift
@linkage(...) let global: Int = 42 // ok
@linkage(...) let global: UInt = 42 // ok
@linkage(...) let global: UInt8 = 42 // ok
@linkage(...) let global: Bool = true // ok
@linkage(...) let global: Float = 7.77 // ok
@linkage(...) let global: StaticString = "string literal" // ok
```

Tuples consisting of other constant-foldable expressions and types are constant-foldable:

```swift
@linkage(...) let global: (Int, Int) = (42, 43) // ok
@linkage(...) let global: (Bool, Int) = (true, 42) // ok
@linkage(...) let global: (Int?, Int?) = (nil, 42) // ok
```

UnsafePointer, UnsafeRawPointer and other standard library pointer types are constant-foldable:

```swift
@linkage(...) let global: UnsafeRawPointer? = UnsafeRawPointer(bitPattern: 0xaaaa_aaaa) // ok
@linkage(...) let global: UnsafePointer<UInt>? = UnsafePointer(bitPattern: 0xaaaa_aaaa) // ok
```

Custom structs with a frozen layout and a trivial initializer are constant-foldable if all the stored properties only use other constant-foldable types and the initializer call uses constant-foldable expressions:

```swift
/*@frozen*/ struct MyStruct { var x, y: Int }
@linkage(...) let global: MyStruct = MyStruct(x: 42, y: 777) // ok
```

As mentioned above, the definition of what exactly is and isn’t constant-foldable is fluid and depends on the abilities of the SIL optimizer. However, there are expressions that we cannot reasonably expect to be constant-foldable, for example expressions that call truly dynamic runtime code (e.g. random number generation), types that have layout with non-foldable members (references), or types that have allocating code in their initializer, etc.:

```swift
@linkage(...) let global: Bool = .random() // error

@linkage(...) let global: [Int] = [1, 2, 3] // error, though a future compiler is likely
// to get smart enough to statically initialize this and allow @linkage on constant arrays

@linkage(...) let global: [Int:Int] = [:] // error
@linkage(...) let global: String = "string" // error
@linkage(...) let global: Any = 42 // error
```

## Source compatibility

This proposal is purely additive (adding a new attribute), without impact on source compatibility.

## Effect on ABI stability

This change does not impact ABI stability for existing code.

Adding, removing, or changing the `@linkage` attribute on both variables and functions should generally be viewed as an ABI breaking change for those individual declarations and the symbols they generate. Such code changes should only be applied in cases where the ABI is not locked down (e.g. newly added declarations), and/or the user must carefully consider the exact effect of the change and whether is breaks an existing ABI. In some cases, it is possible to make careful non-ABI-breaking changes via the `@linkage` attribute.

## Effect on API resilience

This change does not impact API resilience for existing code.

Adding, removing, or changing the `@linkage` attribute on both variables and functions is considered a API breaking change in resilient modules.

## Future Directions

### Additional parameters in the @linkage attribute

This proposal introduces the `@linkage` attribute that is a container for parameters that actually specify desired linkage settings, but only adds two of them, “section” and “used”, to solve a concrete use case with them. One can imagine a whole range of other linkage settings that can be exposed this way in the future:

```swift
@linkage(alignment(64)) var cacheLineAlignedData = ...
@linkage(mangledName: "customMangling") var abiCompatibilitySymbol = ...
@linkage(weak) func overridableFunctionWithDefaultImplementation() { ... }
@linkage(visibility: .external, cSymbolName: "...") func pluginEntryPoint() { ... }
```

### Linker set language feature / API

This proposal only builds one part of a user-facing “linker set” mechanism (placing structured data into sections). To become a full replacement for [SE-0385](https://github.com/apple/swift-evolution/blob/main/proposals/0385-custom-reflection-metadata.md), we still need to build the runtime mechanisms / APIs that query the data and allow enumeration of the linker set entries. One can imagine a direct, still relatively low-level API like this:

```swift
func enumerateLinkerSet<T>(fromSectionNamed: String) -> Sequence<T> {
  // extract section data, assuming the raw data in the section are records of "T"
  // use dyld APIs when on Darwin, start/stop symbols when using static linking, etc.
}
```

But a solution based on macros could achieve a higher-level abstraction for the entire “linker set” mechanism:

```swift
@DefineLinkerSet("name", type: Int) // other macros understand that linker set "name" 
                                    // has entries of type Int

@LinkerSetEntry("name") let entry1: Int = 42 // ok
@LinkerSetEntry("name") let entry2: Float = 7.7 // error

for entry in #enumerateLinkerSet("name") {
  print(entry)
}
```

### Allowing the attribute to be used in generic contexts

This proposal prohibits the use of `@linkage` in generic contexts. Global variables cannot be generic and static variables are prohibited in generic contexts already in general, but one can imagine that placing generic functions into custom sections could make sense and be allowed:

```swift
// all instantiations of this function are placed into the custom section,
// specialization is mandatory, and no generic implementation is emitted
@linkage(section: "__TEXT,boot")
func isMemoryCorrectlyInitialized<T>(pointer: UnsafeBufferPointer<T>, expectedValue: T)
```

### Generalized constant evaluation

Several Swift evolution pitches/proposals have in the past attempted to build foundations of a generalized system for constant evaluation, compile-time expressions and related language features. Namely, [SE-0359 (Build-Time Constant Values)](https://github.com/apple/swift-evolution/blob/main/proposals/0359-build-time-constant-values.md) and also an earlier [Pitch for Compile Time Constant Expressions](https://gist.github.com/marcrasi/b0da27a45bb9925b3387b916e2797789), both of which attempted to introduce a new qualifier/modifier to variables, declarations, functions and arguments, and feedback on these proposal expressed interest in removing the need to adding these “viral” annotations to all code/data that wants to participate in compile-time evaluation.

This proposal presents a potentially interesting counter-option: The compilation pipeline attempts forced constant-folding and compile-time evaluation *only when there is higher-level requirement to do so*. The use case for that described in this proposal is placing data into custom sections, but one could imagine the same approach for compiler-evaluation in general. If an expression is required to be compile-time evaluated, the compiler performs a top-down attempt to do so (this is essentially equivalent to assuming that all the participating code/variables *are implicitly annotated with @const / @compilerEvaluatable*) and only the one final expression is what needs to be annotated or otherwise use the language feature that triggers the compile-time evaluation. What’s left is listing the potential higher-level requirements, see the referenced proposals for those, two concrete examples are described below.

#### Compile-time evaluation of raw-value enum cases

Enums backed by raw value integers or strings today strictly require that the case values are literals. With compile-time evaluation as described in this proposal, the following could simply be allowed and the case values would be guaranteed to constant-fold:

```swift
enum PageSize: Int {
  case fourK = 4 * 1024     // ok, guaranteed to constant-fold
  case sixteenK = 16 * 1024 // ok, guaranteed to constant-fold
}
```

#### Compile-time evaluation for #static_assert

If we have the ability to guarantee that an initializer expression of a global/static is constant-foldable, the same mechanism can be used to implement a compile-time assertion (`#assert` or `#static_assert` or `#const_assert` as previously pitched on the Swift forums). Building on top of this proposal’s approach would allow only forcing expressions to be constant-folded when there’s a requirement for it (rather than by annotating all such code with a new modifier):

```swift
// what previous proposals/pitches suggest: viral annotations on all code/variables
// that are allowed to participate in static assertion expressions -- note that there
// is also a lot of stdlib code involved, e.g. Int, operator +, operator ==
@const let global = 0
@const func foo() -> Int { return global + 1 }
#static_assert(foo() == 1)

// static assertions built on this proposal's approach would not need the modifier
let global = 0
func foo() -> Int { return global + 1 }
#static_assert(foo() == 1)
// essentially just @linkage(...) var check: Bool = (foo() == 1)
// with an extra requirement that the folded boolean value is true
```

## Alternatives Considered

### Alternative syntax for the attribute

Several different spellings and usage patterns on attribute `@linkage` were considered:

* Separate attributes instead of a unified one, i.e. separate attributes `@used` and `@section`, and possibly other for other future linkage control settings (`@alignment` , `@weak` , etc.). This approach was initially pitched on the Swift forums, and received feedback pointed out issues with extensibility and discoverability. The unified `@linkage` attribute with parameters addresses that.
* Naming the attribute `@symbol` or `@symbolName` instead. Given that the attribute does not only affect the name, and does not actually change the generated symbol, only its linkage properties, `@linkage` is preferred.
* Naming the attribute `@unsafeLinkage` instead. Given the low-level nature of the linkage control settings, `@unsafeLinkage` spelling might seem like a reasonable name to highlight that incorrect usage might cause crashes that are outside of the high-level programming model of Swift. However, there is not a precedent for this: The only currently supported non-underscored attribute in Swift that uses “unsafe” in its name is `@unsafe_no_objc_tagged_pointer`, which if used incorrectly directly violates the memory safety properties of Swift, and just because an attribute can be used in a wrong way and cause crashes today doesn’t necessitate the “unsafe” word in the syntax. The two concrete settings that this proposal expose, custom sections and the “used” marker, also don’t violate Swift’s safety on their own; it’s only interaction with other OS/system components that might not work as expected.
* Structured section names with separate segment and section names, `@linkage(segment: "...", section: "...")`. This pattern does not generalize across object file formats, and is Mach-O specific (ELF and PE/COFF don’t have segments).
* Shorthand syntax to specify different section names for different object file formats (often a necessity to support multiple file formats), `@linkage(sectionELF: “...”, sectionMachO: “...”, sectionCOFF: “...”)`. Embedded and systems use cases often only support a single object file format, known to the developer ahead of time. And for other use cases, the `@linkage` attribute is in most cases not supposed to be used directly but instead wrapped in a macro or otherwise hidden in a higher-level API, this doesn’t seem like a worthwhile change, also it would hardcode the set of supported object file formats. The alternative of using conditional compilation is what is expected to be used for those cases instead:

```swift
#if os(macOS)
@linkage(section: "__DATA,mysection")
#elif os(Linux)
@linkage(section: ".mysection")
#endif
var global: Int = 42
```

### Not providing the low-level control attribute and instead building use case features directly in the compiler

[SE-0385 (Custom Reflection Metadata)](https://github.com/apple/swift-evolution/blob/main/proposals/0385-custom-reflection-metadata.md) is certainly implementable as a high-level language feature in the compiler, but the past review feedback suggested to go into the opposite direction. It’s worth noting that the same applies to a freestanding “linker set” language feature (as opposed to the type-attached metadata described in SE-0385), which could be built as a language feature directly in the compiler, without ever exposing a way to place custom data into custom sections. However, on top of the past review feedback, such a solution would also fail to address the embedded and systems needs to control data placement into sections as described in the Motivation section (e.g. to satisfy a pre-existing contract where a C-written library used in a firmware simply requires some data to be placed in a specific section).

### General const / constexpr modifier-based compile-time evaluation

This proposal introduces guaranteed compile-time evaluation without the need for a new language-level system for it. An alternative approach would be to instead first introduce such a system, for example one that [SE-0359 (Build-Time Constant Values)](https://github.com/apple/swift-evolution/blob/main/proposals/0359-build-time-constant-values.md) describes, then expand the language-level support for this system and the standard library code to make enough code “compile-time evaluable” and then add a custom section control feature on top of that (requiring that @linkage attribute is only used on globals that are marked as “compile-time evaluable”):

```swift
@linkage(section: "__DATA,mysection") var global = 42 // error: requires @const

@linkage(section: "__DATA,mysection") @const var global = 42 // ok
// though consider that even a "42" literal involves stdlib code in the Int struct,
// namely there is an implicit Int.init(integerLiteral:) initializer call
```

While this might appear to be compelling and more explicit about the desired properties of the global, there are concerns about the virality of the `@const` annotation, which are stated as one of the reasons for the rejection of SE-0359. In a case where only a literal is used in the initializer expression, this might not pose as a problem, but it becomes more clear when non-trivial yet still very simple expressions are used:

```swift
@linkage(section: "__DATA,mysection") @const var global = 22 % 5 // ?
// the implementation of Int.init(integerLiteral:) is trivial, but the implementation
// of Int.%(lhs:rhs:) isn't, it calls to Int.%= which in turns performs an overflow
// check -- all of this code would need to fully adopt @const first
```
