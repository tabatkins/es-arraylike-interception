# Numeric-Property Interception (Array-likes)

This is a proposal to add a "proxy-lite" functionality to JS objects that allows userland code to intercept gets and sets of arbitrary "numeric" indexes, exactly as if they'd installed a getter/setter for the property.

General Justification
---------------------

WebIDL extensively uses type-checking throughout the APIs written in it.  Method arguments are type-checked, properties are type-checked, and any violation of this results in an error being thrown, right at the point of the violated contract (or alternately, invokes some automatic coercion right at the boundary between "JS" and "IDL").  All of this is done consistently and, more importantly, *automatically*, so that browsers have a reasonably consistent behavior for this kind of thing between different APIs, which authors can depend on.

Due to JS's design, all of this can be done within the bounds of idiomatic JS. Method arguments are obvious (it's just the first code in the method), but properties are done via a getter/setter pair on the prototype.  When you need arbitrary keys, JS now has Maps and Sets; while these aren't *directly* usable via WebIDL (for interesting design reasons), WebIDL at least allows spec authors to create Map/Set look-alikes, which use the same method signatures and can generally be treated like the real thing.  So far so good.

The one exception, the one major JS object type that can't be reasonably emulated in WebIDL with WebIDL's typechecking semantics, is the humble Array.  Authors expect arrays to work with the standard array syntax - getting and setting with `foo[0]` - and to have the full set of Array methods.  WebIDL can do the latter, but it can't do the former without manually installing a bunch of numeric-key getter/setter pairs to exactly match the indexes that it expects (and then you can't add or remove from the array in a reasonable way).  The only way to make this work is to go "full Proxy" - using a Proxy you can intercept arbitrary property reads/writes, and keep `.length` up to date, etc.  Unfortunately, Proxies also allow *a whole lot more*, and as such end up being relatively expensive in engines.  Not good!

People have been struggling with doing Array-likes in WebIDL for a long time, and have never come up with a good answer. This proposal hopes to finally solve this problem.

### Specific Use-Cases ###

The CSS Houdini APIs would like to provide lists of objects in the Typed OM, where those objects are guaranteed to be of particular types.  In particular, [`CSSUnparsedValue`](https://drafts.css-houdini.org/css-typed-om/#unparsedvalue-objects) would like to represent itself as a list of strings and `CSSVariableReferenceValue` objects, and [`CSSTransformValue`](https://drafts.css-houdini.org/css-typed-om/#csstransformvalue) would like to be a list of `CSSTransformComponent` objects.

There are many more use-cases scattered thruout the web platform, but these two are actively on my plate.  For example, [`FileList`](http://dev.w3.org/2006/webapi/FileUpload/publish/FileAPI.html#FileList-if) and [`DOMTokenList`](https://dom.spec.whatwg.org/#interface-domtokenlist) are both Array-like, but fairly clumsy in implementation as a result.

Possible Approaches
-------------------

1. Just use Proxies, but with optimizations (either transparent, or explicit new Proxy-creation APIs) when you're only trapping certain operations. [Mark Miller discusses that here.](https://github.com/tvcutsem/es-lab/issues/21) In [this recent es-discuss thread](https://esdiscuss.org/topic/intercepting-sets-on-array-like-objects), aklein suggests that V8, at least, can already optimize integer-property-setting-only APIs.

2. Go the [Object Model Reformation route](https://web.archive.org/web/20160425220917/http://wiki.ecmascript.org/doku.php?id=strawman:object_model_reformation), and provide three Symbol-keyed methods on Arrays that are called whenever a numeric property is get/set/deleted.
