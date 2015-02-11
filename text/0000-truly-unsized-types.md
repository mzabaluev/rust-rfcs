- Start Date: 2015-01-22
- RFC PR:
- Rust Issue:

# Summary

Allow opting out of `Sized` for types with statically-sized contents.
Pointers to unsized non-DSTs will be thin, while allocating slots or
copying values of such types is not possible outside unsafe code.

# Motivation

There are cases where borrowed references provide a view to a larger object
in memory than the nominal sized type being referenced. Of interest are:

- Token types for references to data structures whose size is not available as
  a stored value, but determined by parsing the content, such as
  null-terminated C strings. An example is `CStr`, a
  [proposed](https://github.com/rust-lang/rfcs/pull/592) 'thin' dereference
  type for `CString`.
- References to structs as public views into larger contiguously allocated
  objects, the size and layout of which is hidden from the client.

When only immutable references are available on such types, it's possible
to prevent copying out the nominal sized value in safe code, but there is
still potential for unintentional misuse due to values under reference having
a bogus size.
If mutable references are available, there is added trouble with
`std::mem::swap` and similar operations.

# Detailed design

Any type with statically-sized contents can be opted out of `Sized`:

```rust
struct CStr {
    head: libc::c_char  // Contains at least one character...
}

impl !Sized for CStr { }  // ...but there can be more
```

This ensures that a slot cannot be created with type `CStr`, and makes
references to values of such types unusable for copying the value out,
`std::mem::swap`, being the source in `transmute_copy`, etc.
The type cannot be the parameter of `size_of`, so the size of declared
type contents is never made to matter.

Current generic code may use the `Sized` bound on a type, often implicit,
to ensure that a reference can be converted to or from a raw pointer.
To allow unsized types, such code would need to be changed to use conversion
traits like the following:

```rust
pub trait AsPtr<Raw = Self> {
    fn as_ptr(&self) -> *const Raw;
}

pub trait AsMutPtr<Raw = Self> {
    fn as_mut_ptr(&mut self) -> *mut Raw;
}

pub trait FromPtr<Raw = Self> {
    unsafe fn from_ptr<'a>(ptr: *const Raw) -> &'a Self;
}

pub trait FromMutPtr<Raw = Self> {
    unsafe fn from_mut_ptr<'a>(ptr: *mut Raw) -> &'a mut Self;
}

impl<T> AsPtr<T> for T where T: Sized {
    fn as_ptr(&self) -> *const T { self as *const T }
}

impl<T> AsMutPtr<T> for T where T: Sized {
    fn as_mut_ptr(&mut self) -> *mut T { self as *mut T }
}

impl<T> FromPtr<T> for T where T: Sized {
    unsafe fn from_ptr<'a>(ptr: *const T) -> &'a T {
        std::mem::transmute(&*ptr)
    }
}

impl<T> FromMutPtr<T> for T where T: Sized {
    unsafe fn from_mut_ptr<'a>(ptr: *mut T) -> &'a mut T {
        std::mem::transmute(&mut *ptr)
    }
}
```

Unsized types can have these traits implemented manually, or, with added
compiler support, get them with `#[derive(...)]`. The same traits also make it
possible to convert between thin pointers and DSTs where such transformations
can be done losslessly. Suppose that a C string is represented by a DST:

```rust
// Asserted to be null-terminated
pub struct CStr {
    chars: [c_char]
}

impl AsPtr<c_char> for CStr {
    fn as_ptr(&self) -> *const c_char { self.chars.as_ptr() }
}

impl FromPtr<c_char> for CStr {
    unsafe fn from_ptr<'a>(ptr: *const c_char) -> &'a CStr {
        transmute(slice::from_raw_parts(ptr, libc::strlen() as usize))
    }
}
```

# Drawbacks

The change adds further complexity to the type system.

Auditing existing libraries for possibilities to replace the implicit `Sized`
bound with pointer conversion traits will be tedious. The changes will not be
backwards-compatible on the ABI level (which probably no one cares about
in Rust 1.0 timeframe anyway).

# Alternatives

Keep to the current practice of Irrelevantly Sized Types, learning to avoid
trouble by design discipline, documentation, and best practices. The problem
of mutable references can be resolved on a case-by-case basis by providing
an accessor facade for fields that can be safely mutated, and disallowing
direct mutable references to the pseudo-sized type. This feels at odds
with the overall philosophy of Rust, though.

# Unresolved questions

Should pointer conversion traits `AsPtr`, `FromPtr`, etc. as described above,
be enshrined into the core libraries and given compiler support for
`#[derive()]`?
