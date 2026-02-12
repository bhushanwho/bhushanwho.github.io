---
layout: design-post
title: "informal gentle nudge to rust"
date: 2026-02-09 08:21:21 +0530
tags: [notes]
---

here's a loose, informal and gentle introduction and nudge into rust.
at least the basics, syntactically.

this was buried in my obsidian notes and figured might as well be on here for reference.

random tutorial-doc snippets of rust.

---

## tuples

<pre><code class="language-rust">
let coords = (10.5, 20.3);
</code></pre>

tuple elements can be accessed with `a.0` or `a.1`. strange but oddly convenient.

---

## matches

here's how match can be used in Rust

<pre><code class="language-rust">
enum Direction {
	North,
	South,
	East,
	West
}

fn get_direction( dir: Direction ) {
	match dir {
		Direction::North => (0, 1),
		Direction::South => (0, -1),
		Direction::East => (1, 0),
		Direction::West => (-1, 0),
	}
}
</code></pre>

| matches, like everything, are expressions

---

## modules

<pre><code class="language-rust">
use std::cmp::min;
use std::cmp::max;

// also, work

use std::cmp::{min, max};

// but like... why.
</code></pre>

---

## if-let and match-arms

woah.

<pre><code class="language-rust">
struct Number {
	odd: bool,
	value: i32,
}

fn main() {
	...
}

fn print_number( num: Number ) {
	
	if let Number { odd: true, value } = num {
		// something
	} else if let Number { odd: false, value } = num {
		// whatever
	}
	
	// similarly
	
	match num {
		Number { odd: true, value } => // something
		Number { odd: false, value } => // whatever
	}
	
}
</code></pre>

| a match has to be exhaustive, at least one has to match. say `Number { value, .. }` to catch-all, or better yet, just `_ => // something`

## structs

there's the struct update syntax

<pre><code class="language-rust">
let v3 = Vec2 { x: 14.0, ..v2 };
let v4 = Vec3 { ..v3 };

</code></pre>

destructuring is a thing

<pre><code class="language-rust">
let (left, right) = something();

let v = Vec2 { x: 3.0, y: 6.0 };
let Vec2 { x, y } = v;
let Vec2 { x, .. } = v; // called throwaway, cause it throws away.
</code></pre>

functions for a struct. you can say `impl <struct Name>` to add the function to it. implements.

<pre><code class="language-rust">
struct Number {
	odd: bool,
	value: i32,
}

impl Number {
	fn someFunction(self) -> bool {
		println!("ayy pythonic 'self'");
		self.value == 2
	}
}
</code></pre>

variable bindings are immutable, can't change

<pre><code class="language-rust">
fn main() {
	let n = Number {
		odd: true,
		value: 3,
	};

	n.odd = false; // error
}
</code></pre>

can't reassign either.

say `let mut n = 4` for mutability.

---

## traits

traits are like interfaces, of sort. signature is written and you write implementations.

<pre><code class="language-rust">
trait Signed {
	fn is_signed(self) -> bool;
}

impl Signed for Number {
	// implementation
}
</code></pre>

also kind of overloading. say a custom implementation of negation

<pre><code class="language-rust">

impl std::ops::Neg for Number {
	type Output = Number;

	fn neg(self) -> Number {
		Number {
			value: -self.value,
			odd: self.odd,
		}
	}
}

// later in the program let m = -n of 
// type Number will work because of that implementation.

</code></pre>

`Self` is a reference to the `for` type of the implementation.

also passing by reference

<pre><code class="language-rust">
impl std::clone::Clone for Number {
	fn clone(&self) -> Self {
		Self { ..*self }
	}
}
</code></pre>

marker traits, like marker interfaces in java.

<pre><code class="language-rust">
impl std::marker::Copy for Number {}
</code></pre>

since some traits are common, you can directly derive them through, directives?

<pre><code class="language-rust">
#[derive(Clone, Copy)]
struct Number {
	odd: bool,
	value: i32,
}

