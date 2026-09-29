# Drop

We can make a stack, push on to, pop off it, and we've even tested that it all
works right!

Do we need to worry about cleaning up our list? Technically, no, not at all!
Like C++, Rust uses destructors to automatically clean up resources when they're
done with. A type has a destructor if it implements a _trait_ called Drop.
Traits are Rust's fancy term for interfaces. The Drop trait has the following
interface:

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

Basically, "when you go out of scope, I'll give you a second to clean up your
affairs".

> **In simple terms:** "Out of scope" means the variable's block has ended, e.g.
> the closing `}` of a function. At that moment Rust automatically calls `drop`
> on the value. You never call it yourself. It's the "last chance to tidy up"
> hook, like a destructor in C++.

You don't actually need to implement Drop if you contain types that implement
Drop, and all you'd want to do is call _their_ destructors. In the case of
List, all it would want to do is drop its head, which in turn would _maybe_
try to drop a `Box<Node>`. All that's handled for us automatically... with one
hitch.

The automatic handling is going to be bad.

> **Why do we normally not need to write `Drop`?**
> `List` only _contains_ other things (`Link`, which contains `Box<Node>`, which contains `Node`...).
> Each of those already knows how to clean itself up, so Rust generates the drop code for `List` for us.
> "drop every field, one after another".
> That's fine for most types. Here it's the problem, because the fields _nest inside each other_ all the way down the list.

Let's consider a simple list:

```text
list -> A -> B -> C
```

When `list` gets dropped, it will try to drop A, which will try to drop B,
which will try to drop C. Some of you might rightly be getting nervous. This is
recursive code, and recursive code can blow the stack!

> **Picture it as a chain of people holding hands.**
> `list` can't let go until A has let go, A can't finish until B has, and B can't finish until C has.
>
> Every _"I'm waiting for the next one"_ is a function call sitting on the **stack**, and the stack is small (a few MB).
> With 3 nodes that's nothing. With 100,000 nodes there are 100,000 calls waiting at once, and the stack runs out.
>
> To test it I added a test in our code in `first.rs`
>
> ```rust,ignore
>    #[test]
>    fn long_list() {
>       let mut list = List::new();
>       for i in 0..100_000 {
> 	        list.push(i);
>       }
>       drop(list);
>    }
> ```
>
> **Output**:
>
> ```text
> thread 'first::test::long_list' (11788435) has overflowed its stack
> fatal runtime error: stack overflow, aborting
> error: test failed, to rerun pass `--lib`
> ```
>
> Nothing in the code is wrong; the list is just too long for one-call-per-node.

Some of you might be thinking "this is clearly tail recursive, and any decent
language would ensure that such code wouldn't blow the stack". This is, in fact,
incorrect! To see why, let's try to write what the compiler has to do, by
manually implementing Drop for our List as the compiler would:

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: you can't actually explicitly call `drop` in real Rust code;
        // we're pretending to be the compiler!
        self.head.drop(); // tail recursive - good!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // Done!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // tail recursive - good!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // uh oh, not tail recursive!
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

We _can't_ drop the contents of the Box _after_ deallocating, so there's no
way to drop in a tail-recursive manner! Instead we're going to have to manually
write an iterative drop for `List` that hoists nodes out of their boxes.

> **What "tail recursive" means, and why it fails here.**
>
> A call is _tail_ recursive when it's the very last thing the function does.
> Then the compiler can reuse the same stack space (in effect turning it into a loop), and nothing piles up. That's the case for the `List` and `Link` drops above.
>
> `Box<Node>` breaks it. Its drop has two steps, in this order:
>
> - drop the `Node` inside (`self.ptr.drop()`). This is where the _rest of the
>   list_ gets dropped.
> - free the memory (`deallocate(self.ptr)`).
>
> Step 2 has to come _after_ step 1, because you can't free memory you're still using.
>
> So step 1 is not the last thing the function does: the box still has work left and must wait for the whole rest of the list to finish.
>
> That waiting is what makes the stack grow, and no compiler can optimize it away.
>
> So if we can't stop the recursion, we make sure it's never deep, as we also saw in the test we added earlier. That's the next snippet.

```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "do this thing until this pattern doesn't match"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node goes out of scope and gets dropped here;
            // but its Node's `next` field has been set to Link::Empty
            // so no unbounded recursion occurs.
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

> **The idea: take the list apart one node at a time.**
>
> Before a node is destroyed, cut its link to the next node. Then destroying it is instant, because it isn't holding anything.
>
> In the hand-holding picture: walk down the line, tell each person to let go of the next person's hand, and send them away. Nobody waits on anyone.
>
> Walkthrough on `A -> B -> C`:
>
> | Round       | `cur_link` holds | What we do                                            | What gets dropped |
> | ----------- | ---------------- | ----------------------------------------------------- | ----------------- |
> | before loop |                  | `cur_link = A -> B -> C`, `self.head` becomes `Empty` | nothing           |
> | 1           | `A -> B -> C`    | pull A out, set `A.next = Empty`, `cur_link = B -> C` | A alone           |
> | 2           | `B -> C`         | pull B out, set `B.next = Empty`, `cur_link = C`      | B alone           |
> | 3           | `C`              | pull C out, `cur_link = Empty`                        | C alone           |
> | 4           | `Empty`          | pattern `Link::More(..)` doesn't match, loop ends     | nothing           |
>
> Each drop touches exactly one node, so stack depth stays constant however long
> the list is.
>
> **The Rust pieces:**
>
> - `mem::replace(&mut x, new)` puts `new` into `x` and hands you the old value. Rust won't let you move a field out of something you only borrow (`&mut`) because it would leave a hole. `replace` always leaves something valid behind. You already used this in `push` and `pop`.
> - `while let PATTERN = value` means "keep looping while `value` matches `PATTERN`", here "while there's another node".
> - We never call `drop` ourselves. Rust still drops `boxed_node` automatically at the end of each round. We only changed what it sees: a node with nothing attached.
>   The following test that we added, passes after implementing the Drop Trait
>
> ```rust,ignore
>    #[test]
>    fn long_list() {
>       let mut list = List::new();
>       for i in 0..100_000 {
> 	        list.push(i);
>       }
>       drop(list);
>    }
> ```
>
> Output:
>
> ```text
> running 2 tests
> test first::test::basics ... ok
> test first::test::long_list ... ok
>
> test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
> ```

Great!

---

<span style="float:left">![Bonus](img/profbee.gif)</span>

## Bonus Section For Premature Optimization!

Our implementation of drop is actually _very_ similar to
`while let Some(_) = self.pop() { }`, which is certainly simpler. How is
it different, and what performance issues could result from it once we start
generalizing our list to store things other than integers?

<details>
  <summary>Click to expand for answer</summary>

Pop returns `Option<i32>`, while our implementation only manipulates Links (`Box<Node>`). So our implementation only moves around pointers to nodes, while the pop-based one will move around the values we stored in nodes. This could be very expensive if we generalize our list and someone uses it to store instances of VeryBigThingWithADropImpl (VBTWADI). Box is able to run the drop implementation of its contents in-place, so it doesn't suffer from this issue. Since VBTWADI is _exactly_ the kind of thing that actually makes using a linked-list desirable over an array, behaving poorly on this case would be a bit of a disappointment.

If you wish to have the best of both implementations, you could add a new method,
`fn pop_node(&mut self) -> Link`, from-which `pop` and `drop` can both be cleanly derived.

</details>
