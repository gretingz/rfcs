- Feature Name: custom_dsts_and_references
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Make it possible to have custom DSTs that know their size and custom reference-like types while preserving the simplicity of &T, &mut T, *const T and *mut T

# Motivation
[motivation]: #motivation

Rust currently has three types of references: ones that point to a `Sized` type (eg. `u8`),
ones that point to a `!Sized` type (eg. `str`) and ones that point to a trait object (eg. `dyn Foo`).

These three references aren't suitable to express some types. For example a `CStr` is `!Sized`,
but it doesn't require a fat pointer, as it can know it's own size since it's null terminated.
C also has Variable-length arrays, which are similar in that they often know their own size, but are `!Sized`.

Types such as a bit vector have the opposite problem, in that they require more information than a fat pointer can contain.
For example a "bit slice" would perhaps look something like this:

```rust

struct BitSlice {
    pointer: *const u8,
    length: usize,
    start_offset: u8,
}

```

The `start_offset` field is required to distinguish between the 8 bits inside a byte.

A more extreme example of this is a crate like ndarray, which provides multidimensional arrays.
A multi-dimensional slice would have to store a lot more data, than a 1D slice.

Currently the only solution is to define custom reference types, but these are a bit clunky to use,
as many traits such as `Index`, `IndexMut`, `Deref`, `Borrow` and `BorrowMut` are impossible implement mainly because they return a reference.

This RFC also has the goal of leaving & and &mut mostly untouched, and instead having regular types with reference-like scemantics.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

This RFC consists of two parts that complement each other but are techically orthogonal. First part is the `KnownSize` trait.
The second part is a collection of traits that are meant to make custom references more ergonomic to use.

In this RFC there are some terms that might be a bit ambiguos, so they are defined here:
* `Sized`: The size and alignment of this type are known at compile time
* `?Sized`: The size and alignment of this type might not be known at compile time
* `!Sized`: The *size* of this type is not known at compile time (but the alignment may be)

## The KnownSize trait

This trait is meant to be implemented by `!Sized` types that know their size without a fat pointer.
The trait has the following signature:

```rust
unsafe trait KnownSize {
    fn get_size(&self) -> usize;
}
```

Implementing this trait for a `!Sized` type causes all references and fat pointers to it to become one word wide.
In places where the length of the fat pointer would have been inspected, `get_size` is called.

This trait is auto-implemented by all Sized types.

#### Example CStr:

```rust
use std::os::raw::c_char;

// Needed for unsafe code
#[repr(transparent)]
// Syntax is the same as for regular unsized structs
struct CStr ([c_char]);

unsafe impl KnownSize for CStr {
    fn get_size(&self) -> usize {
        // Finds the first null character
        // Similarly to implementing drop we have to be careful not to cause infinite recursion 
        let mut ptr = self as *const Self as *const c_char;
        let mut count = 1usize;
        unsafe {
            while *ptr != 0 {
                 count+= 1;
                 ptr = ptr.add(1);
            }
        }
        count
    }
}

static_assert_eq!(size_of::<&CStr>(), size_of::<usize>());
static_assert_eq!(size_of::<*const CStr>(), size_of::<&CStr>());

impl CStr {
    fn get_second_byte(&self) -> Option<c_char> {
        // This calls get_size implicitly
        self.0.get(1)
    }
}
```

