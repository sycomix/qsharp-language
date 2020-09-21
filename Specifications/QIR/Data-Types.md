## Data Type Representation

We define LLVM representations for a variety of classical and quantum data types.

In most cases, we represent complex types as pointers to opaque LLVM structure
types.
This allows each target to provide a structure definition appropriate for that
target.

### Reference Counting

In QIR, all types represented as pointers, other than qubits, are reference-counted.
All types follow the same pattern:

- Runtime routines that create a new instance always initialize the instance
  with a reference count of 1.
- Each type has a `_reference` runtime routine that increments the reference
  count of an instance and an `_unreference` routine that decrements the
  reference count.
- The `_unreference` routine will release the instance if the reference count
  is decremented to zero.
- The `_unreference` routine should accept a null instance pointer and simply
  ignore the call if the pointer is null. This allows us to avoid null checks
  strewn through the QIR, with the attendant plethors of LLVM basic blocks.

A target is free to provide some other mechanism for garbage collection and
treat calls to these runtime functions as hints or as simple no-ops.

### Unit

For source languages that include a `Unit` type, the representation of this type
in LLVM depends on its usage.
If used as a return type for a callable, it should be translated into an LLVM
`void` function.

If it is used as a value, for instance as a user-defined type or as an element of
a tuple, a tuple type with no contained elements should be used.
In this case, the one possible value of `Unit`, `()`, should be represented as a
null tuple pointer.

### Simple Types

The simple types are those whose values are fixed-size and do not contain pointers.
They are represented as follows:

| Type     | LLVM Representation        | Comments |
|----------|----------------------------|----------|
| `Int`    | `i64`                      |          |
| `Double` | `double`                   |          |
| `Bool`   | `i1`                       | 0 is false, 1 is true. |
| `Result` | `%Result*`                 | `%Result` is an opaque type. |
| `Pauli`  | `%Pauli = {i2}`            | 0 is PauliI, 1 is PauliX, 3 is PauliY, and 2 is PauliZ. |
| `Qubit`  | `%Qubit*`                  | `%Qubit` is an opaque type. |
| `Range`  | `%Range = {i64, i64, i64}` | In order, these are the start, step, and end of the range. When passed as a function argument or return value, ranges should be passed by value. |

The following global constants are defined for use with the `%Result` and `%Pauli` types:

```LLVM
@ResultZero = external global %Result*
@ResultOne = external global %Result*

@PauliI = constant i2 0
@PauliX = constant i2 1
@PauliY = constant i2 -1
@PauliZ = constant i2 -2
```

The following utility functions are provided by the classical runtime to support
simple types:

| Function                          | Signature                | Description |
|-----------------------------------|--------------------------|-------------|
| __quantum__rt__int_power          | `i64(i64, i64)`          | Returns the first integer raised to the second integer power. This function is required because the LLVM `llvm.powi`
intrinsic does not allow `i64` exponents. |
| __quantum__rt__result_equal       | `i1(%Result*, %Result*)` | Returns true if the two results are the same, and false if they are different. |
| __quantum__rt__result_reference   | `void(%Result*)`         | Increments the reference count of a Result pointer. |
| __quantum__rt__result_unreference | `void(%Result*)`         | Decrements the reference count of a Result pointer and releases the result if appropriate. |

### Strings

Strings are represented as pointers to an opaque type.

| Type   | LLVM Representation |
|--------|---------------------|
| String | `%String*`          |

The following utility functions should be provided by the classical runtime to support
strings:

| Function                          | Signature                      | Description |
|-----------------------------------|--------------------------------|-------------|
| __quantum__rt__string_create      | `%String*(i32, [0 x i8])`      | Creates a string from an array of UTF-8 bytes. |
| __quantum__rt__string_reference   | `void(%String*)`               | Indicates that a new reference has been added. |
| __quantum__rt__string_unreference | `void(%String*)`               | Indicates that an existing reference has been removed and potentially releases the string. |
| __quantum__rt__string_concatenate | `%String*(%String*, %String*)` | Creates a new string that is the concatenation of the two argument strings. |
| __quantum__rt__string_equal       | `i1(%String*, %String*)`       | Returns true if the two strings are equal, false otherwise. |

The following utility functions support converting values of other types to strings.
In every case, the returned string is allocated on the heap; the string can't be
allocated by the caller because the length of the string depends on the actual value.

| Function                         | Signature            | Description |
|----------------------------------|----------------------|-------------|
| __quantum__rt__int_to_string     | `%String*(i64)`      | Returns a string representation of the integer. |
| __quantum__rt__double_to_string  | `%String*(Double)`   | Returns a string representation of the double. |
| __quantum__rt__bool_to_string    | `%String*(i1)`       | Returns a string representation of the Boolean. |
| __quantum__rt__result_to_string  | `%String*(%Result)`  | Returns a string representation of the result. |
| __quantum__rt__pauli_to_string   | `%String*(%Pauli)`   | Returns a string representation of the Pauli. |
| __quantum__rt__qubit_to_string   | `%String*(%Qubit)`   | Returns a string representation of the qubit. |
| __quantum__rt__range_to_string   | `%String*(%Range)`   | Returns a string representation of the range. |
| __quantum__rt__bigint_to_string  | `%String*(%BigInt*)` | Returns a string representation of the big integer. |

