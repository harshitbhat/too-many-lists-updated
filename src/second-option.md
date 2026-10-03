# Using Option

Particularly observant readers may have noticed that we actually reinvented
a really bad version of Option:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link is just `Option<Box<Node>>`. Now, it's nice not to have to write
`Option<Box<Node>>` everywhere, and unlike `pop`, we're not exposing this
to the outside world, so maybe it's fine. However Option has some _really
nice_ methods that we've been manually implementing ourselves. Let's _not_
do that, and replace everything with Options. First, we'll do it naively
by just renaming everything to use Some and None:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// yay type aliases!
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

This is marginally better, but the big wins will come from Option's methods.

First, `mem::replace(&mut option, None)` is such an incredibly
common idiom that Option actually just went ahead and made it a method: `take`.

> The pattern "take the contents out and leave `None` behind" is what you want almost every time you detach something from a linked structure. In the chapter's `push` and `pop`:
>
> ```text
> // push: take the old head so the new node can point to it
> next: mem::replace(&mut self.head, None),
>
> // pop: take the head so we can inspect/consume it
> match mem::replace(&mut self.head, None) { ... }
> ```
>
> Because this came up so often, the standard library added a method to Option that does the same thing:
>
> ```rust,ignore
> impl<T> Option<T> {
>    pub fn take(&mut self) -> Option<T> {
>        mem::replace(self, None)
>    }
> }
> ```
>
> So these two lines are identical in behavior:
>
> ```rust,ignore
> let old = mem::replace(&mut self.head, None);
> let old = self.head.take();
> ```
>
> Key points:
>
> - `take()` takes `&mut self`, so it works through a mutable reference, which is why it fits in `&mut self` methods.
> - It returns the old `Option` by value, so you own it and can move out of it.
> - It leaves `None` in place, so the original location stays valid.
> - It is shorter and communicates intent better. `take()` reads as "take this out", while `mem::replace(&mut x, None)` makes the reader work out the same thing.

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

Second, `match option { None => None, Some(x) => Some(y) }` is such an
incredibly common idiom that it was called `map`. `map` takes a function to
execute on the `x` in the `Some(x)` to produce the `y` in `Some(y)`. We could
write a proper `fn` and pass it to `map`, but we'd much rather write what to
do _inline_.

The way to do this is with a _closure_. Closures are anonymous functions with
an extra super-power: they can refer to local variables _outside_ the closure!
This makes them super useful for doing all sorts of conditional logic. The
only place we do a `match` is in `pop`, so let's just rewrite that:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

> #### `map`: "if there's something, transform it"
>
> This pattern:
>
> ```rust,ignore
> match option {
>     None => None,
>     Some(x) => Some(y),   // y is computed from x
> }
> ```
>
> If it's empty, stay empty. If there's a value, do something to it and wrap the result back up.
> We do this all the time, so Option has a method for it:
>
> ```rust,ignore
> option.map(|x| /* compute y from x */)
> ```
>
> Think of `Option` as a box that's either empty or holds one thing.
> `map` says: "if the box has something, run this function on it and put the result in a new box. If it's empty, do nothing and hand back an empty box."
>
> A simple example:
>
> ```rust,ignore
>   let a = Some(3);
>   let b = a.map(|x| x + 1);   // Some(4)
>
>   let c: Option<i32> = None;
>   let d = c.map(|x| x + 1);   // None
> ```

> #### Closures: functions you write inline
>
> `|x| x + 1` is a closure. It's the same as:
>
> ```rust,ignore
> fn add_one(x: i32) -> i32 { x + 1 }
> ```
>
> but with no name, written right where you use it.
> Between the |...| are the parameters, and after that comes the body:
>
> ```rust,ignore
> |node| { ...body... }
> ```
>
> **The "super-power": capturing variables**
> A normal `fn` can only use its own parameters. A closure can also use variables from the surrounding code:
>
> ```rust,ignore
> let offset = 10;
> let add_offset = |x| x + offset;   // uses `offset` from outside
> ```
>
> That is exactly what happens in pop:
>
> ```rust,ignore
> pub fn pop(&mut self) -> Option<i32> {
>    self.head.take().map(|node| {
>        self.head = node.next;   // `self` comes from outside the closure
>        node.elem
>    })
> }
> ```
>
> Step by step:
>
> - `self.head.take()` gives you the old head as an `Option<Box<Node>>` and leaves None behind.
> - `.map(|node| ...)` :
>   - If it was `None`, `map` returns `None`. The closure never runs, which matches the old `None => None` arm.
>   - If it was `Some(node)`, the closure runs with that node:
>     - `self.head = node.next;` sets the list's head to the next node. This uses `self`, which isn't a parameter of the closure. It's captured from `pop's` scope.
>     - `node.elem` is the last expression, so it's the closure's return value. `map` wraps it as `Some(node.elem)`.

Ah, much better. Let's make sure we didn't break anything:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

Great! Let's move on to actually improving the code's _behaviour_.
