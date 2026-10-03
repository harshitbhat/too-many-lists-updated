# IntoIter

Collections are iterated in Rust using the *Iterator* trait. It's a bit more
complicated than `Drop`:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

The new kid on the block here is `type Item`. This is declaring that every
implementation of Iterator has an *associated type* called Item. In this case,
this is the type that it can spit out when you call `next`.

> An associated type in Rust is a placeholder type defined within a trait. It allows a trait to use a specific type in its method signatures, while leaving the exact choice of that type up to the struct or enum that actually implements the trait.
>
> They are primarily used to keep trait definitions clean and prevent the proliferation of generic type parameters.
> 
> You might wonder why we use `trait Iterator { type Item; }` instead of a generic trait like `trait Iterator<T>`. The choice comes down to how many times a trait can be implemented for a single type.
> 
> When you use a generic type parameter (`trait MyTrait<T>`), you can implement that trait multiple times for the same struct, as long as `T` is different each time.
>
> For example, the `From<T>` trait uses generics because a single struct might need to be created from a `String`, `i32` or a `bool` and we have to specify the types everywhere. 
>
> ```rust,ignore
> impl From<String> for MyStruct { ... }
> impl From<i32> for MyStruct { ... }
> ```
>
> When we use an associated type, you can only implement the trait once for a given struct.
>
> The compiler already knows there's only one type it could possibly be, so you write: `fn process_data<I>(iter: I) where I: Iterator`
> 

The reason Iterator yields `Option<Self::Item>` is because the interface
coalesces the `has_next` and `get_next` concepts. When you have the next value,
you yield
`Some(value)`, and when you don't you yield `None`. This makes the
API generally more ergonomic and safe to use and implement, while avoiding
redundant checks and logic between `has_next` and `get_next`. Nice!

> In many languages, interacting with an iterator requires two separate methods:
> 
> - `has_next()`: Returns a boolean (`true` or `false`) to tell you if there are any items left.
> - `get_next()`: Actually fetches the next item.
>
> ```java,ignore
> // Java example
> while (iterator.hasNext()) {        // 1. Check if it exists
>     String item = iterator.next();  // 2. Fetch it
>    System.out.println(item);
> }
> ```
>
> Rust merges the "check" (`has_next`) and the "fetch" (`get_next`) into a single method: `next()`.
>
> Because it returns an `Option<Self::Item>`, the single method can communicate both pieces of information at exactly the same time:
>
> - Returns `Some(value)`: "Yes, there is a next item, and here it is."
> - Returns `None`: "No, there are no more items left."
> 
> ```rust,ignore
> // Rust example, xhecks and fetches simultaneously
> while let Some(item) = iter.next() { 
>     println!("{}", item);
> }
> ```


Sadly, Rust has nothing like a `yield` statement (yet), so we're going to have to
implement the logic ourselves. Also, there's actually 3 different kinds of
iterator each collection should endeavour to implement:

* IntoIter - `T`
* IterMut - `&mut T`
* Iter - `&T`

We actually already have all the tools to implement
IntoIter using List's interface: just call `pop` over and over. As such, we'll
just implement IntoIter as a newtype wrapper around List:


```rust ,ignore
// Tuple structs are an alternative form of struct,
// useful for trivial wrappers around other types.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // access fields of a tuple struct numerically
        self.0.pop()
    }
}
```

> **Breakdown of the above code** 
> 
> ```rust,ignore
> pub struct IntoIter<T>(List<T>);
> ```
>
> This is a tuple struct (a struct where fields don't have names, only index numbers). It acts as a wrapper around your `List<T>`.
>
> **Why wrap it? Why not just implement `Iterator` directly on `List<T>`?**
>
> In Rust, collections typically offer three different ways to iterate:
>
> - `into_iter()`: Yields owned values (`T`), consuming the list.
> - `iter()`: Yields shared references (`&T`), leaving the list intact.
> - `iter_mut()`: Yields mutable references (`&mut T`), allowing you to modify the list in place.
>
> Because a single struct can only implement Iterator once, you need three separate structs to represent these three different modes of iteration. 
>
> IntoIter is the standard name for the struct that handles the first mode.
>
> ```rust,ignore
> impl<T> List<T> {
>    pub fn into_iter(self) -> IntoIter<T> {
>        IntoIter(self)
>    }
> }
> ```
>
> This method transitions your List into an iterator.
> - Notice it takes `self` (not `&self` or `&mut self`). This means calling this method takes ownership of the List.
> - Once you call `my_list.into_iter()`, my_list is moved inside the `IntoIter` wrapper and can no longer be used directly in the original scope.
>
> ```rust,ignore
> impl<T> Iterator for IntoIter<T> {
>    type Item = T;
>    
>    fn next(&mut self) -> Option<Self::Item> {
>        self.0.pop()
>    }
> }
> ```
> 
> - **Associated Type**: `type Item = T`; declares that this specific iterator will yield owned `T` values (not references).
> - **The Logic (self.0.pop())**: Because `IntoIter` is a tuple struct, `self.0` accesses its first (and only) field: the inner `List<T>`.
> - The `next` method requires an `Option<T>` to be returned. Look at your `List::pop` method—it takes `&mut self`, removes the first node, and returns exactly `Option<T>`.


And let's write a test:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

Nice!
