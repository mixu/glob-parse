# glob-parse

Returns a parsed representation of a glob string.

## Use cases

- Works on any string, does not require Minimatch or any other separate glob library.
- Does not perform glob matching: it just parses a glob expression into segments and produces relevant metadata about those segments.
- The primary use case is to parse a glob into segments for use in a glob-based file resolver like wildglob. To recursively resolve against a set of file paths (which grow in length as the traversal progresses) you need to split the glob expression into logical pieces.
- Another use case is to extract the base path from a glob.
  - either the "minimum base path" which is the portion of the path prior to any special characters, e.g. `{./*/*,/tmp/glob-test/*}` => `''`. This is what gulp's `glob2base` does but it depends on Minimatch.
  - or the expanded base path, e.g. `{./*/*,/tmp/glob-test/*}` => `[ './', '/tmp/glob-test/' ]`

## Algorithm

A glob expression is an expression which can potentially match against any path in the file system. For example `/foo/**` basically says take the whole file system and compare it against that expression, and return the result.

However, stat'ing the whole file system is obviously inefficient, since we can use the glob expression itself to narrow down the potential matches:

- first, we can parse the path prefix. This is the part of the expression which can be safely expanded into a definite path string (or multiple path string).
- next, we start a directory traversal starting from the path prefix.
- if we come across a directory entry, then before recursing into the directory we will check whether a partial (path segment-by-path-segment) match against the directory fails. If the partial match fails all include globs, then any files inside that path will also never match.
- every fs entry which is found will be matched against the include globs, then the any exclude globs.

Two types of globs:

- absolute globs: must be matched against the absolute path of each file
- relative globs: must be matched against the relative path of each file

Partial matching works like this:

- partial matching accepts a path prefix, typically a directory which can potentially be recursed into using fs.readdir.
- the purpose of partial matching is to definitively answer false if the given pattern does not match, and to return true if either the current prefix fully matches a pattern or if the pattern matches up until the end of the prefix with some additional unknown patterns.
- for an include glob, if a partial match returns false we know that the directory does not need to be processed further as no paths under it can match the pattern.
- for an include glob, if a partial match returns true we will need to process the directory further.
- for an exclude glob, if a partial match returns false, we know that the directory will need to be processed, assuming at least one include glob had a partial match.
- for an exclude glob, if a partial match returns true, we cannot safely conclude anything, as some more tricky patterns may still return a match when given a longer path to work with.

As you may notice, only `false` results in partial matching are useful in pruning the directory traversal subdirectories and partial matching `false` results from exclude globs are not useful so it makes sense to focus on include globs.


AAAA

- any patterns containing a globstar are only matched for the definite portion preceding the globstar expression. This is exactly the right behavior if there is only one globstar at the end of the glob, and overly conservative for more complex globstar expression. Globstar expressions can match in fairly complex ways (and there can be many of them), so we'll accept a hit in terms of performing fs.stat calls; this works out OK compared to Minimatch which does a more exact but more CPU-intensive matching.
- patterns containing: `[...]`, `[^...]`, `?`, `*`, `{a,b}`, `{0..9}` will be matched on a segment by segment basis against the path.
- any patterns containing: `?(...)`, `*(...)`, `+(...)`, `@(...)`, `!(...)` are only matched for the definite portion preceding the extglob expression.

