# Structured diagnostics

A limitation of the current compiler diagnostics (error messages, warnings, time
profile information and other compiler developer debugging output) is that it
cannot be parsed easily and safely by developer tools.

This means that not only those tools have the choice to either parse the
diagnostics produced by the compiler, which is brittle; or lose meaningful
information.

Consider for instance, a moderately large error message:
```ocaml
File "err.ml", line 2, characters 11-33:
2 | module R = F(struct type y end)()
               ^^^^^^^^^^^^^^^^^^^^^^
Error: This application of the functor F is ill-typed.
       These arguments:
         $S2 ()
       do not match these parameters:
         functor (X : $T1) (Y : ...) () -> ...
       1. An argument appears to be missing with module type
              $T1 = sig type x end
       2. Module $S2 matches the expected module type
       3. Module () matches the expected module type
```
The internal compiler report data types for this error messages contains
an explicit locations, one main message
```ocaml
This application of the functor F is ill-typed.
       These arguments:
         $S2 ()
       do not match these parameters:
         functor (X : $T1) (Y : ...) () -> ...
```

and three submessages.

All this information is lost when printed as a plain text. Even worse for tools,
the compiler is using `Format` to shape the output to adjust to a 78-column
width and semantic tags to control the styling of the text through ANSI terminal
escape sequence. All this work is lost for developer tools that will try to render
the error message in another text shape, with another styling context. 

The aim of this RFCs is to describe a way to add "structured diagnostics" to
the compiler as a way to render the internal compiler ADT in any structured file formats.
For instance, if rendered to s-expression, the following error message should look likes:

```lisp
((metadata ((version (1 0)) (valid true)))
(error
  ((kind Report_error)
  (main
    ((msg
       ((Open_box (HV 0)) (Text "This application of the functor ")
       (Open_tag Inline_code) (Text "F") Close_tag (Text " is ill-typed.")
       (Simple_break (1 0)) (Text "These arguments:") (Simple_break (1 2))
       (Open_box (B 0)) (Open_tag Preservation) (Text "$S2") Close_tag
       (Simple_break (1 0)) (Open_tag Preservation) (Text "()") Close_tag
       Close_box (Simple_break (1 0)) (Text "do not match these parameters:")
       (Simple_break (1 2)) (Open_box (B 0)) (Text "functor")
       (Simple_break (1 0)) (Open_tag Insertion) (Text "(") (Text "X")
       (Text " : ") (Text "$T1") (Text ")") Close_tag (Simple_break (1 0))
       (Open_tag Preservation) (Text "(") (Text "Y") (Text " : ")
       (Text "...") (Text ")") Close_tag (Simple_break (1 0))
       (Open_tag Preservation) (Text "()") Close_tag (Simple_break (1 0))
       (Text "-> ...") Close_box Close_box))
    (loc ((file "err.ml") (start_line 2) (stop_line 2) (characters (11 33))))))
  (sub
    (((msg
        ((Tab_break (0 0)) Open_tbox (Open_tag Insertion) (Text "1")
        (Text ". ") Close_tag Set_tab (Open_box (HV 2))
        (Text "An argument appears to be missing with module type")
        (Simple_break (1 2)) (Open_box (B 0)) (Text "$T1")
        (Simple_break (1 0)) (Text "=") (Simple_break (1 0)) (Open_box (B 2))
        (Open_box (HV 2)) (Text "sig") (Simple_break (1 0)) (Open_box (B 2))
        (Open_box (HV 2)) (Text "type") (Text " ") (Text "x") Close_box
        Close_box (Simple_break (1 -2)) (Text "end") Close_box Close_box
        Close_box Close_box Close_tbox))
     (loc ((file "_none_"))))
    ((msg
       ((Tab_break (0 0)) Open_tbox (Open_tag Preservation) (Text "2")
       (Text ". ") Close_tag Set_tab (Open_box (HV 2)) (Text "Module ")
       (Text "$S2") (Text " matches the expected module type") Close_box
       Close_tbox))
    (loc ((file "_none_"))))
    ((msg
       ((Tab_break (0 0)) Open_tbox (Open_tag Preservation) (Text "3")
       (Text ". ") Close_tag Set_tab (Open_box (HV 2)) (Text "Module ")
       (Text "()") (Text " matches the expected module type") Close_box
       Close_tbox))
    (loc ((file "_none_"))))))
  (quotable_locs
    (((file "err.ml") (start_line 2) (stop_line 2) (characters (11 33))))))))
```


## Requirements

