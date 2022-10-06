---
slug: 2022-07-20-js-backend-prim-types
title: Primitive Type Representation in GHC's upcoming JS-backend
date: July 20, 2022
authors: [ doyougnu , sylvain ]
tags: [ghc, javascript, explanation, knowledge_engineering]
---

# Table of Contents

1.  [GHC Primitives](#orge86cd4a)
    1.  [The Easy Cases](#org75a0c27)
    2.  [ByteArray#, MutableByteArray#, SmallArray#, MutableSmallArray#,](#org5eb1aee)
    3.  [Addr# and StablePtr#](#org0de7f9e)
    4.  [Numbers: The Involved Case](#orgfa8aeb4)
        1.  [Working with 64-bit Types](#orgc9d245e)
        2.  [Unwrapped Number Optimization](#org453f9cc)
    5.  [But what about the other stuff!](#org2e2e79e)

One of the key challenges in any novel backend is representing GHC primitive
types in the new backend. For JavaScript, this is especially tricky, as
JavaScript only has 8 primitive types and some of those types, such as `number` do
not directly map to any Haskell primitive type, such as `Int8#`. This post walks
through the most important GHC primitives and describes our implementation for
each in the JavaScript backend. This post is intended to be an
explanation-oriented post, light on details, but just enough to understand how
the system works.


<a id="orge86cd4a"></a>

# GHC Primitives

There are 36 `primtype`s that GHC defines in `primops.txt.pp`:

1.  `Char#`
2.  `Int8#`, `Int16#`, `Int32#`, `Int64#`, `Int#`
3.  `Word8#`, `Word16#`, `Word32#`, `Word64#`, `Word#`
4.  `Double#`, `Float#`,
5.  `Array#`, `MutableArray#`,, `SmallArray#`, `SmallMutableArray#`
6.  `ByteArray#`, `MutableByteArray#`
7.  `Addr#`
8.  `MutVar#`, `TVar#`, `MVar#`,
9.  `IOPort#`, `State#`, `RealWorld`, `ThreadId#`
10. `Weak#`, `StablePtr#`, `StableName#`, `Compact#`, `BCO`,
11. `Fun`, `Proxy#`
12. `StackSnapshot#`
13. `VECTOR`

Some of these are unsupported in the JS-backend, such as `VECTOR` or lower
priority such as `StackSnapshot#`. We&rsquo;ll begin with the easy cases.


<a id="org75a0c27"></a>

## The Easy Cases

The easy cases are the cases that are implemented as JavaScript objects. In
general, this is the big hammer used when nothing else will do. We&rsquo;ll expand on
the use of objects&#x2014;especially representing heap objects&#x2014;in a future post,
but for the majority of cases we mimic the STG-machine behavior for GHC heap
objects using JavaScript heap objects. For example,

    var someConstructor =
        { f  =                   // entry function of the datacon worker
        , m  = 0                 // garbage collector mark
        , d1 = first arg         // First data field for the constructor
        , d2 = arity = 2: second arg // second field, or object containing the remaining fields
               arity > 2: { d1, d2, ...} object with remaining args (starts with "d1 = x2"!)
        }

This is the general recipe; we define a JavaScript object that contains
properties which correspond to the entry function of the heap object; in this
case that is the entry function, `f` for a constructor, some meta data for garbage
collection `m`, and pointers to the fields of the constructor or whatever else the
heap object might need. Using JavaScript objects allows straightforward
translations of several GHC types. For example `TVar`s and `MVar`s:

    // stg.js.pp
    /** @constructor */
    function h$TVar(v) {
        TRACE_STM("creating TVar, value: " + h$collectProps(v));
        this.val        = v;           // current value
        this.blocked    = new h$Set(); // threads that get woken up if this TVar is updated
        this.invariants = null;        // invariants that use this TVar (h$Set)
        this.m          = 0;           // gc mark
        this._key       = ++h$TVarN;   // for storing in h$Map/h$Set
    #ifdef GHCJS_DEBUG_ALLOC
        h$debugAlloc_notifyAlloc(this);
    #endif
    }

    // stm.js.pp
    function h$MVar() {
      TRACE_SCHEDULER("h$MVar constructor");
      this.val     = null;
      this.readers = new h$Queue();
      this.writers = new h$Queue();
      this.waiters = null;  // waiting for a value in the MVar with ReadMVar
      this.m       = 0; // gc mark
      this.id      = ++h$mvarId;
    #ifdef GHCJS_DEBUG_ALLOC
      h$debugAlloc_notifyAlloc(this);
    #endif
    }

Notice that both implementations defined properties specific to the semantics of
the Haskell type. JavaScript functions which create these objects follow the
naming convention `h$<something>` and reside in *Shim* files. *Shim* files are
JavaScript files that the JS-backend links against and are written in pure
JavaScript. This allows us to save some compile time by not generating code
which doesn&rsquo;t change, and decompose the backend into JavaScript modules.

This strategy is also how functions are implemented in the JS-backend. Function
objects are generated by `StgToJS.Expr.genExpr` and `StgToJS.Apply.genApp` but
follow this recipe:

    var myFUN =
     { f  = <function itself>
     , m  = <garbage collector mark>
     , d1 = free variable 1
     , d2 = free variable 2
     }

To summarize; for most cases we write custom JavaScript objects which hold
whatever machinery is needed as properties to satisfy the expected semantics of
the Haskell type. This is the strategy that implements: `TVar`, `MVar`, `MutVar` and
`Fun`.


<a id="org5eb1aee"></a>

## ByteArray#, MutableByteArray#, SmallArray#, MutableSmallArray#,

`ByteArray#` and friends map to JavaScript's
[`ArrayBuffer`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer)
object. The `ArrayBuffer` object provides a fixed-length, raw binary data
buffer. To index into the `ArrayBuffer` we need to know the type of data the
buffer is expected to hold. So we make engineering tradeoff; we allocate typed
views of the buffer payload once at buffer allocation time. This prevents
allocations from views later when we might be handling the buffer in a hot loop,
at the cost of slower initialization. For example, consider the `mem.js.pp`
shim, which defines `ByteArray#`:

    // mem.js.pp
    function h$newByteArray(len) {
      var len0 = Math.max(h$roundUpToMultipleOf(len, 8), 8);
      var buf = new ArrayBuffer(len0);
      return { buf: buf
             , len: len
             , i3: new Int32Array(buf)
             , u8: new Uint8Array(buf)
             , u1: new Uint16Array(buf)
             , f3: new Float32Array(buf)
             , f6: new Float64Array(buf)
             , dv: new DataView(buf)
             , m: 0
             }
    }

 `buf` is the payload of the `ByteArray#`, `len` is the length of the
`ByteArray#`. `i3` to `dv` are the _views_ of the payload; each view is an
object which interprets the raw data in `buf` differently according to type. For
example, `i3` interprets `buf` as holding `Int32`, while `dv` interprets `buf`
as a
[`DataView`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/DataView)
and so on. The final property, `m`, is the garbage collector marker.


<a id="org0de7f9e"></a>

## Addr# and StablePtr#

`Addr#` and `StablePtr#` are implemented as a pair of `ByteArray#` and an `Int#`
offset into the array. We&rsquo;ll focus on `Addr#` because `StablePtr#` is the
same implementation, with the exception that the `StablePtr#` is tracked in the
global variable `h$stablePtrBuf`. `Addr#`s do not have an explicit constructor,
rather they are implicitly constructed. For example, consider `h$rts_mkPtr`
which creates a `Ptr` that contains an `Addr#`:

    function h$rts_mkPtr(x) {
      var buf, off = 0;
      if(typeof x == 'string') {
    
        buf = h$encodeUtf8(x);
        off = 0;
      } else if(typeof x == 'object' &&
         typeof x.len == 'number' &&
         x.buf instanceof ArrayBuffer) {
    
        buf = x;
        off = 0;
      } else if(x.isView) {
    
        buf = h$wrapBuffer(x.buffer, true, 0, x.buffer.byteLength);
        off = x.byteOffset;
      } else {
    
        buf = h$wrapBuffer(x, true, 0, x.byteLength);
        off = 0;
      }
      return (h$c2(h$baseZCGHCziPtrziPtr_con_e, (buf), (off)));
    }

The function does some type inspection to check for the special case on
`string`. If we do not have a string then a `Ptr`, which contains an `Addr#`, is
returned. The `Addr#` is implicitly constructed by allocating a new
`ArrayBuffer` and an offset into that buffer. The `object` case is an idempotent
check; if the input is already such a `Ptr`, then just return the input. The
cases which do the work are the cases which call to `h$wrapBuffer`:

    // mem.js.pp
    function h$wrapBuffer(buf, unalignedOk, offset, length) {
      if(!unalignedOk && offset && offset % 8 !== 0) {
        throw ("h$wrapBuffer: offset not aligned:" + offset);
      }
      if(!buf || !(buf instanceof ArrayBuffer))
        throw "h$wrapBuffer: not an ArrayBuffer"
      if(!offset) { offset = 0; }
      if(!length || length < 0) { length = buf.byteLength - offset; }
      return { buf: buf
             , len: length
             , i3: (offset%4) ? null : new Int32Array(buf, offset, length >> 2)
             , u8: new Uint8Array(buf, offset, length)
             , u1: (offset%2) ? null : new Uint16Array(buf, offset, length >> 1)
             , f3: (offset%4) ? null : new Float32Array(buf, offset, length >> 2)
             , f6: (offset%8) ? null : new Float64Array(buf, offset, length >> 3)
             , dv: new DataView(buf, offset, length)
             };
    }

`h$wrapBuffer` is a utility function that does some offset checks and performs
the allocation for the typed views as described above.


<a id="orgfa8aeb4"></a>

## Numbers: The Involved Case

Translating numbers has three issues. First, JavaScript has no concept of
fixed-precision 64-bit types such as `Int64#` and `Word64#`. Second, JavaScript
bitwise operators only support _signed_ 32-bit values (except the unsigned
[right
shift](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Unsigned_right_shift)
operator of course). Third, numbers are atomic types and do not require any
special properties for correct semantics, thus using wrapping objects gains us
nothing at the cost of indirection.


<a id="orgc9d245e"></a>

### Working with 64-bit Types

To express 64-bit numerics, we simply use two 32-bit numbers, one to express
the high bits, one for the low bits. For example, consider comparing two `Int64#:`

    // arith.js.pp
    function h$hs_ltInt64(h1,l1,h2,l2) {
      if(h1 === h2) {
        var l1s = l1 >>> 1;
        var l2s = l2 >>> 1;
        return (l1s < l2s || (l1s === l2s && ((l1&1) < (l2&1)))) ? 1 : 0;
      } else {
        return (h1 < h2) ? 1 : 0;
      }
    }

The less than comparison function expects four inputs, two for each `Int64#` in
Haskell. The first number is represented by `h1` and `l1` (*high* and *low*),
and similarly the second number is represented by `h2` and `l2`. The comparison
is straightforward, we check equivalence of our high bits, if equal then we
check the lower bits while being careful with signedness. No surprises here.

For the bitwise operators we store both `Word32#` and `Word#` as 32-bit signed
values, and then map any values greater or equal `2^31` bits to negative values.
This way we stay within the 32-bit range even though in Haskell these types only
support nonnegative values.


<a id="org453f9cc"></a>

### Unwrapped Number Optimization

The JS backend uses JavaScript values to represent both Haskell heap objects and
unboxed values (note that this isn't the only possible implementation, see
[^1]). As such, it doesn't require that all heap objects have the same
representation (e.g. a JavaScript object with a "tag" field indicating its type)
because we can rely on JS introspection for the same purpose (especially
`typeof`). Hence this optimization consists in using a more efficient JavaScript
type to represent heap objects when possible, and to fallback on the generic
representation otherwise.

This optimization particularly applies to `Boxed` numeric values (`Int`, `Word`,
`Int8`, etc.) which can be directly represented with a JavaScript number,
similarly to how unboxed `Int#`, `Word#`, `Int8#`, etc. values are represented.

Pros:

- Fewer allocations and indirections: instead of one JavaScript object with a
  field containing a number value, we directly have the number value.

Cons:

- More complex code to deal with heap objects that can have different
  representations

The optimization is applicable when:

1.  We have a single data type with a single data constructor.
2.  The constructor holds a single field that *can only* be a particular type.

If these invariants hold then, we remove the wrapping object and instead refer
to the value held by the constructor directly. `Int8` is the simplest case for
this optimization. In Haskell we have:

    data Int8 = Int8 Int8#

Notice that this definition satisfies the requirements. A direct translation in
the JS backend would be:

    // An Int8 Thunk represented as an Object with an entry function, f
    // and payload, d1.
    var anInt8 = { d1 = <Int8# payload>
                 , f  : entry function which would scrutinize the payload
                 }

We can operationally distinguish between a `Thunk` and an `Int8` because these
will have separate types in the `StgToJS` GHC pass and will have separate types
(`object` vs `number`) at runtime. In contrast, in Haskell an `Int8` may
actually be a `Thunk` until it is scrutinized *and then* becomes the `Int8`
payload (i.e., call-by-need). So this means that we will always know when we
have an `Int8` rather than a `Thunk` and therefore we can omit the wrapper
object and convert this code to just:

    // no object, just payload
    var anInt8 = = <Int8# payload>

For the interested reader, this optimization takes place in the JavaScript code
generator module `GHC.StgToJS.Arg`, specifically the functions `allocConStatic`,
`isUnboxableCon`, and `primRepVt`.


<a id="org2e2e79e"></a>

## But what about the other stuff!

-   `Char#`: is represented by a `number`, i.e., the [code point](https://en.wikipedia.org/wiki/Code_point)
-   `Float#/Double#`: Both represented as a JavaScript Double. This means that
    `Float#` has excess precision and thus we do not generate exactly the same
    answers as other platforms which are IEEE754 compliant. Full emulation of
    single precision Floats does not seem to be worth the effort as of writing.
    Our implementation represents these in a `ByteArray#`, where each `Float#`
    takes 4 bytes in the `ByteArray#`. This means that the precision is reduced
    to a 32-bit Float.

[^1]: An alternative approach would be to use some JS ArrayBuffers as memory
    blocks into which Haskell values and heap objects would be allocated. As an
    example this is the approach used by the Asterius compiler. The RTS would
    then need to be much more similar to the C RTS and the optimization
    presented in this section wouldn't apply because we couldn't rely on
    introspection of JS values.