# Custom reference traits
These are traits desinged for making _reference-like types_. Reference-like types are just regular types that "behave" like references. They usually look something like this:
```rust
// A lifetime parameter to indicate how long this reference is valid for
struct CustomReference<'a> {
    ptr: *const u8, // Implementation details
    _phantom: PhantomData<&'a u8>, // Required for lifetime covariance
}
```
When using reference-like types, they should be generally moved, instead of borrowed. For example
```rust
fn do_something(ref: &u8){...}
// Would be written as
fn do_something(ref: CustomReference) {...}
// And this
fn do_something(ref: &CustomReference) {...}
//Is a bit like doing
fn do_something(ref: &&u8) {...}
```
The reason for this is explained [here](https://github.com/gretingz/rfcs/new/master/text#). 

## The Reborrow traits

```rust
trait Reborrow {
    type Ref<'a> : IntoImmutable<Immutable = Self::Ref>;
    fn reborrow<'a>(&'a Self) -> Ref<'a>;
}

trait ReborrowMut : Reborrow {
    type RefMut<'a> : IntoImmutable<Immutable = Self::Ref>;
    fn reborrow_mut<'a>(&'a mut Self) -> RefMut<'a>;
}

trait IntoImmutable {
    type Immutable;
    fn into_immutable(self) -> Immutable;
}
```

Reborrow and ReborrowMut are meant to be implemented by references (eg. &str) and by referrants (eg. String). They serve two purposes. One is the Deref-style implicit
coercion from &T to &U, and the other one is reborrowing. IntoImmutable is meant to be implemented by references only.
It converts a mutable reference to an immutable one without shortening the lifetime of the reference.

## The IndexGet and IndexGetMut traits

```rust
pub trait IndexGet<Idx: ?Sized> {
    type Output;
    fn index_get(self, index: Idx) -> Self::Output;
}

pub trait IndexGetMut<Idx: ?Sized> : IndexGet<Idx> {}
```

IndexGet and IndexGetMut are meant to be implemented by references only, and they also return references.
In order to keep rusts slicing syntax somewhat consistent, the following desugaring follows:

Lvalues:
```rust
reference[index] = value; -> reference.index_get(index).deref_set(value);
```
Rvalues:
```rust
let foo = reference[index] -> let foo = *reference.index_get(index) -> let foo = reference.index_get(index).deref_get();
let foo = &reference[index] -> let foo = reference.index_get(index).into_immutable();
// The next one requires the IndexGetMut trait to be implemented
let foo = &mut reference[index] -> let foo = reference.index_get(index);
```

## The DerefGet and DerefSet traits

```rust
pub trait DerefGet {
    type Output;
    fn deref_get(self) -> Self::Output;
}

pub trait DerefSet {
    type Input;
    fn deref_set(self, value: Self::Input);
}
```

DerefGet and DerefSet are syntactic sugar for *value, depending if it's a lvalue or rvalue.

Lvalues:
```rust
*reference = value; -> reference.deref_set(value);
```

Rvalues:
```rust
let foo = *reference -> let foo = reference.deref_get();
```

## Examples:
```rust
// Regular slices
fn do_stuff<'a>(slice: &mut [u8]) -> &mut [u8] {
    let one = &slice[1..2];
    let two = &slice[1..2];
    read_slice(one);
    read_slice(two);
    let three = &mut slice[1..2];
    let (left, right) = three.split_at_mut(1);
    write_slice(left);
    right[0] = left[0];
    let (left, right) = three.split_at_mut(1);
    left
}

// Custom slices
fn do_stuff<'a>(slice: BitSliceMut<'a>) -> BitSliceMut<'a> {
    let one = &slice[1..2];
    let two = &slice[1..2];
    read_slice(one);
    read_slice(two);
    let three = &mut slice[1..2];
    let (left, right) = three.split_at_mut(1);
    write_slice(left);
    right[0] = left[0];
    let (left, right) = three.split_at_mut(1);
    left
}

// Ultimately desugars to
fn do_stuff<'a>(slice: BitSliceMut<'a>) -> BitSliceMut<'a> {
    let one = slice.reborrow().index_get(1..2).into_immutable();
    let two = slice.reborrow().index_get(1..2).into_immutable();
    read_slice(one);
    read_slice(two);
    let three = slice.index_get(1..2);
    let (left, right) = three.reborrow_mut().split_at_mut(1);
    write_slice(left);
    right.index_get(0).deref_set(left.index_get(0).deref_get(0));
    let (left, right) = three.split_at_mut(1);
    left
}
```


# Full Example Implementations

## CStr 

```rust
use std::os::raw::c_char;

// Needed for unsafe code
#[repr(transparent)]
// Syntax is the same as for regular unsized structs
struct CStr ([c_char]);

unsafe impl KnownSize for CStr {
    fn get_size(&self) -> usize {
        // Finds the first null character
        // Similarly to implementing drop we have to be careful not to cause infinite recursion 
        let mut ptr = self as *const Self as *const c_char;
        let mut count = 1usize;
        unsafe {
            while *ptr != 0 {
                 count+= 1;
                 ptr = ptr.add(1);
            }
        }
        count
    }
}

// A CStr slice is a [c_char] slice, so we can use traditional methods
impl<Idx: std::slice::SliceIndex> std::ops::Index<Idx> for CStr {
    type Output = [c_char];
    fn index(&self, index: Idx) -> &[c_char] {
       &self.0[index]
    }
}

impl<Idx: std::slice::SliceIndex> std::ops::IndexMut<Idx> for CStr {
    fn index_mut(&mut self, index: Idx) -> &mut [c_char] {
       &mut self.0[index]
    }
}

impl Deref for CStr {
    type Target = [c_char];
    fn deref(&self) -> &[c_char] {
        &self.0
    }
}

impl DerefMut for CStr {
    fn deref_mut(&mut self) -> &mut [c_char] {
        &mut self.0
    }
}
```

## BitVec, BitSlice and BitRef

```rust
struct BitVec {
    ...
}

#[derive(IntoImmutable)] // Implements IntoImmutable<Immutable = Self>
struct BitSlice<'a> {
    ...
    _phantom: std::marker::PhantomData<&'a bool>,
}

#[derive(IntoImmutable)]
struct BitRef<'a> {
    ...
    _phantom: std::marker::PhantomData<&'a bool>,
}

// BitVec can be implicitly turned into a BitSlice, so it implements Reborrow
impl Reborrow for BitVec {
    type Ref<'a> = BitSlice<'a>;
    fn reborrow<'a>(&'a self) -> BitSlice<'a> {
        todo!()
    }
}

// BitSlice can be also reborrowed
impl<'a> Reborrow for BitSlice<'a> {
    type Ref<'_> = BitSlice<'a>;
    // Because immutable references can alias
    // we can return a longer lifetime
    fn reborrow<'_>(&'_ self) -> BitSlice<'a> {
        todo!()
    }
}

// BitRef
impl<'a> Reborrow for BitRef<'a> {
    type Ref<'_> = BitRef<'a>;
    fn reborrow<'_>(&'_ self) -> BitRef<'a> {
        todo!()
    }
}

// BitSlice can be sliced
impl<'a> IndexGet<std::ops::Range> for BitSlice<'a> {
    type Output = BitSlice<'a>;
    fn index_get(self, index: std::ops::Range) -> BitSlice<'a> {
        todo!()
    }
}

// Or you can just get a single bit
impl<'a> IndexGet<usize> for BitSlice<'a> {
    type Output = BitRef<'a>;
    fn index_get(self, index: usize) -> BitRef<'a> {
        todo!()
    }
}

// BitRef can be used to read a single bit
impl DerefGet<usize> for BitRef<'_> {
    type Output = bool;
    fn deref_get(self, index: usize) -> bool {
        todo!()
    }
}

// Now for the mutable versions
struct BitSliceMut<'a> {
    ...
    _phantom: std::marker::PhantomData<&'a mut bool>,
}

struct BitRefMut<'a> {
    ...
    _phantom: std::marker::PhantomData<&'a mut bool>,
}


// BitVec can be mutably borrowed into a BitSliceMut
impl ReborrowMut for BitVec {
    type RefMut<'a> = BitSliceMut<'a>;
    fn reborrow<'a>(&'a self) -> BitSliceMut<'a> {
        todo!()
    }
}

// BitSliceMut can be mutably reborrowed
impl ReborrowMut for BitSliceMut {
    type RefMut<'a> = BitSliceMut<'a>;
    fn reborrow_mut<'a>(&mut 'a self) -> BitSliceMut<'a> {
        todo!()
    }
}

// But also immutably!
impl Reborrow for BitSliceMut<'_> {
    type Ref<'a> = BitSlice<'a>;
    fn reborrow<'a>(&'a self) -> BitSlice<'a> {
        todo!()
    }
}

// Reborrow doesn't preserve the full lifetime of BitSliceMut so IntoImmutable is needed
impl<'a> IntoImmutable for BitSliceMut<'a> {
    type Immutable = BitSlice<'a>;
    fn into_immutable(self) -> BitSlice<'a> {
        todo!()
    }
}

// BitSliceMut can be sliced
impl<'a> IndexGet<std::ops::Range> for BitSliceMut<'a> {
    type Target = BitSliceMut<'a>;
    fn index_get(self, index: std::ops::Range) -> BitSliceMut<'a> {
        todo!()
    }
}

// Also mutably
impl IndexGetMut<std::ops::Range> for BitSliceMut {}

// You can take a single bit
impl<'a> IndexGet<usize> for BitSliceMut<'a> {
    type Output = BitRefMut<'a>;
    fn index_get(self, index: usize) -> BitRefMut<'a> {
        todo!()
    }
}

// Also mutably
impl IndexGetMut<usize> for BitSliceMut {}

// BitRefMut can be mutably reborrowed
impl ReborrowMut for BitRefMut {
    type RefMut<'a> = BitRefMut<'a>;
    fn reborrow_mut<'a>(&mut 'a self) -> BitSliceMut<'a> {
        todo!()
    }
}

// And also immutably
impl Reborrow for BitRefMut {
    type Ref<'a> = BitRef<'a>;
    fn reborrow<'a>(&'a self) -> BitRef<'a> {
        todo!()
    }
}

// Reborrow doesn't preserve the full lifetime of BitRefMut so IntoImmutable is needed
impl<'a> IntoImmutable for BitRefMut<'a> {
    type Immutable = BitRef<'a>;
    fn into_immutable(self) -> BitRef<'a> {
        todo!();
    }
}

// BitRefMut can be used to read a single bit
impl DerefGet<usize> for BitRefMut<'_> {
    type Output = bool;
    fn deref_get(self) -> bool {
        todo!()
    }
}

// or set a single bit
impl DerefSet<usize> for BitRefMut<'_> {
    type Input = bool;
    fn deref_set(self, input: bool) {
        todo!()
    }
}

// Finally, split_at and split_at_mut
impl<'a> BitSlice<'a> {
    fn split_at(self, index: usize) -> (Self, Self) {
        todo!()
    }
}

impl<'a> BitSliceMut<'a> {
    fn split_at_mut(self, index: usize) -> (Self, Self) {
        todo!()
    }
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## How do KnownSize and trait objects interact?

This RFC purposefully keeps trait objects somewhat magical. De-magifying box and trait objects are intended to be in the scope of another RFC. Their current interaction looks like this:
```rust
trait Foo {}
// Can't implement KnownSize, because d still requires a vtable pointer
struct Dynamic {
    d: dyn Foo;
}

// Because CStr is KnownSize, it can be used as a trait object
impl Foo for CStr {}
```

## Why doesn't KnownSize tell anything about alignment?
The only realistic situation (that I can think of) where a type's alignment would not be known without a thin pointer, would be something like:
```rust
struct DynWrapper {
    vtable_ptr: *const u8,
    trait_object: [u8],
}

impl KnownSize for DynWrapper {...}
```
Such a type would be impossible to be a member of another struct like so
```rust
#[repr(C)]
struct Foo {
    some_number: usize,
    wrapper: DynWrapper,
}
```
If DynWrapper has a stricter alignment than usize, it's not really possible to tell how much padding there is between ```some_number``` and ```wrapper``` (without a fat pointer).
This would be possible to solve by having wrapper be the first element of the struct. Having two DynWrappers in a struct is not possible.

## Rules for adding reborrows:

The rules should be kept vauge and restrictive to allow for future changes, but here are anyways my naïve set of rules for reborrowing.

Zeroth rule is that no reborrowing should occur if not neccesary.

First rule: A reborrow should be attempted if there is a method that expects a wrong type ("Expected T found U").

Second rule: If a reborrow that was attemped because of the first rule fails because of a lifetime error, the reborrow should be attempted to be replaced by an into_immutable.

Third rule: If a value is moved, and then used again, try to replace the previous move with a reborrow.

## Auto impls:

```rust
// Standard references
impl<'a, T: ?Sized> IntoImmutable for &'a T {
    type Immutable = &'a T;
    ...
}

impl<'a, T: ?Sized> IntoImmutable for &'a mut T {
    type Immutable = &'a T;
    ...
}

impl<'a, T> DerefGet for &'a T {
    type Output = T;
    ...
}

impl<'a, T> DerefGet for &'a mut T {
    type Output = T;
    ...
}

impl<'a, T> DerefSet for &'a mut T {
    type Input = T;
    ...
}

//Deref retrofitting
impl<T: Deref + ?Sized> Reborrow for T {
    type Ref<'a> = &'a Self::Target;
    ...
}

impl<T: DerefMut + ?Sized> Reborrow for T {
    type Ref<'a> = &'a Self::Target;
    ...
}

impl<T: DerefMut + ?Sized> ReborrowMut for T {
    type RefMut<'a> = &'a mut Self::Target;
    ...
}

//Index retrofitting

impl<'a, Idx, T: ?Sized + Index<Idx>> IndexGet for &'a T {
    type Output = &'a T::Output;
    ...
}

impl<'a, Idx, T: ?Sized + IndexMut<Idx>> IndexGet for &'a mut T {
    type Output = &mut 'a T::Output;
    ...
}

impl<'a, Idx, T: ?Sized + IndexMut<Idx>> IndexGetMut for &'a mut T {}
```

The following specialized impl is more correct, but may not be neccesary.

```rust
impl<'a, T: ?Sized> Reborrow for &'a T {
    type Ref<'_> = &'a T;
    ...
}
```

# Drawbacks
[drawbacks]: #drawbacks

- Implementing custom references is very verbose
- Future improvements to lifetime ergonomics might not be retrofittable. This RFC is also largely designed around NLL.
- More complexity to the language
- Potential to abuse Reborrow and ReborrowMut as a makeshift ```Copy``` trait.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This is an alternative to both #2594 and #2953, so here is a comparison between them:

## Advantages over #2594
- Reference-like types allow more flexibility for metadata. For example a reference-like type can contain a Box, Rc, Arc, Mutex, whatever. 
- Reference-like types allow more specific rules for references than regular references.
For example let's say we want to implement a BitSlice that can be shared between threads. Due to the possibility of mutable slices that alias at the byte level,
we have to use some kind of atomic operations. We can use relaxed operations, but DerefSet would require a compare-exchange style operations.
This may be expensive on some archs, and we'd rather not do that.
If we have only one thread that is writing, we can simply mark ```BitSliceMut``` and ```BitRefMut``` as ```!Send``` and use a Relaxed load and store for DerefSet.
- It's possible to make reference-like types with safe rust
- Reference-like types only require the compiler to add reborrows in various places.

## Disadvantages over #2594
- Reference-like types are much more verbose to implement
- Some types may lose their meaning with regular references. For example say we have the following:
```rust
struct 2DArray {
    data: [f64],
}

struct 2DArraySlice {
    ptr: *const f64,
    len_x: usize,
    len_y: usize,
    stride: usize,
}
```
And wanted to return a ```Box<2DArray>```.  Box has no way of knowing that it should use 2DArraySlice as the pointer.
Hence information about the dimensions of 2DArray is lost.

## Advantages/Disadvantages over #2953
- Forces library authors to have reference-like types for single elements.
- Allows library authors to overload the ```& v[i]``` and ```&mut v[i]``` syntax.

## Why have KnownSize? Can't you use reference-like types for CStr etc?

1. It is for a very specific subset of types that
2. Have clearly defined memory layouts and
3. Are very common in various C APIs

## Why is Reborrow and IntoImmutable a thing? Why do you have to move custom references?
[why-reborrow]: #why-reborrow

```rust
// Suppose we have something like the following
fn split_at_mut(slice: &mut [u8], index: usize) -> (&mut [u8], &mut [u8])
// And we wanted to use custom slices
// We can't do something like
fn split_at_mut<'a>(slice: &'_ mut CustomSliceMut<'a>, index: usize) -> (CustomSliceMut<'a>, CustomSliceMut<'a>) {...}
// Because now the lifetime of the mutable borrow is not tied to anything and so you can do this
fn make_aliasing_mut_references(slice: CustomSliceMut) {
    let (left1, right1) = split_at_mut(slice, 1);
    let (left2, right2) = split_at_mut(slice, 1);
}
// We could do this
fn split_at_mut<'a>(slice: &'a mut CustomSliceMut<'_>) -> (CustomSliceMut<'a>, CustomSliceMut<'a>) {...}
// But now it's not possible to do this
fn just_the_left_part(slice: CustomSliceMut<'a>) -> CustomSliceMut<'a> {
   let (ret, _) = split_at_mut(slice, 1);
   ret
}
// The only solution is to move the reference
fn split_at_mut<'a>(slice: CustomSliceMut<'a>) -> (CustomSliceMut<'a>, CustomSliceMut<'a>) {...}
// But now that the reference gets moved, it's not possible to do this
fn baz<'a>(slice: CustomSliceMut<'a>) -> CustomSliceMut<'a> {
    let (left1, _) = split_at_mut(slice, 1);
    do_something_with_slice(left1);
    let (left2, _) = split_at_mut(slice, 1);
    left2
}
// Here, left1 didn't really need the full 'a lifetime, so we need a way to strategically shorten the lifetime of reference-like types
// Specifically, we need the ReborrowMut trait
trait ReborrowMut : Reborrow {
    type RefMut<'a> : ?Sized + IntoImmutable<Immutable = Self::Ref>;
    fn reborrow_mut<'a>(&'a mut self) -> RefMut<'a>;
}
// Let's look how it is implemented for CustomSliceMut
// Here 'a is the lifetime this reference is valid for
impl<'a> ReborrowMut for CustomSliceMut<'a> {
    //'b is a GAT lifetime parameter
    type RefMut<'b> = CustomSliceMut<'b>;
    //'c is the lifetime of the reborrow
    fn reborrow_mut<'c>(&'c mut self) -> CustomSliceMut<'c> {
        todo!()
    }
}
// In our example the compiler strategically places a reborrow_mut
fn baz<'a>(slice: CustomSliceMut<'a>) -> CustomSliceMut<'a> {
    // The lifetime paramter of left1 is now shorter
    let (left1, _) = split_at_mut(slice.reborrow_mut(), 1);
    do_something_with_slice(left1);
    // Reborrow ends here
    let (left2, _) = split_at_mut(slice, 1);
    left2
}
```
Reborrow is similar to ReborrowMut. IntoImmutable is used to convert a mutable reference into an immutable one without shortening the lifetime parameter.

# Prior art
[prior-art]: #prior-art

## DST:

- [FAMs in C](https://en.wikipedia.org/wiki/Flexible_array_member)
- [FAMs in C++](https://htmlpreview.github.io/?https://github.com/ThePhD/future_cxx/blob/master/papers/d1039.html) (unfinished proposal)
- Other RFCs
  - [mzabaluev's Version](https://github.com/rust-lang/rfcs/pull/709)
  - [strega-nil's Old Version](https://github.com/rust-lang/rfcs/pull/1524)
  - [strega-nil's New Version](https://github.com/rust-lang/rfcs/pull/2594)
  - [japaric's Pre-RFC](https://github.com/japaric/rfcs/blob/unsized2/text/0000-unsized-types.md)
  - [mikeyhew's Pre-RFC](https://internals.rust-lang.org/t/pre-erfc-lets-fix-dsts/6663)
  - [MicahChalmer's RFC](https://github.com/rust-lang/rfcs/pull/9)
  - [nrc's Virtual Structs](https://github.com/rust-lang/rfcs/pull/5)
  - [Pointer Metadata and VTable](https://github.com/rust-lang/rfcs/pull/2580)
  - [Syntax of ?Sized](https://github.com/rust-lang/rfcs/pull/490)

## Reborrows:
 - [mzabaluev's Reborrows](https://github.com/rust-lang/rfcs/pull/2364)
 - [Issue on Reborrows](https://github.com/rust-lang/rfcs/issues/1403)
 
## Custom references:
 - [carbotaniuman's IndexGet and IndexSet](https://github.com/rust-lang/rfcs/pull/2953)

## Inspiration:
This RFC is largely inspired by python's indexing operator that "just works" with libraries such as Tensorflow.

# Unresolved questions
[unresolved-questions]: #unresolved-questions
- What are some actually good names for the traits? What module should they go in?
- Should more than one ```KnownSize + !Sized``` be allowed in a struct?
- Should ```&*v``` and ```&mut *v``` be desugared to ```v.reborrow()``` and ```v.reborrow_mut()``` respectively?
- What exactly happens if you implement ```KnownSize``` for a ```Sized``` type or a trait object? Does it error, do nothing or something else?
  - What happens if you implement ```KnownSize``` for a struct with a ```?Sized``` generic parameter?

# Future possibilities
[future-possibilities]: #future-possibilities

## Polonius

With Polonius the BorrowMut trait is fundamentally too restrictive in some cases.
With most reference-like types it doesn't matter if the reference that was reborrowed is moved or destroyed, etc.
It just matters if it's used to access data.

### Solution

Null-lifetimes and borrow contract specification.

Null-lifetimes are a way to express the idea "The borrow that this lifetime parameter represents may be expired". For example one could write
```rust
impl Drop for CustomRefMut<'null> {
    fn drop(&mut self) {
        ...
    }
}
```
And drop would not be able to access any field with CustomRef's first lifetime parameter.

Borrow contract specialization is a way to express how long different lifetimes should be borrowed.
In our case we have a CustomRefMut and we want to "borrow" the lifetime for longer than the actual CustomRefMut.
```rust
impl<'a> ReborrowMut for CustomRefMut<'a> {
    type RefMut<'b> = CustomRefMut<'b>;
    fn reborrow_mut<'c>(&{'_, 'c} mut Self) -> CustomRefMut<'c>;
}
```
Here the mutable borrow has a lifetime ```'c```, but we can specify that it doesn't apply to the whole borrow.
```'_``` means that the borrowed CustomRefMut must exist at least for the duration of this function.
Specifically, we only need a short borrow of CustomRefMut, but a longer one for the lifetime parameter.
After reborrow_mut is called, methods that don't use the ```'a``` lifetime parameter can be called, even if they overlap with the 'a lifetime parameter.
For example the reference that was borrowed could be dropped.
```rust
fn bar<'a>(ref: CustomRefMut<'a>) -> CustomRefMut<'a> {
    let reborrowed = {
        let ref_2 = ref; // ref is moved here
        reborrow_mut(ref_2) // ref_2 is reborrowed
        // ref_2 is dropped. Drop doesn't use the lifetime parameter 'a, 
    } // Reborrowed still has lifetime 'a
    reborrowed
}
```

Some previous discussion [here](https://internals.rust-lang.org/t/proposal-about-expired-references/11675)
