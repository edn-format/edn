edn
===

extensible data notation [eed-n]

# Rationale

**edn** is an extensible data notation. A superset of **edn** is used by Clojure to represent programs, and it is used by Datomic and other applications as a data transfer format. This spec describes **edn** in isolation from those and other specific use cases, to help facilitate implementation of readers and writers in other languages, and for other uses.

**edn** supports a rich set of built-in elements, and the definition of extension elements in terms of the others. Users of data formats without such facilities must rely on either convention or context to convey elements not included in the base set. This greatly complicates application logic, betraying the apparent simplicity of the format. **edn** is simple, yet powerful enough to meet the demands of applications without convention or complex context-sensitive logic.

# Spec

Currently this specification is casual, as we gather feedback from implementors. A more rigorous e.g. BNF will follow.

## General considerations

Elements are generally separated by whitespace. Whitespace is not otherwise significant, nor need redundant whitespace be preserved during transmissions. Commas `,` are also considered whitespace, other than within strings.

The delimiters `{ } ( ) [ ]` need not be separated from adjacent elements by whitespace.

There is no enclosing element at the top level. Thus **edn** is suitable for streaming and interactive applications.

## Built-in elements

### nil

`nil` represents nil, null or nothing. It should be read as an object with similar meaning on the target platform.

### booleans

`true` and `false` should be mapped to booleans.

### strings

Strings are enclosed in `"double quotes"`. May span multiple lines. Standard C/Java escape characters `\t \r \n` are supported.

### characters

Characters are preceded by a backslash: `\c`. `\newline`, `\space` and `\tab` yield the corresponding characters.

### symbols

Symbols are used to represent identifiers, and should map to something other than strings, if possible.

Symbols begin with a non-numeric character and can contain alphanumeric characters and `. * + ! - _ ?`. If `-` or `.` are the first character, the second character must be non-numeric. Additionally, `: #` are allowed as constituent characters in symbols but not as the first character. 

`/` has special meaning in symbols. It can be used once only in the middle of a symbol to separate the _prefix_ (often a namespace) from the _name_, e.g. `my-namespace/foo`. `/` by itself is a legal symbol.

### keywords

Keywords are identifiers that typically designate themselves. They are semantically akin to enumeration values. Keywords follow the rules of symbols, except they can (and must) begin with a colon, e.g. `:fred` or `:my/fred`. If the target platform does not have a keyword type distinct from a symbol type, the same type can be used without conflict, since the mandatory leading `:` of keywords is disallowed for symbols..

### integers

Integers consist of the digits `0` - `9`, optionally prefixed by `-` to indicate a negative number. An integer can have the suffix `N` to indicate that arbitrary precision is desired.

### floating point numbers

64-bit (double) precision is expected.

    floating-point-number
      int frac
      int exp
      int frac exp
    int
      digit
      1-9 digits 
      - digit
      - 1-9 digits
    frac
      . digits
    exp
      ex digits
    digits
      digit
      digit digits
    ex
      e
      e+
      e-
      E
      E+
      E-

In addition, a floating-point number may have the suffix `M` to indicate that exact precision is desired.

### lists

A list is a sequence of values. Lists are represented by zero or more elements enclosed in parentheses `()`. Note that lists can be heterogeneous.
 
    (a b 42)

### vectors

A vector is a sequence of values that supports random access. Vectors are represented by zero or more elements enclosed in square brackets `[]`. Note that vectors can be heterogeneous.

    [a b 42]

### maps

A map is a collection of associations between keys and values. Maps are represented by zero or more key and value pairs enclosed in curly braces `{}`. No semantics should be associated with the order in which the pairs appear.

    {:a 1, "foo" :bar, [1 2 3] four}

Note that keys and values can be elements of any type. The use of commas above is optional, as they are parsed as whitespace.

### sets

A set is a collection of unique values. Sets are represented by zero or more elements enclosed in curly braces preceded by `#` `#{}`. Note that sets can be heterogeneous.

    #{a b [1 2 3]}

## Tagged elements

**edn** supports extensibility through a simple mechanism. Symbols prefixed by `#` are designated as **_tags_**. A tag indicates the semantic interpretation of _the following element_. It is envisioned that a reader implementation will allow clients to register handlers for specific tags. Upon encountering a tag, the reader will first read the next element, then pass the result to the corresponding handler for further interpretation, and the result of the handler will be the data item yielded by the tag + tagged element. This will bottom out on elements either understood or built-in. 

Thus you can build new distinct readable elements out of (and only out of) other readable elements, keeping extenders and extension consumers out of the text business.

The semantics of a tag, and the type and interpretation of the tagged element are defined by the steward of the tag.

    #myapp/Person {:first "Fred" :last "Mertz"}

If a reader encounters a tag for which no handler is registered, the implementation can either report an error, call a designated 'unknown element' handler, or create a well-known generic representation that contains both the tag and the tagged element, as it sees fit.

### Rules for tags

Tag symbols without a prefix are reserved by **edn** for built-ins defined using the tag system. 

User tags _**must**_ contain a prefix component, which must be owned by the user (e.g. trademark or domain) or known unique in the communication context.

A tag _may_ specify more than one format for the tagged element, e.g. both a string and a vector representation.

## Built-in tagged elements

### #inst "rfc-3339-format"

An instant in time. The tagged element is a string in [RFC-3339](http://www.ietf.org/rfc/rfc3339.txt) format.

`#inst "1985-04-12T23:20:50.52Z"`

### #uuid "f81d4fae-7dec-11d0-a765-00a0c91e6bf6"

A [UUID](http://en.wikipedia.org/wiki/Universally_unique_identifier). The tagged element is a canonical UUID string representation.

## Comments

If a `;` character is encountered outside of a string, that character and all subsequent characters to the next newline should be ignored.

## Discard

If the sequence `#_` is encountered outside of a string, the next element should be read and discarded. Note that the next element must still be a readable element. A reader should not call user-supplied tag handlers during the processing of the element to be discarded.

    [a b #_foo 42] => [a b 42]

