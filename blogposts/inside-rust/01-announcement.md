# Announcing the first FFI-unwind project design meeting

-- Kyle Strand and Niko Matsakis on behalf of the FFI-unwind project group --

The FFI-unwind project group, announced in [this RFC][rfc-announcement], is
working to extend the language to support unwinding that crosses FFI
boundaries.

We have reached our first technical decision point, on a question we have been
discussing internally for quite a while. This blog post lays out the arguments
on each side of the issue and invites the Rust community to join us at an
upcoming meeting to help finalize our decision, which will be formalized and
published as our first language-change RFC. This RFC will propose an "MVP"
specification for well-defined cross-language unwinding.

The meeting will be on [Monday the 24th at 17:00 UTC, via Zoom](meeting-link).

## Background: what is unwinding?

Exceptions are a familiar control flow mechanism in many programming languages.
They are particularly commonplace in managed languages such as Java, but they
are also part of the C++ language, which introduced them to the world of
unmanaged systems programming.

When an exception is thrown, the runtime _unwinds_ the stack, essentially
traversing it backwards and calling clean-up or error-recovery code such as
destructors or `catch` blocks.

Compilers may implement their own unwinding mechanisms, but in native code such
as Rust binaries, the mechanism is more commonly provided by the platform ABI.

It is well known that Rust does not have exceptions as such. But Rust _does_
support unwinding! There are two scenarios that will cause unwinding to occur:

* By default, Rust's `panic!()` unwinds the stack.
* Using FFI, Rust can call functions in other languages (such as C++) that can
  unwind the stack.
  * There are some special cases where C libraries can actually cause
    unwinding.  For instance, on Microsoft platforms, `longjmp` is implemented
    as "forced unwinding" (see below)

Currently, when foreign (non-Rust) code invokes Rust code, the behavior of a
`panic!()` unwind that "escapes" from Rust is explicitly undefined. Similarly,
when Rust calls a foreign function that unwinds, the behavior once the unwind
operation encounters Rust frames is undefined. The primary reason for this is
to ensure that Rust implementations may use their own unwinding mechanism,
which may not be compatible with the platform-provided "native" unwinding
mechanism. Currently, however, `rustc` uses the native mechanism, and there are
no plans to change this.

### Forced unwinding

Platform ABIs can define a special kind of unwinding called "forced unwinding."
This type of unwinding operation cannot be stopped prematurely using language
mechanisms such as Rust's `catch_unwind` or C++'s `catch`. (These mechanisms
can intercept forced unwinds, but the runtime will resume unwinding regardless
of the content of the intercepting code, unless the program is aborted).

There are two common types of forced unwinding:

* On Windows platforms, `longjmp` is implemented as a forced unwind.
* On UNIX-like platforms, `pthread_exit` and `pthread_cancel` result in a forced unwind.

## Requirements for any cross-language unwinding specification

* Unwinding between Rust functions (and in particular unwinding of Rust panics)
  may not necessarily use the system unwinding mechanism
  * In practice, we do use the system mechanism today, but we would like to
    reserve the freedom to change this
