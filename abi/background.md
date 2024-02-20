# Background

Tools to assess an application binary interface (ABI) typically can output a diff, or
the result of a set of comparisons between one version of a library or another. Examples
of such tools include [libabigail](https://sourceware.org/libabigail/), and often the output
is dumped into the terminal, and in a format that is not easy to parse. This is a problem
of comparison. The problen can be simplified almost to an extreme, first for one library:

```
[library] -- extract facts --> [facts]
```

or for two libraries, for example that we want to compare:

```
[library A] -- extract facts --> [facts A]
                                           -- facts diff --> [change set] 
[library B] -- extract facts --> [facts B]
```

And more generally, we don't need to be talking about a library. The library in
the example above could be static code, version or metadata files, containers,
or binaries. Or even more simply:


```
A 
   --> A-B --> change set
B
```

In modern day tooling, if we imagine there is a continuum for understanding differences,
it might look like this:

1. Create a definition for facts (and a way to namespace them) (a composition)
2. Define a method for diff-ing two (or more) sets of facts (compspec)
3. Structure the output of the diff into a specification (diffspec)
4. Apply 1-3 to a specific domain to instantiate a compspec (examples below)
5. Create implementations that can take an input, derive and save facts, and optionally diff, and provide a humanly understandable output.

If you look at the above steps, you might guess the problem - our modern day tools
like libabigail started at step 5, and as a result the process and rules are a black box and
largely the output is "snowflake special" to libabigail and we can't easily understand,
extend, or programmatically use it.

## Rationale

I want to step back and work on steps 1-4. I believe that if we can better
define a language to express changes, and not just for one domain but any domain,
this will empower creators of tools to create implementations for software that can
easily perform a diff or extraction of facts. The only thing that would need to
be decided by the implementer are the domain specific details. 

### Application Binary Interfaces

With respect to ABI, if we expect to not only develop better tools and APIs for assessing it, we need
the following:

1. To write down a list of humanly understandable checks
2. To provide a format for representing these facts (a compspec).
3. To provide a format for showing differences between these facts (a changespec).

This repository is an attempt to define such a standard for points 2 and 3, and to think
about design. If we are attempting to model complex software there is likely no way around needing
a graph design. But perhaps we can "flatten" the graph into a set of facts with associated
identifiers and then give the entire thing to a solver (as a faster solution than
manual parsing). I think using [clingo](https://potassco.org/clingo/) should work nicely.
We can also try more traditional graph methods, if clingo is not sufficient.

### Use Cases

This standard or spec should be able to support:

1. changes in version files
2. changes in software static / source code or binaries
3. changes in package manager metadata
4. changes in container metadata (or interactions)
