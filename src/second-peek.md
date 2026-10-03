# Peek

One thing we didn't even bother to implement last time was peeking. Let's go
ahead and do that. All we need to do is return a reference to the element in
the head of the list, if it exists. Sounds easy, let's try:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```

```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

_Sigh_. What now, Rust?

Map takes `self` by value, which would move the Option out of the thing it's in.
Previously this was fine because we had just `take`n it out, but now we actually
want to leave it where it was. The _correct_ way to handle this is with the
`as_ref` method on Option, which has the following definition:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

It demotes the `Option<T>` to an Option to a reference to its internals. We could
do this ourselves with an explicit match but _ugh no_. It does mean that we
need to do an extra dereference to cut through the extra indirection, but
thankfully the `.` operator handles that for us.

> #### `as_ref`: `Option<T>` to `Option<&T>`
>
> `as_ref` doesn't move anything. It gives you a new `Option` that holds a reference to the contents instead of the contents themselves:
>
> ```text
> Option<Box<Node<T>>>   // what self.head is
>        ↓ as_ref()
> Option<&Box<Node<T>>>   // same shape, but borrowing the node
> ```
>
> Now `map` consumes this new `Option<&...>`, which is just a cheap reference, and the original `self.head` is untouched:
>
> ```rust,ignore
> pub fn peek(&self) -> Option<&T> {
>    self.head.as_ref().map(|node| &node.elem)
> }
> ```
>
> Here node is `&Box<Node<T>>`, and `&node.elem` is a reference to the element inside. It's returned wrapped in `Some` by `map`, or `None` if the list is empty.
>
> A mental model: `as_ref` is "let me look at what's in the box without taking it out".

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

Nailed it.

We can also make a _mutable_ version of this method using `as_mut`:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
> cargo build

```

> #### `as_mut`: `Option<T>` to `Option<&mut T>`
>
> Same idea, but the reference is mutable, so you can change what's inside:
>
> ```text
> Option<Box<Node<T>>>
>        ↓ as_mut()
> Option<&mut Box<Node<T>>>
> ```
>
> This is used for `peek_mut`:
>
> ```rust,ignore
> pub fn peek_mut(&mut self) -> Option<&mut T> {
>    self.head.as_mut().map(|node| &mut node.elem)
> }
> ```
>
> Now the caller can modify the head element in place:
>
> ```rust,ignore
> if let Some(x) = list.peek_mut() {
>    *x = 42;
> }
> ```

EZ

Don't forget to test it:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

That's nice, but we didn't really test to see if we could mutate that `peek_mut` return value, did we? If a reference is mutable but nobody mutates it, have we really tested the mutability? Let's try using `map` on this `Option<&mut T>` to put a profound value in:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

The compiler is complaining that `value` is immutable, but we pretty clearly wrote `&mut value`; what gives? It turns out that writing the argument of the closure that way doesn't specify that `value` is a mutable reference. Instead, it creates a pattern that will be matched against the argument to the closure; `|&mut value|` means "the argument is a mutable reference, but just copy the value it points to into `value`, please." If we just use `|value|`, the type of `value` will be `&mut i32` and we can actually mutate the head:

> **closure parameters are patterns**
>
> In Rust, the thing between the `|...|` (or in a `let`, or a `match` arm) isn't just a variable name. It's a **pattern** that gets matched against the incoming argument, and it can destructure it.
>
> We've seen this with match:
>
> ```rust,ignore
> match opt {
>    Some(x) => ...,   // pattern: "if it's Some, bind the inside to x"
> }
> ```
>
> The same works for closure arguments. And `&` and `&mut` are also patterns, and they mean the opposite of what they mean in expressions.
>
> | Where      | `&mut foo` means                                                                 |
> | ---------- | -------------------------------------------------------------------------------- |
> | Expression | "make a mutable reference to foo"                                                |
> | Pattern    | "the incoming value is a mutable reference; strip it off and bind what's inside" |
>
> **What the test was doing**
>
> `peek_mut()` returns `Option<&mut i32>`. 
>
> So the closure receives an argument of type `&mut i32`.
>
> ```rust,ignore
> list.peek_mut().map(|&mut value| {
>    value = 42;
> });
> ```
>
> Here `|&mut value|` is a pattern. Rust reads it as:
>
> > "The argument is a `&mut i32`. Take the `i32` it points to, copy it into a new variable called `value`."
> > So `value` is not a reference at all. It's a plain `i32`, a copy, and it's not declared mut. That's why the compiler says `value` is immutable. The confusing part is that the error is about the local `value`, not about the reference you thought you had.
>
> Even if you added `mut` and made it compile, you'd only be changing your local copy. The list would be unchanged.
>
> **The fix**
> Skip the destructuring and let `value` be the argument itself:
>
> ```rust,ignore
> list.peek_mut().map(|value| {
>    *value = 42;
> });
> ```
>
> Now `value` has type `&mut i32`, a real mutable reference into the list. You write through it with `*value = 42`, which means "store 42 in the thing this reference points to." The head element actually changes.
>
> ```text
> // |&mut value|  →  value: i32      (a copy; the reference is stripped)
> // |value|       →  value: &mut i32 (the reference itself)
> ```
>
> **Rule of Thumb**
> If you want to use the reference (to mutate through it), name it plainly: `|value|`. Only write `&` or `&mut` in a closure's parameters when you specifically want to unwrap the reference and get the value inside.

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

Much better!
