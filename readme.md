# glob-parse

Returns a parsed representation of a glob string.

## Use cases

- Works on any string, does not require Minimatch or any other separate glob library.
- Does not perform glob matching: it just parses a glob expression into segments and produces relevant metadata about those segments.
- The primary use case is to parse a glob into segments for use in a glob-based file resolver like wildglob. To recursively resolve against a set of file paths (which grow in length as the traversal progresses) you need to split the glob expression into logical pieces.
- Another use case is to extract the base path from a glob.
  - either the "minimum base path" which is the portion of the path prior to any special characters, e.g. `{./*/*,/tmp/glob-test/*}` => `''`. This is what gulp's `glob2base` does but it depends on Minimatch.
  - or the expanded base path, e.g. `{./*/*,/tmp/glob-test/*}` => `[ './', '/tmp/glob-test/' ]`