// shorthand for impl Clone for Number and impl Copy for Number blocks
</code></pre>

well, formally "attributes" but you get the idea. INNER, attributes. implying the existence of other types. but you can go read about those.

---

## panics

`Option` is an `Enum` actually, so is `Result`.

`enum Option<T> { None, Some(T), }`

`enum Result<T, E> { Ok(T), Err(E), }`

an `.unwrap()` on nothing causes a panic. so if you wanna panic in case of a failure, do that.

<pre><code class="language-rust">
fn main() {
    let s = std::str::from_utf8(&[240, 159, 141, 137]).unwrap();
    println!("{:?}", s);
    // prints: "🍉"

    let s = std::str::from_utf8(&[195, 40]).unwrap();
    // prints: thread 'main' panicked at 'called `Result::unwrap()`
    // on an `Err` value: Utf8Error { valid_up_to: 0, error_len: Some(1) }',
    // src/libcore/result.rs:1165:5
}
</code></pre>

or .expect(), for a custom message:

<pre><code class="language-rust">
fn main() {
    let s = std::str::from_utf8(&[195, 40]).expect("valid utf-8");
    // prints: thread 'main' panicked at 'valid utf-8: Utf8Error
    // { valid_up_to: 0, error_len: Some(1) }', src/libcore/result.rs:1165:5
}
</code></pre>

or, you can match:

<pre><code class="language-rust">
fn main() {
    match std::str::from_utf8(&[240, 159, 141, 137]) {
        Ok(s) => println!("{}", s),
        Err(e) => panic!(e),
    }
    // prints 🍉
}
</code></pre>

or you can if-let:

<pre><code class="language-rust">
fn main() {
    if let Ok(s) = std::str::from_utf8(&[240, 159, 141, 137]) {
        println!("{}", s);
    }
    // prints 🍉
}
</code></pre>

or you can bubble up the error:

<pre><code class="language-rust">
fn main() -> Result<(), std::str::Utf8Error> {
    match std::str::from_utf8(&[240, 159, 141, 137]) {
        Ok(s) => println!("{}", s),
        Err(e) => return Err(e),
    }
    Ok(())
}
</code></pre>

or you can use ?:

<pre><code class="language-rust">
fn main() -> Result<(), std::str::Utf8Error> {
    let s = std::str::from_utf8(&[240, 159, 141, 137])?;
    println!("{}", s);
    Ok(())
}
</code></pre>

static lifetime.

<pre><code class="language-rust">

struct Person {
	name: &'static str,
}

fn main() {
	let p = Person { name: "monkey", };
}
</code></pre>

there's a lot more to read about lifetimes. starts with a single quote, as `'static`. go over it.

---

## closures

closure (arrow functions in react. in a way. in a really confusing, convoluted way) are functions of type `Fn`, `FnMut` or `FnOnce`.

<pre><code class="language-rust">
fn for_each_something<\T>(f: F)
	where F: Fn(&'static str)
{
	f(1);
	f(2);
	f(3);
}

for_each_something(|x| println!("{}", 2*x));
</code></pre>

a countdown closure

<pre><code class="language-rust">
fn countdown<\f>( count: usize, tick: F )
	where F: Fn(usize)
{
	for i in (1..=count).rev() {
		tick(i);
	}
}

fn main() {
	countdown(3, |i| println!("tick {}", i));
}
</code></pre>

familiar react-like functional code.

<pre><code class="language-rust">

fn main() {
	for c in "somestring"
		.chars()
		.filter(|c| c.is_lowercase())
		.flat_map(|c| c.to_uppercase())
		{
			print!("{}", c)
		}
	println!();
}
</code></pre>

it won't hurt to keep reading rust code from time to time and say "ah yeah this".

there's a lot more rules, and verbosely boring concepts mutability, borrowing, lifetimes, pointers, dereferencing and such.

this was served as a silly lightweight nudge into looking at and understand rust code at a syntactical level. and thus, this, ends here.

have a nice day.