* If you enable `-Cpanic=abort`, we are able to optimize the size of binaries
  to remove most code related to unwinding.
  * In extreme cases, it should be possible to remove **all** traces of
    unwinding, when you know that it will not occur.
  * In practice, most "C" functions are never expected to unwind (because they
    are written in C, for example, and not in C++).
    * However, because unwinding is now part of most system ABIs, even C
      functions can unwind -- most notably, `pthread_cancel` can cause all
      manner of C functions to unwind, including common functions like `read`
      and `write`.  (For a complete list, search for "cancellation points" in
      the [pthreads man
      page](http://man7.org/linux/man-pages/man7/pthreads.7.html).)
* Changing the behavior from `-Cpanic=unwind` to `-Cpanic=abort` should not
  cause Undefined Behavior.
  * However, this may not be tenable, or at least not without making binaries
    much larger. See the discussion below for more details.
  * It may, of course, cause programs to abort that used to execute
    successfully. This could occur if a panic would've been caught and
    recovered.

## The primary question: should the `"C"` ABI permit unwinding?

The core question that we would like to decide is whether the `"C"` ABI, as
defined by Rust, should permit unwinding.

This is not a question we expected to be debating. We've long declared that
unwinding through Rust's `"C"` ABI is undefined behavior. In part, this is
because nobody had spent the time to figure out what the correct behavior would
be, or how to implement it, although (as we'll see shortly) there are other
good reasons for this choice. 

In any case, in PR #65646, @Amanieu proposed that we could, in fact, simply
define the behavior of unwinding across `"C"` boundaries. In discussing this,
discovered that the question of whether the `"C"` ABI should permit unwinding was
less clear-cut than we had assumed.

If the `"C"` ABI does not permit unwinding, a new ABI, called `"C unwind"`,
will be introduced specifically to support unwinding.

## Three specific proposals

The project group has narrowed the design space down to three specific
proposals. Two of these introduce the new `"C unwind"` ABI, and one does not.

Each proposal specifies the behavior of each type of unwind (Rust `panic!`,
foreign (e.g. C++), and forced (e.g. `pthread_exit`)) when it encounters an
ABI boundary under either the `panic=unwind` or `panic=abort` compile-mode.

Throughout, the  unwind generated by `panic!` will be referred to as
`panic`-unwind.

### Proposal 1: Introduce `"C unwind"`, minimal specification

* `"C"` ABI boundary, `panic=<any>`
  * `panic`-unwind: program aborts
  * foreign unwind, no destructors: unwind behaves normally
  * foreign unwind, with destructors: undefined behavior
* `"C unwind"` ABI boundary
  * With `panic=unwind`: all types of unwinding behave normally
  * With `panic=abort`: all types of unwinding abort the program

### Proposal 2: Introduce `"C unwind"`, forced unwinding always permitted

This is the same as the previous design, except that when compiled with
`panic=abort`, forced unwinding would *not* be intercepted at `"C unwind"` ABI
boundaries; that is, they would behave normally, without causing the program to
abort. `panic`-unwind and non-forced foreign exceptions would still cause the
program to abort.

### Proposal 3:  No new ABI

* `panic=unwind`: unwind behaves normally
* `panic=abort`:
  * `panic`-unwind: does not exist; `panic!` aborts the program
  * foreign unwinding, no destructors: unwind behaves normally
  * foreign unwinding with destructors: undefined behavior

## Comparison table for the proposed designs

In this table, "UB" stands for "undefined behavior". Some instances of UB can
be detected at runtime, but the code to do so would impose an undesirable
code-size penalty; the group recommends generating such code only for debug
builds. These cases are marked "UB (debug: abort)"

Note that forced unwinding always causes undefined behavior when it traverses
stack frames with destructors.

 |                                                        | `panic`-unwind                        | Forced unwind, no destructors | Foreign unwind, non-forced, no destructors | Foreign unwind (forced or unforced) with destructors |
 | ------------------------------------------------------ | ------------------------------------- | ----------------------------- | ------------------------------------------ | ---------------------------------------------------- |
 | Proposals 1 & 2, `"C"` boundary, `panic=unwind`        | abort                                 | unwind                        | unwind                                     | UB                                                   |
 | Proposals 1 & 2, `"C"` boundary, `panic=abort`         | `panic!` aborts (no unwinding occurs) | unwind                        | unwind                                     | UB                                                   |
 | Proposals 1 & 2, `"C unwind"` boundary, `panic=unwind` | unwind                                | unwind                        | unwind                                     | unwind                                               |
 | Proposal 1, `"C unwind"` boundary, `panic=abort`       | `panic!` aborts                       | abort                         | abort                                      | abort                                                |
 | Proposal 2, `"C unwind"` boundary, `panic=abort`       | `panic!` aborts                       | unwind                        | abort                                      | UB if forced; abort otherwise                        |
 | Proposal 3,  `"C"` boundary, `panic=unwind`            | unwind                                | unwind                        | unwind                                     | unwind                                               |
 | Proposal 3, `"C"` boundary, `panic=abort`              | `panic!` aborts                       | unwind                        | unwind                                     | UB (debug: abort)                                    |

[rfc-announcement]: https://github.com/rust-lang/rfcs/pull/2797
[meeting-link]: https://arewemeetingyet.com/UTC/2020-02-24/17:00/Lang%20Team%20Design%20Meeting:%20FFI-unwind#eyJ1cmwiOiJodHRwczovL21vemlsbGEuem9vbS51cy9qLzc2ODIzMTc2MCJ9
