# Whelp specification

As input, the Whelp compiler takes a non-empty array of paths to source files and a possibly empty set of _options._  Each option is a string of length [1, 255] that contains only US-ASCII alphanumerics and underscores with the first character being anything but a number.  The order of options does not matter.  Including the same option more than once is no different than including it only once.

Whelp interprets each source file in the order given.  The result of interpretation is a dependency graph, where the nodes are _modules_ and the relationships are one-way _dependencies._  A single source file may define multiple modules.

Exactly one module has the special name `main`.  In the output, the `main` module will always be the last module in the compilation.  The rest of the compilation is all the direct and indirect dependencies of the `main` module, with the restriction that if module A depends on module B, module B must occur before module A in the compilation.  Circular dependencies cause an error if they prevent the formation of a coherent compilation.

Each module has an associated _payload,_ which is an arbitrary sequence of bytes.  Payloads may be embedded in source files, or stored in external files, or generated dynamically by a script in the source file.  The final compilation is the binary concatenation of the payloads of each module in the compilation.

The rest of this specification describes the data formats and compilation process in further detail.

## Source file format

Whelp source files are valid Shastina files.  The specification for Shastina and a parsing library is available at the [libshastina](https://github.com/canidlogic/libshastina/) project.

According to the Shastina specification, Shastina parsing must stop when it reaches the special `|;` token that ends the Shastina file.  However, the Shastina specification allows for additional data to be present after this final `|;` token, which Shastina will never parse.  Whelp takes advantage of this feature by allowing for a module payload to be embedded after the final `|;` token.  Only one module payload may be embedded per source file.

If a module payload is embedded, then it consists of a possibly empty sequence of bytes.  The module payload does _not_ need to be UTF-8 text.  However, the module payload must be separated from the `|;` token by _padding._  The padding is a sequence of zero or more US-ASCII Space (SP) or Horizontal Tab (HT) characters, followed by a US-ASCII Line Feed (LF).  The Line Feed (LF) may optionally be preceded by a US-ASCII Carriage Return (CR).  The first byte of the payload is the first byte after the LF.  If the LF is the last byte in the file, the embedded payload is empty.

## Whelp header

The first four Shastina entities parsed from a source file must form a valid _Whelp header_ such as the following:

    %whelp 1.0;

More specifically, the first four Shastina entities must be:

    1. BEGIN METACOMMAND %
    2. META TOKEN whelp-source
    3. META TOKEN [version]
    4. END METACOMMAND ;

The `[version]` is the version number of the Whelp source format.  This consists of an unsigned decimal integer, a period, and another unsigned decimal integer.  The first integer is the major version number and the second integer is the minor version number.

This specification describes Whelp version 1.0.  If the major version is something other than one, then Whelp will not attempt to parse any further and will report that the version is unsupported.  If the major version is one but the minor version is greater than zero, then Whelp will continue on parsing.

### Forward compatibility

The `[version]` declaration in the Whelp header combined with conditional structures described in later sections allow this Whelp specification to be forward-compatible with future versions.  This subsection explains how forward compatibility works.  It is only necessary to understand this section if you are writing a future version of the Whelp specification.

Suppose that a future version of Whelp adds a feature that is not supported by earlier versions of the Whelp compiler.  Let us refer to these features as _future features._

To add a future feature in a manner that preserves backwards compatibility with earlier Whelp compilers, the minor version number reported in the `[version]` declaration should be increased, but the major version number should remain the same.  This allows earlier Whelp compilers to continue parsing the file.

Conditional blocks can then be used hide future features from earlier Whelp compilers that do not support them.  If future features are required, conditional blocks combined with the `error` operator described later can be used to add a version check that will cause an error if the Whelp compiler is not recent enough.

The check done in these conditional blocks can follow two different strategies.  The first strategy is to use the `wlevel` operator described later to check whether the running Whelp compiler is at least a certain version.  The second strategy is to define built-in options corresponding to certain features.  If the built-in option is set, it indicates the corresponding feature is supported.  To avoid conflicting with source files for earlier versions that might coincidentally already use that option name as a generic option, built-in options should only be set for source files declaring a `[version]` that understands the relevant built-in option.

## Interpreter state

The Whelp interpreter that is used to run the Shastina scripts at the start of source files has a well-defined virtual machine state.  The Whelp virtual machine state has the following components:

1. Option set
2. Atom table
3. Namespace
4. Stack

The following subsections describe each of these virtual machine state components in detail.

### Option set

The _option set_ is a set of unique strings.  The option set may be empty.  Each option string is a string of length [1, 255] that contains only US-ASCII alphanumerics and underscores with the first character being anything but a number.

The option set is defined by command-line parameters provided to the Whelp compiler.  This allows different combinations of options to be used during compilations.  Conditional structures can then cause the compilation to be different without requiring any changes to the source files.

Options can also be used to indicate which features are supported by the Whelp compiler, to provide for future compatibility as explained earlier.  Feature options should not be defined by a Whelp compiler unless the Whelp source file declares a `[version]` that is aware of those specific feature options.

### Atom table

The _atom table_ is a mapping of unique string keys to unique integer values.

Each string key is a string of length [1, 255] that contains only US-ASCII alphanumerics and underscores with the first character being anything but a number.  Each string key is unique in the atom table.  Comparison is case sensitive.

Unique integer values are assigned to each string key in the order they are defined.  The first string key gets unique integer value zero, the second string key gets unique integer value one, and so forth.

Atoms are used for things like module names, where the name itself isn't important but the only thing that matters is whether two atoms compare as equal or not equal.

In Whelp source files, atoms are represented by Shastina string entities that enclose the string key within double quotes.  If the given string key does not exist in the atom table, a new string key mapping is added to the table using a unique integer and this unique integer stands in for the atom.  If the given string key already exists in the atom table, the existing integer mapping will stand in for the atom.

Although atoms are actually integers on the interpreter stack, the atom and integer type are kept separate from each other.

### Namespace

The _namespace_ is a mapping of unique string keys to values.  The values can be any value type that is allowed on the interpreter stack.  Each mapping record also has a flag indicating whether or not it is a constant.  The Shastina declare variable entity, declare constant entity, get variable or constant entity, and assign variable entities all use the namespace to store variables and constants.  Variables and constants share the same namespace.

Variables and constants must be declared before they can be used in get or assign entities.  Also, it is an error to use the assign entity with a constant.

Variable and constant names are strings of length [1, 255] that contain only US-ASCII alphanumerics and underscores with the first character being anything but a number.  Comparison is case sensitive.

### Stack

The _stack_ is the core data structure used to run Shastina scripts.  Shastina operator entities pop their arguments off the interpreter stack and push their return values back on the interpreter stack.  Different Shastina entities can push literal values onto the stack and check the state of the stack.

The stack always starts out empty at the start of each Shastina script.  At the end of each Shastina script, the interpreter stack must be empty or an error will occur.  Whelp interpreters must support at least 255 elements on the stack.

The following data types may be stored on the Whelp interpreter stack:

1. Booleans
2. Integers
3. Atoms
4. Blobs

_Booleans_ have only two values:  true and false.

_Integers_ support a range of [-9007199254740991, 9007199254740991].  This is the range of integers that can be safely represented with double-precision floating-point numbers.  When used as file offsets and length values, this range supports files of more than 8,000 terabytes.

_Atoms_ are unique integer values that stand in for string keys using the mapping defined in the atom table.  Although atoms are stored as integers, their numeric value is not relevant and can not be accessed, except to determine whether two atoms are equal.

_Blobs_ are sequences of zero or more bytes.  Blob objects are abstractions that provide their total length in bytes and a method to output a requested byte range to a compilation.  Specific types of blobs might be based on literal data stored in memory, data generated dynamically, data stored in an embedded payload within the current source file, or data stored within an external file.  It is also possible to have blobs that compile multiple existing blobs together in various ways or apply transformations to existing blobs.

One special category of blob is called a _filter blob._  Filter blobs differ from other blobs in that they are unable to determine their total length in bytes and that they can only output byte ranges that begin at offset zero.  This is the case for certain transformations.  Blobs that are built from existing blobs may or may not support using filter blobs as source blobs.

## Interpreter output

The result of interpreting a Whelp source file is a set of _modules._  Each module has a unique name, a set of module names it depends on, and a blob defining its data.  Each module's name must be unique across all the module definitions in all the source files fed into the Whelp compiler.  Furthermore, after all the Whelp source files have been interpreted, there must exist a module with the special name `main` (case sensitive).

After all the modules have been defined by interpreting each source file, the next step is to resolve dependencies.  In order to resolve dependencies, the Whelp compiler begins with the module named `main`.  Each of the dependencies declared by `main` is added into the _unresolved set._  Dependencies of modules in the unresolved set are recursively visited until the unresolved set contains all modules that are either directly or indirectly depended on by `main`.

To determine the order of modules in the compilation, Whelp performs the following reduction procedure until the unresolved set is empty.  Begin with the _order of modules_ as an empty array.  First, choose a module that has no dependencies.  If no such module exists, Whelp compilation fails due to circular dependencies.  If a module without dependencies exists, remove it from the unresolved set, append it to the order of modules, and then remove the name of that module from the dependency sets of all modules remaining in the unresolved set.  Continue with this reduction procedure until either the unresolved set becomes empty or the process fails due to circular dependencies.

Once the order of modules, the Whelp compiler emits the compiled binary.  This is the concatenation of the binary data in the blobs for each module in the order of modules, in the order given in the order of modules.

## Shastina entities

The following Shastina entity types are supported by the Whelp interpreter after the Whelp header:

    SNENTITY_EOF          End Of File
    SNENTITY_STRING       String literal
    SNENTITY_NUMERIC      Numeric literal
    SNENTITY_VARIABLE     Declare variable
    SNENTITY_CONSTANT     Declare constant
    SNENTITY_ASSIGN       Assign value of variable
    SNENTITY_GET          Get variable or constant
    SNENTITY_BEGIN_GROUP  Begin group
    SNENTITY_END_GROUP    End group
    SNENTITY_ARRAY        Define array
    SNENTITY_OPERATION    Operation

It is possible to use unsupported Shastina entities without causing an error if they are hidden by conditionals.  This is useful for forward compatibility.

The following subsections describe the behavior of each of these Shastina entities.

### EOF entity

The EOF entity `|;` indicates the end of the Shastina script.  Therefore, it is always the last entity in the script.  You can not block an EOF entity with a conditional, so there must be exactly one EOF entity.

When the EOF entity is encountered, there must not be any open conditional structure and the interpreter stack must be empty.

### String entity

Shastina string entities are used for atoms, option queries, and declaring literal binary blob values.

#### Atoms

Atoms use double-quoted strings without any string prefix.  The quoted string must follow the string key format specified earlier in the atom table section.  If no atom with the given name currently exists, a new mapping is added and then an atom using that unique integer is pushed onto the stack.  If an atom with the given name already exists, the unique integer already mapped to that name is pushed onto the stack as an atom.

#### Option queries

Option queries use double-quoted strings with a `q` string prefix.  The quoted string must follow the option name format specified earlier in the option set section.  If an option with that name exists in the option set, a boolean value of true is pushed onto the stack.  Otherwise, a boolean value of false is pushed onto the stack.

#### Textual literal blobs

Literal blobs use curly-quoted strings.  There are three styles of literal blobs, which are distinguished by their string prefix.

If there is no string prefix in front of the curly brace, the text style of literal blob will be used.  Line breaks are allowed within these literal blobs, and the line breaks will be included in the literal blob data, unless blocked by an escape.  Line breaks are always included as lone US-ASCII Line Feed (LF) characters, even if the source file uses CR+LF line breaks.  You must attach a filter blob to the constructed literal blob if you want CR+LF line breaks.

The following escape sequences are supported within the text style of literal blobs:

    \\       - literal backslash
    \{       - unbalanced left curly
    \}       - unbalanced right curly
    \.       - ignore line break
    \t       - horizontal tab
    \r       - carriage return
    \n       - line feed
    \u####   - UTF-8 4-digit escape
    \U###### - UTF-8 6-digit escape
    \x##     - byte value escape

You may used curly braces in unescaped form only if they are properly balanced.  The escapes for curly braces cause a curly brace to be included but not to be counted for purposes of balancing curly braces.

The _ignore line break_ escape causes any following ASCII Space (SP), Horizontal Tab (HT), Carriage Return (CR), and Line Feed (LF) characters to be skipped, up to and including the first LF that is encountered.  If no LF is encountered before the end of the literal, the rest of the literal is skipped.  It is an error if anything but SP and HT is present before the line break or the end of the literal, and CR may only occur immediately before LF.  The _ignore line break_ escape is useful for adding a line break in the source file that will not be present in the literal data.

UTF-8 characters can be directly included in the literal.  Properly paired surrogates will be corrected into the appropriate direct UTF-8 encoding of supplementals.  You may also specify Unicode codepoints to be directly included in the literal in UTF-8 format by using the `\u` or `\U` escapes.  Both of these take a sequence of base-16 digits (in lowercase, uppercase, or any mixture) specifying a non-surrogate Unicode codepoint.  The only difference between the two escapes is that the first takes four digits and the second takes six digits (which allows for supplemental codepoints).  Literal blobs, however, are binary strings, so specified Unicode codepoints will be translated into UTF-8 byte sequences in the literal blob.

You may also directly specify byte values using the `\x` escape, which takes two base-16 digits (lowercase, uppercase, or any mixture) specifying a literal byte value to include in the literal binary string.

#### Base-16 literal blobs

Curly-braced strings that have the string prefix `b16` (case sensitive) use a binary base-16 format.  In these literal strings, all ASCII Space (SP), Horizontal Tab (HT), Carriage Return (CR), and Line Feed (LF) characters are ignored.  The only characters allowed besides those ignored whitespace characters are base-16 digits `0-9`, `A-F`, and `a-f`, with no difference in meaning between uppercase and lowercase letters.  The total number of base-16 characters must be even.  Each pair of base-16 characters specifies the literal value of a byte to include within the literal blob.  If there are no base-16 digits in the curly braces, the literal blob will be an empty sequence of no bytes.

#### Base-64 literal blobs

Curly-braced strings that have the string prefix `b64` (case sensitive) use a binary base-64 format.  In these literal strings, all ASCII Space (SP), Horizontal Tab (HT), Carriage Return (CR), and Line Feed (LF) characters are ignored.  The only characters allowed besides those ignored whitespace characters are the base-64 alphabet `A-Z` `a-z` `0-9` two extra characters, and the optional padding character `=`.  The two extra characters may be either the standard `+` and `/` or the alternatives `-` and `_` with no difference in meaning, and any kind of mixture of the two systems allowed.  At the end of the string, padding `=` characters may or may not be present.  If present, they must be valid in the base-64 format.  If no characters of the base-64 alphabet are present in the curly braces, the literal blob will be an empty sequence of no bytes.

### Numeric entity

Shastina numeric entities push a signed integer literal onto the stack.  The range of values is [-9007199254740991, 9007199254740991].  The `+` sign is optional for positive integers.  Zero may have either `+` or `-` or neither sign, with no difference in meaning.

### Variable and constant entities

The _declare variable, declare constant, assign value of variable,_ and _get variable or constant_ entities are used to manipulate the namespace state.  The namespace starts out empty.  You can declare variables and constants using the _declare variable_ (`?`) and _declare constant_ (`@`) Shastina entities, respectively.  Both of these entities pop a value off the stack of any type and use this to initialize the value of the name in the namespace.

Variables and constants share the same namespace.  See the earlier section on the namespace for details of the variable and constant name format, which is case sensitive.  It is an error to attempt to redeclare a name that has already been defined in the namespace.

The _get variable or constant_ entity (`=`) pushes a copy of the value associated with a variable or constant onto the stack.  It is an error to attempt to get the value of a name that has not been declared yet.

The _assign value of variable_ entity (`:`) pops a value off the stack and uses it to replace the current value associated with the given name in the namespace.  The given name must have already been declared as a variable.  It is an error to attempt to assign a value to a name that has not been declared, or that has been declared as a constant.

### Group entities

The _begin group_ and _end group_ entities perform integrity checks on the stack.  Each _begin group_ entity must be paired with an _end group_ entity that follows it.  Groups may be nested.

When a _begin group_ entity is encountered, the existing stack will be hidden, so that it is as if the group started with an empty stack.  It is an error to attempt to pop values off the part of the stack that has been hidden by a group.  When the _end group_ entity is encountered, a check will be made that exactly one value is on the part of the stack that has not been hidden, with an error otherwise.  After that check passes, the part of the stack that was hidden is restored, with the single new value on top.

### Array entities

Shastina arrays are delimited by `[` and `]` tokens, and array elements are separated by `,` tokens.  Arrays may be nested.  Each array element implicitly contains a group, except for the special case of an empty array.  At the end of the array, each array element will be on the stack with the first element of the array deepest in the stack.  At the end of the array, an integer that counts the number of elements in the array will always be pushed.

Array notation is a shorthand.  You can equivalently push the count of elements as an integer literal after pushing each array element.

Shastina arrays are useful for operations that take variable numbers of inputs along with an integer count.

### Operation entities

Operation entities perform operations.  Each operation may pop parameters off the interpreter stack and push return values back onto the interpreter stack.  The available operations are described in later sections.  Operators are specified with the following syntax:

    [p_1] [p_2] ... [p_n] opname [ret_1] [ret_2] ... [ret_n]

The operator name is in the center.  The parameters that popped off the stack are given to the left of the operator name, and the parameters that will be pushed back onto the stack are given to the right of the operator name.  In this example, `[p_n]` will be on top of the stack when the operator `opname` is invoked, and `[ret_n]` will be on top of the stack when the operator invocation is complete.  If there are no parameters for the operator, a hyphen `-` is given to the left of the operator name.  If there are no return values from the operator, a hyphen `-` is given to the right of the operator name.

## Whelp operators

The following subsections document the operator entities that are available in Whelp scripts.  The operator syntax is described with notation that was specified in the earlier "Operation entities" section.
