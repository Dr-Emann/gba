# GBA PRNG

You often hear of the "Random Number Generator" in video games. First of all,
usually a game doesn't have access to any source of "true randomness". On a PC
you can send out a web request to [random.org](https://www.random.org/) which
uses atmospheric data, or even just [point a camera at some lava
lamps](https://blog.cloudflare.com/randomness-101-lavarand-in-production/). Even
then, the rate at which you'll want random numbers far exceeds the rate at which
those services can offer them up. So instead you'll get a pseudo-random number
generator and "seed" it with the true random data and then use that.

However, we don't even have that! On the GBA, we can't ask any external anything
what we should do for our initial seed. So we will not only need to come up with
a few PRNG options, but we'll also need to come up with some seed source
options. More than with other options within the book, I think this is an area
where you can tailor what you do to your specific game.

## What is a Pseudo-random Number Generator?

For those of you who somehow read The Rust Book, plus possibly The Rustonomicon,
and then found this book, but somehow _still_ don't know what a PRNG is... Well,
I don't think there are many such people. Still, we'll define it anyway I
suppose.

> A PRNG is any mathematical process that takes an initial input (of some fixed
> size) and then produces a series of outputs (of a possibly different size).

So, if you seed your PRNG with a 32-bit value you might get 32-bit values out or
you might get 16-bit values out, or something like that.

We measure the quality of a PRNG based upon:

1) **Is the output range easy to work with?** Most PRNG techniques that you'll
   find these days are already hip to the idea that we'll have the fastest
   operations with numbers that match our register width and all that, so
   they're usually designed around power of two inputs and power of two outputs.
   Still, every once in a while you might find some page old page intended for
   compatibility with the `rand()` function in the C standard library that'll
   talk about something _crazy_ like having 15-bit PRNG outputs. Stupid as it
   sounds, that's real. Avoid those. We almost always want generators that give
   us uniformly distributed `u8`, `u16`, `u32`, or whatever size value we're
   producing. From there we can mold our random bits into whatever else we need
   (eg: turning a `u8` into a "1d6" roll).
2) **How long does each generation cycle take?** This can be tricky for us. A
   lot of the top quality PRNGs you'll find these days are oriented towards
   64-bit machines so they do a bunch of 64-bit operations. You _can_ do that on
   a 32-bit machine if you have to, and the compiler will automatically "lower"
   the 64-bit operation into a series of 32-bit operations. What we'd really
   like to pick is something that sticks to just 32-bit operations though, since
   those will be our best candidates for fast results. As with other
   benchmarking related things, we can use [Compiler
   Explorer](https://rust.godbolt.org/z/JyX7z-) set for the `thumbv6m-none-eabi`
   target as a basic approximation, which we'll do in this section. Of course,
   not every instruction is the same time to execute, but basically less ASM is
   better for us. If you wanted to be even more precise you could also try to
   coax rustc to spit out the ASM directly (though `xbuild` makes that a hair
   tricky) and then pick through that and use the [execution
   times](http://problemkaputt.de/gbatek.htm#armcpuoverview) listed in GBATEK to
   figure out a total cycle cost, or you could even try to make some sort of
   benchmarking harness for the GBA itself if you were really dedicated.
3) **What is the statistical quality of the output?** This involves heavy
   amounts of math. Since computers are quite good a large amounts of repeated
   math you might wonder if there's programs for this already, and there are.
   Many in fact. They take a generator and then run it over and over and perform
   the necessary tests and report the results. I won't be explaining how to hook
   our generators up to those tools, they each have their own user manuals.
   However, if someone says that a generator "passes BigCrush" (the biggest
   suite in TestU01) or "fails PractRand" or anything similar it's useful to
   know what they're referring to. Example test suites include:
   * [TestU01](https://en.wikipedia.org/wiki/TestU01)
   * [PractRand](http://pracrand.sourceforge.net/)
   * [Dieharder](https://webhome.phy.duke.edu/~rgb/General/dieharder.php)
   * [NIST Statistical Test
     Suite](https://csrc.nist.gov/projects/random-bit-generation/documentation-and-software)

Note that generators with a small state size will _always_ fail the statistical
test suites simply because the suites ask them to produce too much output
relative to their state size. The same _would_ also happen to larger generators
too if you ran them long enough, it's just that the amount of required output to
make the generators fail can quickly range up into "100s of years" and beyond as
your generator gets bigger. With a modern "actual" computer (desktop, server,
cloud VM, etc) a good PRNG can produce an output in about 1 nanosecond (depends
on your exact CPU of course). If we wanted to see how long it'd take to run
through a PRNG's whole state, well [2^32
nanoseconds](https://www.wolframalpha.com/input/?i=2%5E32+nanoseconds+in+years)
is 4.295 seconds, but [2^64
nanoseconds](https://www.wolframalpha.com/input/?i=2%5E64+nanoseconds+in+years)
is 584.9 _years_. Of course, the GBA can't actually run a PRNG that fast (with
our poor little 16.78MHz), but the difference in scale is still there. A small
amount of extra state can make a big difference in generator quality if your
algorithm is putting it to good use.

### Generator Size

Of course, generator quality has to be held in comparison to generator size and
features. We don't always need the highest possible quality generators. "But
Lokathor!", I can already hear you shouting. "I want the highest quality
randomness at all times! The game depends on it!", you cry out. Well... does it?
Like, really? The [GBA
Pokemon](https://bulbapedia.bulbagarden.net/wiki/Pseudorandom_number_generation_in_Pok%C3%A9mon)
games use a _dead simple_ PRNG technique called LCG, which fails statistical
tests when it's only 32 bits big like the GBA games had. Then starting with the
DS they moved to also using Mersenne Twister, which fails several statistical
tests and is one of the most predictable PRNGs around. [Metroid
Fusion](http://wiki.metroidconstruction.com/doku.php?id=fusion:technical:rng)
has a 100% goofy PRNG system for enemies that would definitely never pass any
sort of statistics tests at all. But like, those games were still awesome. Since
we're never going to be keeping secrets safe with our generator, it's okay if we
trade in some quality for something else in return (we obviously don't want to
trade quality for nothing).

So let's talk about size: Where's the space used for the Metroid Fusion PRNG? No
where at all! They were already using everything involved for other things too,
so they're paying no extra cost to have the randomization they do. How much does
it cost Pokemon to throw in a 32-bit LCG? Just 4 bytes, might as well. How much
does it cost to add in a Mersenne Twister? ~2,500 bytes ya say? I'm sorry _what
on Earth_? Yeah, that's crazy, we're probably not doing that.

### k-Dimensional Equidistribution

So, wait, why did the Pokemon developers add in the Mersenne Twister generator?
They're smart people, surely they had a reason. Obviously we can't know for
sure, but Mersenne Twister is terrible in a lot of ways, so what's its single
best feature? Well, that gets us to a funky thing called **k-dimensional
equidistribution**. Basically, if you take a generator's output and chop it down
to get some value you want, with uniform generator output you can always get a
smaller ranged uniform result (though sometimes you will have to reject a result
and run the generator again). Imagine you have a `u32` output from your
generator. If you want a `u16` value from that you can just pick either half. If
you want a `[bool; 4]` from that you can just pick four bits. However you wanna
do it, as long as the final form of random thing we're getting needs a number of
bits _equal to or less than_ the number of bits that come out of a single
generator use, we're totally fine.

What happens if the thing you want to make requires _more_ bits than a single
generator's output? You obviously have to run the generator more than once and
then stick two or more outputs together, duh. Except, that doesn't always work.
What I mean is that obviously you can always put two `u8` side by side to get a
`u16`, but if you start with a uniform `u8` generator and then you run it twice
and stick the results together you _don't_ always get a uniform `u16` generator.
Imagine a byte generator that just does `state+=1` and then outputs the state.
It's not good by almost any standard, but it _does give uniform output_. Then we
run it twice in a row, put the two bytes together, and suddenly a whole ton of
potential `u16` values can never be generated. That's what k-dimensional
equidistribution is all about. Every uniform output generator is 1-dimensional
equidistributed, but if you need to combine outputs and still have uniform
results then you need a higher `k` value. So why does Pokemon have Mersenne
Twister in it? Because it's got 623-dimensional equidistribution. That means
when you're combining PRNG calls for all those little IVs and Pokemon Abilities
and other things you're sure to have every potential pokemon actually be a
pokemon that the game can generate. Do you need that for most situations?
Absolutely not. Do you need it for pokemon? No, not even then, but a lot of the
hot new PRNGs have come out just within the past 10 years, so we can't fault
them too much for it.

### Other Tricks

Finally, some generators have other features that aren't strictly quantifiable.
Two tricks of note are "jump ahead" or "multiple streams":

* Jump ahead lets you advance the generator's state by some enormous number of
  outputs in a relatively small number of operations.
* Multi-stream generators have more than one output sequence, and then some part
  of their total state space picks a "stream" rather than being part of the
  actual seed, with each possible stream causing the potential output sequence
  to be in a different order.

They're normally used as a way to do multi-threaded stuff (we don't care about
that on GBA), but another interesting potential is to take one world seed and
then split off a generator for each "type" of thing you'd use PRNG for (combat,
world events, etc). This can become quite useful, where you can do things like
procedurally generate a world region, and then when they leave the region you
only need to store a single generator seed and a small amount of "delta"
information for what the player changed there that you want to save, and then
when they come back you can regenerate the region without having stored much at
all. This is the basis for how old games with limited memory like
[Starflight](https://en.wikipedia.org/wiki/Starflight) did their whole thing
(800 planets to explore on just to 5.25" floppy disks!).

## How To Seed

TODO

## Various Generators

### SM64 (16-bit state, 16-bit output, non-uniform, bonkers)

Our first PRNG to mention isn't one that's at all good, but it sure might be
cute to use. It's the PRNG that Super Mario 64 had ([video explanation,
long](https://www.youtube.com/watch?v=MiuLeTE2MeQ)).

```rust
pub fn sm64(mut input: u16) -> u16 {
  if input == 0x560A {
    input = 0;
  }
  let mut s0 = input << 8;
  s0 ^= input;
  input = s0.rotate_left(8);
  s0 = ((s0 as u8) << 1) as u16 ^ input;
  let s1 = (s0 >> 1) ^ 0xFF80;
  if (s0 & 1) == 0 {
    if s1 == 0xAA55 {
        input = 0;
    } else {
        input = s1 ^ 0x1FF4;
    }
  } else {
    input = s1 ^ 0x8180;
  }
  input
}
```

[Compiler Explorer](https://rust.godbolt.org/z/1F6P8L)

If you watch the video you'll note that the first `if` checking for `0x560A` is
only potentially important to avoid being locked in a 2-step cycle, though if
you can guarantee that you'll never pass a bad input value I suppose you could
eliminate it. The second `if` that checks for `0xAA55` doesn't seem to be
important at all from a mathematical perspective. It's left in there only for
authenticity.

### LCG32 (32-bit state, 32-bit output, uniform)

The [Linear Congruential
Generator](https://en.wikipedia.org/wiki/Linear_congruential_generator) is a
well known PRNG family. You pick a multiplier and an additive and you're done.
Right? Well, not exactly, because (as the wikipedia article explains) the values
that you pick can easily make your LCG better or worse all on its own. You want
a good multiplier, and you want your additive to be odd. In our example here
we've got the values that
[Bulbapedia](https://bulbapedia.bulbagarden.net/wiki/Pseudorandom_number_generation_in_Pok%C3%A9mon)
says were used in the actual GBA Pokemon games, though Bulbapedia also lists
values for a few other other games as well.

I don't actually know if _any_ of the constants used in the official games are
particularly good from a statistical viewpoint, though with only 32 bits an LCG
isn't gonna be passing any of the major statistical tests anyway (you need way
more bits in your LCG for that to happen). In my mind the main reason to use a
plain LCG like this is just for the fun of using the same PRNG that an official
Pokemon game did.

You should _not_ use this as your default generator if you care about quality.

It is _very_ fast though... if you want to set everything else on fire for
speed. If you do, please _at least_ remember that the highest bits are the best
ones, so if you're after less than 32 bits you should shift the high ones down
and keep those. If you want to turn it into a `bool` cast to `i32` and then
check if it's negative.

```rust
pub fn pkmn_lcg(seed: u32) -> u32 {
  seed.wrapping_mul(0x41C6_4E6D).wrapping_add(0x0000_6073)
}
```

[Compiler Explorer](https://rust.godbolt.org/z/k5n_jJ)

What's this `wrapping_mul` stuff? Well, in Rust's debug builds a numeric
overflow will panic, and then overflows are unchecked in `--release` mode. If
you want things to always wrap without problems you can either use a compiler
flag to change how debug mode works, or (for more "portable" code) you can just
make the call to `wrapping_mul`. All the same goes for add and subtract and so
on.

### PCG16 XSH-RR (32-bit state, 16-bit output, uniform)

The [Permuted Congruential
Generator](https://en.wikipedia.org/wiki/Permuted_congruential_generator) family
is the next step in LCG technology. We start with LCG output, which is good but
not great, and then we apply one of several possible permutations to bump up the
quality. There's basically a bunch of permutation components that are each
defined in terms of the bit width that you're working with. The "default"
variant of PCG, PCG32, has 64 bits of state and 32 bits of output, and it uses
the "XSH-RR" permutation.

Obviously we'll have 32 bits of state, and so 16 bits of output.

* **XSH:** we do an xor shift, `x ^= x >> constant`, with the constant being half
  the bits _not_ discarded by the next operation (the RR).
* **RR:** we do a randomized rotation, with output half the size of the input.
  This part gets a little tricky so we'll break it down into more bullet points.
  * Given a 2^b-bit input word, we have 32-bit input, `b = 5`
  * the top b−1 bits are used for the rotate amount, `rotate 4`
  * the next-most-significant 2^b−1 bits are rotated right and used as the
    output, `rotate the 16 bits after the top 4 bits`
  * and the low 2^b−1+1−b bits are discarded, `discard the rest`
  * This also means that the "bits not discarded" is 16+4, so the XSH constant
    will be 20/2=10.

Of course, since PCG is based on a LCG, we have to start with a good LCG base.
As I said above, a better or worse set of LCG constants can make your generator
better or worse. I'm not an expert, so I [asked an
expert](http://www.ams.org/journals/mcom/1999-68-225/S0025-5718-99-00996-5/S0025-5718-99-00996-5.pdf).
I'm definitely not the best at reading math papers, but it seems that the
general idea is that we want `m % 8 == 5` and `is_even(a)` to both hold for the
values we pick. There are three suggested LCG multipliers. In a chart. A chart
that's hard to understand. Truth be told I asked some folks that are good at
math papers and even they couldn't make sense of the chart. They concluded the
same as I did that we probably want to pick the `32310901` option. For an
additive value, we can pick any odd value, so we might as well pick something
small so that we can do an immediate add.

_Immediate_ add? That sounds new. An immediate instruction is where the op code
bits of an instruction (add, mul, etc) don't take up much space within the full
instruction, so the rest of the bits can encode one side of the operation
instead of having to specify two separate registers. It usually means one less
load you have to do, if you're working with small enough numbers. To see what I
mean compare [loading the add value](https://rust.godbolt.org/z/LKCFUS) to
[immediate add value](https://rust.godbolt.org/z/SnZW9a). It's something you
might have seen frequently in `x86` or `x86_64` ASM output, but because a thumb
instruction is only 16 bits total, we can only get immediate instructions if the
target value is 8 bits or less, so we haven't used them too much ourselves yet.

I guess we'll pick 5, because I happen to personally like the number.

```rust
pub fn pcg16_xsh_rr(seed: &mut u32) -> u16 {
  *seed = seed.wrapping_mul(32310901).wrapping_add(5);
  const INPUT_SIZE: u32 = 32;
  const OUTPUT_SIZE: u32 = 16;
  const ROTATE_BITS: u32 = 4;
  let mut out32 = *seed;
  let rot = out32 >> (INPUT_SIZE - ROTATE_BITS);
  out32 ^= out32 >> ((OUTPUT_SIZE + ROTATE_BITS) / 2);
  ((out32 >> (OUTPUT_SIZE - ROTATE_BITS)) as u16).rotate_right(rot)
}
```

[Compiler Explorer](https://rust.godbolt.org/z/rGTj7D)

### PCG16 XSH-RS (32-bit state, 16-bit output, uniform)

Instead of doing a random rotate, we can also do a random shift.

* **RS:** A random (input-dependent) shift, for cases where rotates are more
  expensive. Again, the output is half the size of the input.
  * Beginning with a 2^b-bit input word, `b = 5`
  * the top b−3 bits are used for a shift amount, `shift = 2`
  * which is applied to the next-most-significant 2^b−1+2^b−3−1 bits, `the next
    19 bits`
  * and the least significant 2b−1 bits of the result are output. `output = 16`
  * The low 2b−1−2b−3−b+4 bits are discarded. `discard the rest`
  * the "bits not discarded" for the XSH step 16+2, so the XSH constant will be
    18/2=9.

```rust
pub fn pcg16_xsh_rs(seed: &mut u32) -> u16 {
  *seed = seed.wrapping_mul(32310901).wrapping_add(5);
  const INPUT_SIZE: u32 = 32;
  const OUTPUT_SIZE: u32 = 16;
  const SHIFT_BITS: u32 = 2;
  const NEXT_MOST_BITS: u32 = 19;
  let mut out32 = *seed;
  let shift = out32 >> (INPUT_SIZE - SHIFT_BITS);
  out32 ^= out32 >> ((OUTPUT_SIZE + SHIFT_BITS) / 2);
  (out32 >> (NEXT_MOST_BITS + shift)) as u16
}
```

[Compiler Explorer](https://rust.godbolt.org/z/EvzCAG)

Turns out this a fairly significant savings on instructions. We're theoretically
trading in a bit of statistical quality for these speed gains, but a 32-bit
generator was never going to pass muster anyway, so we might as well go with
this for our 32->16 generator.

### PCG32 RXS-M-XS (32-bit state, 32-bit output, uniform)

Having the output be smaller than the input is great because you can keep just
the best quality bits that the LCG stage puts out, and you basically get 1 point
of dimensional equidistribution for each bit you discard as the size goes down
(so 32->16 gives 16). However, if your output size _has_ to the the same as your
input size, the PCG family is still up to the task.

* **RXS:** An xorshift by a random (input-dependent) amount.
* **M:** A multiply by a fixed constant.
* **XS:** An xorshift by a fixed amount. This improves the bits in the lowest
  third of bits using the upper third.

For this part, wikipedia doesn't explain as much of the backing math, and
honestly even [the paper
itself](http://www.pcg-random.org/pdf/hmc-cs-2014-0905.pdf) also doesn't quite
do a good job of it. However, rejoice, the wikipedia article lists what we
should do for 32->32, so we can just cargo cult it.

```rust
pub fn pcg32_rxs_m_xs(seed: &mut u32) -> u32 {
  *seed = seed.wrapping_mul(32310901).wrapping_add(5);
  let mut out32 = *seed;
  let rxs = out32 >> 28;
  out32 ^= out32 >> (4 + rxs);
  const PURE_MAGIC: u32 = 277803737;
  out32 *= PURE_MAGIC;
  x ^ (x >> 22)
}
```

### Xoshiro128** (128-bit state, 32-bit output, non-uniform)

It was suggested that I not show complete favoritism to just the PCG, and so we
will also look at the
[Xoshiro128**](http://xoshiro.di.unimi.it/xoshiro128starstar.c) generator. Take
care not to confuse it with the
[Xoroshiro128**](http://xoshiro.di.unimi.it/xoroshiro128starstar.c) generator
which is the 64 bit variant. Note the extra "ro" hiding in the 64-bit version's
name.

Anyway, weird names aside, you can look at the C version that I linked to, or
this Rust translation below. It's zippy and all, though 0 will be produced one
less time than all other outputs, making it non-uniform by just a little bit. It
also has a fixed jump function.

**Important:** With this generator you _must_ initialize the seed array to not
be all 0s before you start using the generator.

```rust
pub fn xoshiro128_starstar(seed: &mut [u32; 4]) -> u32 {
  let output = seed[0].wrapping_mul(5).rotate_left(7).wrapping_mul(9);
  let t = seed[1] << 9;

  seed[2] ^= seed[0];
  seed[3] ^= seed[1];
  seed[1] ^= seed[2];
  seed[0] ^= seed[3];

  seed[2] ^= t;

  seed[3] = seed[3].rotate_left(11);

  output
}

pub fn xoshiro128_starstar_jump(seed: &mut [u32; 4]) {
  const JUMP: [u32; 4] = [0x8764000b, 0xf542d2d3, 0x6fa035c3, 0x77f2db5b];
  let mut s0 = 0;
  let mut s1 = 0;
  let mut s2 = 0;
  let mut s3 = 0;
  for j in JUMP.iter() {
    for b in 0 .. 32 {
        if *j & (1 << b) > 0 {
            s0 ^= seed[0];
            s1 ^= seed[1];
            s2 ^= seed[2];
            s3 ^= seed[3];
        }
        xoshiro128_starstar(seed);
    }
  }
  seed[0] = s0;
  seed[1] = s1;
  seed[2] = s2;
  seed[3] = s3;
}
```

[Compiler Explorer](https://rust.godbolt.org/z/PGvwZw)

### More Generators?

For completeness I'll even list some generators that I looked at as potential
options and then _didn't_ include, along with why I chose to skip them.

* [Xorshift family](https://en.wikipedia.org/wiki/Xorshift): the base form gives
  N->N with a period of 2^N-1 (aka, non-uniform output). We already have the
  LCG32 example for fast 32->32 with uniform output. There's other Xorshift
  variants but none of them stood out to me since we also have `Xoshiro128**`,
  which is basically the even more refined version of this general group.
* [Mersenne Twister](https://en.wikipedia.org/wiki/Mersenne_Twister): Gosh, 2.5k
  is just way too many for me to ever want to use this thing. If you'd really
  like to use it, there is a
  [crate](https://docs.rs/mersenne_twister/1.1.1/mersenne_twister/) for it that
  already has it. Small catch, they use a ton of stuff from `std` that they
  could be importing from `core`, so you'll have to fork it and patch it
  yourself to get it working on the GBA. They also stupidly depend on an old
  version of `rand`, so you'll have to cut out that nonsense.

TODO

## Placing a Value In Range

I said earlier that you can always take a uniform output and then throw out some
bits, and possibly the whole result, to reduce it down into a smaller range. How
exactly does one do that? Well it turns out that it's [very
tricky](http://www.pcg-random.org/posts/bounded-rands.html) to get right, and we
could be losing as much as 60% of our execution time if we don't do it carefully.

The _best_ possible case is if you can cleanly take a specific number of bits
out of your result without even doing any branching. The rest can be discarded
or kept for another step as you choose. I know that I keep referencing Pokemon,
but it's a very good example for the use of randomization. Each pokemon has,
among many values, a thing called an "IV" for each of 6 stats. The IVs range
from 0 to 31, which is total nonsense to anyone not familiar with decimal/binary
conversions, but to us programmers that's clearly a 5 bit range. Rather than
making math that's better for people using decimal (such as a 1-20 range or
something like that) they went with what's easiest for the computer.

The _next_ best case is if you can have a designated range that you want to
generate within that's known at compile time. This at least gives us a chance to
write some bit of extremely specialized code that can take random bits and get
them into range. Hopefully your range can be "close enough" to a binary range
that you can get things into place. Example: if you want a "1d6" result then you
can generate a `u16`, look at just 3 bits (`0..8`), and if they're in the range
you're after you're good. If not you can discard those and look at the next 3
bits. We started with 16 of them, so you get five chances before you have to run
the generator again entirely.

The goal here is to avoid having to do one of the worst things possible in
computing: _divmod_. It's terribly expensive, even on a modern computer it's
about 10x as expensive as any other arithmetic, and on a GBA it's even worse for
us. We have to call into the BIOS to have it do a software division. Calling
into the BIOS at all is about a 60 cycle overhead (for comparison, a normal
function call is more like 30 cycles of overhead), _plus_ the time it takes to
do the math itself. Remember earlier how we were happy to have a savings of 5
instructions here or there? Compared to this, all our previous efforts are
basically useless if we can't evade having to do a divmod. You can do quite a
bit of `if` checking and potential additional generator calls before it exceeds
the cost of having to do even a single divmod.

### Calling The BIOS

How do we do the actual divmod when we're forced to? Easy: [inline
assembly](https://doc.rust-lang.org/unstable-book/language-features/asm.html) of
course (There's also an [ARM
oriented](http://embed.rs/articles/2016/arm-inline-assembly-rust/) blog post
about it that I found most helpful). The GBA has many [BIOS
Functions](http://problemkaputt.de/gbatek.htm#biosfunctions), each of which has
a designated number. We use the
[swi](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0068b/BABFCEEG.html)
op (short for "SoftWare Interrupt") combined with the BIOS function number that
we want performed. Our code halts, some setup happens (hence that 60 cycles of
overhead I mentioned), the BIOS does its thing, and then eventually control
returns to us.

The precise details of what the BIOS call does depends on the function number
that we call. We'd even have to potentially mark it as volatile asm if there's
no clear outputs, otherwise the compiler would "helpfully" eliminate it for us
during optimization. In our case there _are_ clear outputs. The numerator goes
into register 0, and the denominator goes into register 1, the divmod happens,
and then the division output is left in register 0 and the modulus output is
left in register 1. I keep calling it "divmod" because div and modulus are two
sides of the same coin. There's no way to do one of them faster by not doing the
other or anything like that, so we'll first define it as a unified function that
returns a tuple:

```rust
#![feature(asm)]
// put the above at the top of any program and/or library that uses inline asm

pub fn div_modulus(numerator: i32, denominator: i32) -> (i32, i32) {
  assert!(denominator != 0);
  {
    let div_out: i32;
    let mod_out: i32;
    unsafe {
      asm!(/* assembly template */ "swi 0x06"
          :/* output operands */ "={r0}"(div_out), "={r1}"(mod_out)
          :/* input operands */ "{r0}"(numerator), "{r1}"(denominator)
          :/* clobbers */ "r3"
          :/* options */
    );
    }
    (div_out, mod_out)
  }
}
```

And next, since most of the time we really do want just the `div` or `modulus`
without having to explicitly throw out the other half, we also define
intermediary functions to unpack the correct values.

```rust
pub fn div(numerator: i32, denominator: i32) -> i32 {
  div_modulus(numerator, denominator).0
}

pub fn modulus(numerator: i32, denominator: i32) -> i32 {
  div_modulus(numerator, denominator).1
}
```

We can generally trust the compiler to inline single line functions correctly
even without an `#[inline]` directive when it's not going cross-crate or when
LTO is on. I'd point you to some exact output from the Compiler Explorer, but at
the time of writing their nightly compiler is broken, and you can only use
inline asm with a nightly compiler. Unfortunate. Hopefully they'll fix it soon
and I can come back to this section with some links.

### Finally Those Random Ranges We Mentioned

Of course, now that we can do divmod if we need to, let's get back to random
numbers in ranges that aren't exact powers of two.

yada yada yada, if you just use `x % n` to place `x` into the range `0..n` then
you'll turn an unbiased value into a biased value (or you'll turn a biased value
into an arbitrarily _more_ biased value). You should never do this, etc etc.

So what's a good way to get unbiased outputs? We're going to be adapting some
CPP code from that  that I first hinted at way up above. It's specifically all
about the various ways you can go about getting unbiased random results for
various bounds. There's actually many different methods offered, and for
specific situations there's sometimes different winners for speed. The best
overall performer looks like this:

```cpp
uint32_t bounded_rand(rng_t& rng, uint32_t range) {
    uint32_t x = rng();
    uint64_t m = uint64_t(x) * uint64_t(range);
    uint32_t l = uint32_t(m);
    if (l < range) {
        uint32_t t = -range;
        if (t >= range) {
            t -= range;
            if (t >= range) 
                t %= range;
        }
        while (l < t) {
            x = rng();
            m = uint64_t(x) * uint64_t(range);
            l = uint32_t(m);
        }
    }
    return m >> 32;
}
```

And, wow, I sure don't know what a lot of that means (well, I do, but let's
pretend I don't for dramatic effect, don't tell anyone). Let's try to pick it
apart some.

First, all the `uint32_t` and `uint64_t` are C nonsense names for what we just
call `u32` and `u64`. You probably guessed that on your own.

Next, `rng_t& rng` is more properly written as `rng: &rng_t`. Though, here
there's a catch: as you can see we're calling `rng` within the function, so in
rust we'd need to declare it as `rng: &mut rng_t`, because C++ doesn't track
mutability the same as we do (barbaric, I know).

Finally, what's `rng_t` actually defined as? Well, I sure don't know, but in our
context it's taking nothing and then spitting out a `u32`. We'll also presume
that it's a different `u32` each time (not a huge leap in this context). To us
rust programmers that means we'd want something like `FnMut() -> u32`.

TODO: use `impl FnMut` to avoid the trait object nonsense

```rust
pub fn bounded_rand(rng: &mut FnMut() -> u32, range: u32) -> u32 {
  let mut x: u32 = rng();
  let mut m: u64 = x as u64 * range as u64;
  let mut l: u32 = m as u32;
  if l < range {
    let mut t: u32 = range.wrapping_neg();
    if t >= range {
      t -= range;
      if t >= range {
        t = modulus(t, range);
      }
    }
    while l < t {
      x = rng();
      m = x as u64 * range as u64;
      l = m as u32;
    }
  }
  (m >> 32) as u32
}
```

So, now we can read it. Can we compile it? No, actually. Turns out we can't.
Remember how our `modulus` function is `(i32, i32) -> i32`? Here we're doing
`(u32, u32) -> u32`. You can't just cast, modulus, and cast back. You'll get
totally wrong results most of the time because of sign-bit stuff. Since it's
fairly probable that `range` fits in a positive `i32`, its negation must
necessarily be a negative value, which triggers exactly the bad situation where
casting around gives us the wrong results.

Well, that's not the worst thing in the world either, since we also didn't
really wanna be doing those 64-bit multiplies. Let's try again with everything
scaled down one stage:

```rust
pub fn bounded_rand16(rng: &mut FnMut() -> u16, range: u16) -> u16 {
  let mut x: u16 = rng();
  let mut m: u32 = x as u32 * range as u32;
  let mut l: u16 = m as u16;
  if l < range {
    let mut t: u16 = range.wrapping_neg();
    if t >= range {
      t -= range;
      if t >= range {
        t = modulus(t as i32, range as i32) as u16;
      }
    }
    while l < t {
      x = rng();
      m = x as u32 * range as u32;
      l = m as u16;
    }
  }
  (m >> 16) as u16
}
```

Okay, so the code compiles, _and_ it plays nicely what the known limits of the
various number types involved. We know that if we cast a `u16` up into `i32`
it's assured to fit properly and also be positive, and the output is assured to
be smaller than the input so it'll fit when we cast it back down to `u16`.
What's even happening though? Well, this is a variation on [Lemire's
method](https://arxiv.org/abs/1805.10941). One of the biggest attempts at a
speedup here is that when you have

```rust
a %= b;
```

You can translate that into 

```rust
if a >= b {
  a -= b;
  if a >= b {
    a %= b;
  }
}
```

Now... if we're being real with ourselves, let's just think about this for a
moment. How often will this help us? I genuinely don't know. But I do know how
to find out: we write a program to just [enumerate all possible
cases](https://play.rust-lang.org/?version=stable&mode=release&edition=2015&gist=48b36f8c9f6a3284c0bc65366a4fab47)
and run the code. You can't always do this, but there's not many possible `u16`
values. The output is this:

```
skip_all:32767
sub_worked:10923
had_to_modulus:21846
Some skips:
32769
32770
32771
32772
32773
Some subs:
21846
21847
21848
21849
21850
Some mods:
0
1
2
3
4
```

So, about half the time, we're able to skip all our work, and about a sixth of
the time we're able to solve it with just the subtract, with the other third of
the time we have to do the mod. However, what I personally care about the most
is smaller ranges, and we can see that we'll have to do the mod if our target
range size is in `0..21846`, and just the subtract if our target range size is
in `21846..32769`, and we can only skip all work if our range size is `32769`
and above. So that's not cool.

But what _is_ cool is that we're doing the modulus only once, and the rest of
the time we've just got the cheap operations. Sounds like we can maybe try to
cache that work and reuse a range of some particular size. We can also get that
going pretty easily.

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct RandRangeU16 {
  range: u16,
  threshold: u16,
}

impl RandRangeU16 {
  pub fn new(mut range: u16) -> Self {
    let mut threshold = range.wrapping_neg();
    if threshold >= range {
      threshold -= range;
      if threshold >= range {
        threshold = modulus(threshold as i32, range as i32) as u16;
      }
    }
    RandRangeU16 { range, threshold }
  }

  pub fn roll_random(&self, rng: &mut FnMut() -> u16) -> u16 {
    let mut x: u16 = rng();
    let mut m: u32 = x as u32 * self.range as u32;
    let mut l: u16 = m as u16;
    if l < self.range {
      while l < self.threshold {
        x = rng();
        m = x as u32 * self.range as u32;
        l = m as u16;
      }
    }
    (m >> 16) as u16
  }
}
```

What if you really want to use ranges bigger than `u16`? Well, that's possible,
but we'd want a whole new technique. Preferably one that didn't do divmod at
all, to avoid any nastiness with sign bit nonsense. Thankfully there is one such
method listed in the blog post, "Bitmask with Rejection (Unbiased)"

```cpp
uint32_t bounded_rand(rng_t& rng, uint32_t range) {
    uint32_t mask = ~uint32_t(0);
    --range;
    mask >>= __builtin_clz(range|1);
    uint32_t x;
    do {
        x = rng() & mask;
    } while (x > range);
    return x;
}
```

And in Rust

```rust
pub fn bounded_rand32(rng: &mut FnMut() -> u32, mut range: u32) -> u32 {
  let mut mask: u32 = !0;
  range -= 1;
  mask >>= (range | 1).leading_zeros();
  let mut x = rng() & mask;
  while x > range {
    x = rng() & mask;
  }
  x
}
```

Wow, that's so much less code. What the heck? Less code is _supposed_ to be the
faster version, why is this rated slower? Basically, because of how the math
works out on how often you have to run the PRNG again and stuff, Lemire's method
_usually_ better with smaller ranges and the masking method _usually_ works
better with larger ranges. If your target range fits in a `u8`, probably use
Lemire's. If it's bigger than `u8`, or if you need to do it just once and can't
benefit from the cached modulus, you might want to start moving toward the
masking version at some point in there. Obviously if your target range is more
than a `u16` then you have to use the masking method. The fact that they're each
oriented towards different size generator outputs only makes things more
complicated.

Life just be that way, I guess.

## Summary

TODO