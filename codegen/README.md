Codegen works via a human-in-the-loop system. It's quite challenging to build a codegen engine that can correctly
infer what parameters should be sent and received so we instead have a two-step process.

First, `annotationgen.py` reads in all the `nvml.h`, `cuda.h`, et al headers and copies the function signatures
into `annotations.h`. This file is intended to be modified by humans. In particular, the `@param` annotations
have significant meanings.

Specifically, the order of `@param` annotations indicates the order in which the parameters are sent or received.
`SEND_ONLY`, `RECV_ONLY`, and `SEND_RECV` indicate the directions the parameter is transferred in. Other arguments
available are `NULL_TERMINATED` (to indicate that this is a null-terminated string), or `LENGTH:<param>` and
`SIZE:<value>` to specify the size (aka width) of the parameter. If `LENGTH:<param>` is specified, `<param>` must
be placed in front of the parameter referencing it, otherwise the generated code will not compile.

With the annotations in place, `codegen.py` reads in the annotations and generates the RPC server and client.

The motivation for this approach is grounded in codegen being very good at ensuring that the RPC server and client
match in behavior but not very good at determining the specifics of what should be sent and received. Humans, on
the other hand, are very good at the latter but not the former. Therefore, this specification file allows humans
to edit and build out the "layout" of the RPC call without having to worry about ensuring the server and client
remain at parity.

## Why a custom wire format?

One might ask why not use something like a protobuf instead of this custom wire format? The main reason is we would
end up needing to generate the proto file anyways since the volume of requests and responses is way too high.
Ultimately, that would end up with substantially more code and another layer of abstraction.

Additionally, using protobuf makes it challenging, but not impossible, to transfer opaque blocks of memory and cast
them as protobuf has its own opinion on memory layout. One advantage of a custom wire format is we can read from
the RPC directly into the destination, saving us a byte shuffling layer.

Lastly, doing something very custom in protobuf is not nearly as easy. For example, `cudaMemcpy` requires a completely
custom implementation as the wire format is dependent on the `kind` parameter. This is straightforward to do as
we can stub out the implementation with our custom codegen but if we were to tie ourselves to protobuf we would
need an escape hatch.

## Future Improvements

Some improvements that can be made:

- [ ] Currently, the RPC ID is not deterministic. This is fine for now as we are still in demo-phase but this won't work for backwards compatibility.
- [ ] We could use C++ annotations to make the processing a little more "C++"-y. Worth investigating for a bit.
- [ ] Generate `cublas` and other friends.