### Big Integers

Unlimited-precision integers, also known as "big integers", are represented
as pointers to an opaque type.

| Type   | LLVM Representation |
|--------|---------------------|
| BigInt | `%BigInt*`          |

The following utility functions are provided by the classical runtime to support
big integers.

| Function                          | Signature                      | Description |
|-----------------------------------|--------------------------------|-------------|
| __quantum__rt__bigint_create_i64  | `%BigInt*(i64)`                | Creates a big integer with the specified initial value. |
| __quantum__rt__bigint_create_array | `%BigInt*(i32, [0 x i8])`    | Creates a big integer with the initial value specified by the `i8` array. The 0-th element of the array is the highest-order byte, followed by the first element, etc. |
| __quantum__rt__bigint_reference   | `void(%BigInt*)`               | Indicates that a new reference has been added. |
| __quantum__rt__bigint_unreference | `void(%BigInt*)`               | Indicates that an existing reference has been removed and potentially releases the big integer. |
| __quantum__rt__bigint_negate      | `%BigInt*(%BigInt*)`           | Returns the negative of the big integer. |
| __quantum__rt__bigint_add         | `%BigInt*(%BigInt*, %BigInt*)` | Adds two big integers and returns their sum. |
| __quantum__rt__bigint_subtract    | `%BigInt*(%BigInt*, %BigInt*)` | Subtracts the second big integer from the first and returns their difference. |
| __quantum__rt__bigint_multiply    | `%BigInt*(%BigInt*, %BigInt*)` | Multiplies two big integers and returns their product. |
| __quantum__rt__bigint_divide      | `%BigInt*(%BigInt*, %BigInt*)` | Divides the first big integer by the second and returns their quotient. |
| __quantum__rt__bigint_modulus     | `%BigInt*(%BigInt*, %BigInt*)` | Returns the first big integer modulo the second. |
| __quantum__rt__bigint_power       | `%BigInt*(%BigInt*, i32)`      | Returns the big integer raised to the integer power. |
| __quantum__rt__bigint_bitand      | `%BigInt*(%BigInt*, %BigInt*)` | Returns the bitwise-AND of two big integers. |
| __quantum__rt__bigint_bitor       | `%BigInt*(%BigInt*, %BigInt*)` | Returns the bitwise-OR of two big integers. |
| __quantum__rt__bigint_bitxor      | `%BigInt*(%BigInt*, %BigInt*)` | Returns the bitwise-XOR of two big integers. |
| __quantum__rt__bigint_bitnot      | `%BigInt*(%BigInt*)`           | Returns the bitwise complement of the big integer. |
| __quantum__rt__bigint_shiftleft   | `%BigInt*(%BigInt*, i64)`      | Returns the big integer arithmetically shifted left by the integer amount of bits. |
| __quantum__rt__bigint_shiftright  | `%BigInt*(%BigInt*, i64)`      | Returns the big integer arithmetically shifted right by the integer amount of bits. |
| __quantum__rt__bigint_equal       | `i1(%BigInt*, %BigInt*)`       | Returns true if the two big integers are equal, false otherwise. |
| __quantum__rt__bigint_greater     | `i1(%BigInt*, %BigInt*)`       | Returns true if the first big integer is greater than the second, false otherwise. |
| __quantum__rt__bigint_greater_eq  | `i1(%BigInt*, %BigInt*)`       | Returns true if the first big integer is greater than or equal to the second, false otherwise. |

### Tuples and User-Defined Types

Tuple data, including values of user-defined types, is represented as an
LLVM structure type.
The structure type will contain a reference count as a standard header,
followed by the tuple fields.

Because the `%TupleHeader` type actually appears as part of the structure, LLVM will
not allow it to be an opaque type.
Thus, QIR defines an LLVM type for the tuple header:

```LLVM
%TupleHeader = type { i32 }
```

For instance, a tuple containing two integers, `(Int, Int)`, would be represented in
LLVM as `type {%TupleHeader, i64, i64}`.

Tuple handles are simple pointers to the underlying data structure.
When passed to a callable function, tuples should always be passed by reference,
even if the source language treats them as immutable.