Discussing with dune developers, the OCaml maintainers, and examining existing
compilers hook, I have identified five requirement:

1. Unobstrusive 
2. Versioned with a specified update policy
3. Independent of specific formats
4. Partially redirectable to specific files
5. Easily derivable specification for the produced file format


## Structured diagnostics as incremental algebraic data types

In order to provide loggers for structured diagnostics that are both independent
of file formats, and can can output directly on a terminal this RFC proposes to
describe structured diagnostics using an EDSL for building simple algebraic
datatype definition.

```ocaml
type _ extension = ..
type 'a typ =
  | Unit: unit typ
  | Bool: bool typ
  | Int: int typ
  | String: string typ
  | List: 'a typ -> 'a list typ
  | Pair: 'a typ * 'b typ -> ('a * 'b) typ
  | Triple: 'a typ * 'b typ * 'c typ -> ('a * 'b * 'c) typ
  | Quadruple: 'a typ * 'b typ * 'c typ * 'd typ ->
      ('a * 'b * 'c * 'd) typ
  | Sum: 'a def -> 'a sum typ
  | Record: 'id def -> 'id record typ
  | Custom: { id :'b extension; pull: ('b -> 'a); default: 'a typ} ->
      'b typ
```

A structured diagnostic can them be seen as a simple ADT described by the EDSL.
The `Custom` constructor and its extension `id` makes it easy to both register
specialized printers for some fields. For instance, defining the `msg` type
for error report can be done with
```
module Msg = New_record(Vl)(struct let name="error_msg" let update=v1 end)()
let msg = Msg.new_field v1 "msg" doc
let msg_loc = Msg.new_field v1 "loc" Loc.ctyp
let () = Msg.seal v1
let msg_typ =
  let pull m = Log.Record.(make [ msg ^= m.txt; msg_loc ^= m.loc ]) in
  Custom { id = Msg; pull; default = Record Msg.scheme }

```
and logging an error report on the compiler logger can be done with
```ocaml
let log_report log report = log.Log.%[Error_log.key] <- report
```

Moreover, with this kind of description, the printers for specific formats can be written independently
using only compiler-libs. Similarly, we can ensure that the description is detailled enough 
to generate printers, serializers or schemas.


## Versioning and covariance for maintainability

An important requirement for structured compiler output is the ability to guarantee a
sufficient level of backward compatibility to other developer tools.

A basic measure to ensure this is to have versioned schemes and diagnostics.
This is can be done easily by adding a metadata field to toplevel compiler diagnostics
which describes the version of the logger used to output the diagnostics.

```json
  "metadata" : { "version" : [1, 0], "valid" : true},
```

However, since we are outputting the structured diagnostics incrementally, we cannot guarantee
at build time that all constructed diagnostics will satisfy the expected schema. Nevertheless,
we can check at runtime when flushing the diagnostics if it is valid according to the
scheme for this version.

Adding a version to the type descriptions also makes it easier to enforce a update policy
between version. The policy proposed in this PR is that

- minor updates are covariant
- major update only delete deprecated fields
- logger should support as much as possible the previous major version

In other, a minor update may:

- add new fields from a record type
- remove constructors from a sum type
- deprecate a field from a record type
- promotes an optional field from a record type to a required field

while a major update may:

- remove a deprecated field from a record type
- add a new constructor to a sum type

With this policy, we can be sure that a logger at version `major.minor` can output
a structured diagnostic which is valid for any version `major.m` where `m<minor`.
Indeed this only requires to not output fields from future version.

Enforcing compatibility between major versions is more delicate. In the record
case, the compiler code will need to keep `deprecated` path to output the now
deleted fields. However, there are no good options from constructor.

If there is a new constructor in version `major.0`, it is not always possible
to translate this new constructor to the diagnostic in the previous version.
We could replace future constructor by a special constructor `<future>`, but it is
unclear if this would help tools more than having an unknown constructor name.

This means that developer tools will



## A document type for structured text

Looking more closely to this fragment of error message in a terminal

```ocaml
       2. Modules do not match: P.B : b is not included in b/2
```

it has been formatted by the compiler taking in account that the terminal width is
`80` and it supports ANSI escape. But when a tool reprints this error message,
we are probably not printing to the same terminal.

Thus, the compiler should let formatting decisions to downstream tools. But how
do we update the compiler text format in a lightweight way?
We shall present how the OCaml compiler has done so by using an
alternative \mintinline{ocaml}{Format} implementation
\footnote{\url{https://github.com/ocaml/ocaml/13169}}.