In some situations, a pointer to a tuple of unknown or variable type may be required.
To satisfy LLVM's strong typing in such a situation, tuple pointers are passed as
pointers to the initial tuple header field; that is, as a `%TupleHeader*`.
These pointers should be cast to pointers to the correct data structures by the
receiving code.
For instance, this convention is used for callable wrapper functions; see
[below](#callable-values-and-wrapper-functions).

Many languages provide immutable tuples, along with operators that allow a modified
copy of an existing tuple to be created.
In QIR, this is represented by creating a new copy of the existing tuple and then
modifying the newly-created tuple in place.
If the compiler knows that the existing tuple is not used after the modification,
it is possible to avoid the copy and modify the existing tuple in place.

The following utility functions are provided by the classical runtime to support
tuples and user-defined types:

| Function                         | Signature             | Description |
|----------------------------------|-----------------------|-------------|
| __quantum__rt__tuple_create      | `%TupleHeader*(i64)`  | Allocates space for a tuple requiring the given number of bytes and sets the reference count to 1. |
| __quantum__rt__tuple_reference   | `void(%TupleHeader*)` | Indicates that a new reference has been added. |
| __quantum__rt__tuple_unreference | `void(%TupleHeader*)` | Indicates that an existing reference has been removed and potentially releases the tuple. |

### Arrays

Array data is represented as a pointer to an opaque LLVM structure, `%Array`.

Because LLVM does not provide any mechanism for type-parameterized functions,
runtime library routines that provide access to array elements return byte
pointers that the calling code must `bitcast` to the appropriate type before
using.
When creating an array, the size of each element in bytes must be provided.

Many languages provide immutable arrays, along with operators that allow a modified
copy of an existing array to be created.
In QIR, this is implemented by creating a new copy of the existing arrays on the
heap, and then modifying the newly-created arrays in place.
In some cases, if the compiler knows that the existing arrays is not used after the
creation of the modified copy, it is possible to avoid the copy and modify the
existing arrays as long as there are no other references to the arrays.

There are two special operations on arrays:

- An array *slice* is specified by providing a dimension to slice on and a `%Range` to
  slice with.
- An array *projection* is specified by providing a dimension to project on and an `i64`
  index value to project to. The resulting array has one fewer dimension than the original
  array, and is the segment of the original array with the given dimension fixed to the
  given index value.

Both slicing and projecting are implemented by creating a new `%Array*` with
updated dimension count, offset, stride, and length information.
Runtime library implementations may optimize by initially sharing data between
the slice or projection and the original array and implementing a copy-on-write
strategy to minimize data copying.

The following utility functions are provided by the classical runtime to support
arrays:

| Function                         | Signature                            | Description |
|----------------------------------|--------------------------------------|-------------|
| __quantum__rt__array_create_1d   | `%Array* void(i32, i64)`             | Creates a new 1-dimensional array. The `i32` is the size of each element in bytes. The `i64` is the length of the array. The bytes of the new array should be set to zero. |
| __quantum__rt__array_copy        | `%Array*(%Array*)`                   | Returns a new array which is a copy of the passed-in `%Array*`. |
| __quantum__rt__array_concatenate | `%Array*(%Array*, %Array*)`          | Returns a new array which is the concatenation of the two passed-in arrays. |
| __quantum__rt__array_get_length  | `i64(%Array*, i32)`                  | Returns the length of a dimension of the array. The `i32` is the zero-based dimension to return the length of; it must be 0 for a 1-dimensional array. |
| __quantum__rt__array_get_element_ptr_1d | `i8*(%Array*, i64)`           | Returns a pointer to the element of the array at the zero-based index given by the `i64`. |
| __quantum__rt__array_slice       | `%Array*(%Array*, i32, %Range)`      | Creates and returns an array that is a slice of an existing array. The `i32` indicates which dimension the slice is on, which must be 0 for a 1-dimensional array. The `%Range` specifies the slice. |
| __quantum__rt__array_reference   | `void(%Array*)`                      | Indicates that a new reference has been added. |
| __quantum__rt__array_unreference | `void(%Array*)`                      | Indicates that an existing reference has been removed and potentially releases the array. |

The following utility functions are provided if multidimensional array support is enabled:

| Function                         | Signature                            | Description |
|----------------------------------|--------------------------------------|-------------|
| __quantum__rt__array_create      | `%Array* void(i32, i32, ...)`        | Creates a new array. The first `i32` is the size of each element in bytes. The second `i32` is the dimension count. The variable arguments should be a sequence of `i64`s contains the length of each dimension. The bytes of the new array should be set to zero. |
| __quantum__rt__array_get_dim     | `i32(%Array*)`                       | Returns the number of dimensions in the array. |
| __quantum__rt__array_get_element_ptr | `i8*(%Array*, ...)`              | Returns a pointer to the indicated element of the array. The variable arguments should be a sequence of `i64`s that are the indices for each dimension. |
| __quantum__rt__array_project     | `%Array*(%Array*, i32, i64)`         | Creates and returns an array that is a projection of an existing array. The `i32` indicates which dimension the projection is on, and the `i64` specifies the specific index value to project. The returned `Array*` will have one fewer dimension than the existing array. The new array may share data with the existing array. |

---
_[Back to index](README.md)_