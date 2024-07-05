# Structured diagnostics

A limitation of the current compiler diagnostics (error messages, warnings, time
profile information and other compiler developer debugging output) is that it
cannot be parsed easily and reliably by developer tools.

This means that not only those tools have the choice to either parse the
diagnostics produced by the compiler, which is brittle; or lose meaningful
information.

Consider for instance, a moderately large OCaml error message:
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
one explicit location, one main message
```ocaml
This application of the functor F is ill-typed.
       These arguments:
         $S2 ()
       do not match these parameters:
         functor (X : $T1) (Y : ...) () -> ...
```

and three submessages:
```ocaml
       1. An argument appears to be missing with module type
              $T1 = sig type x end
```

```
       2. Module $S2 matches the expected module type
```

```
       3. Module () matches the expected module type
```
decorated by four distinct semantic tags.

All this information is lost when printed as a plain text. Even worse for tools,
the compiler is using `Format` to shape the output to adjust it to a 78-column
width and semantic tags to control the styling of the text through ANSI terminal
escape sequences. All this formatting is counterproductive for developer tools
that would rather render the error message in another text shape, with another
styling context.

The aim of this RFCs is to describe a way to add structured diagnostics to
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

1. Unobstrusive and short-circuitable

   We don't want to remove the current direct output that prints directly to the
   terminal. The construction of the structured diagnostic should be quite
   similar to the existing printing mechanism.
   
2. Versioned with a specified update policy

  If we want the compiler diagnostic to be usable without friction by non-internal tools, we
  should write down which kind of stability guarantee we offer, and have a clear update paths
  for future changes. For instance, one possibility to keep in mind would be the
  [SARIF](https://sarifweb.azurewebsites.net) specifications.

3. Independent of specific formats

  Some part of the OCaml ecosystem prefer S-expressions to JSON. Both format are
  quite similar when seen as text-based serialization format for algebraic data
  types.

4. Redirectable path to specific files

  The compiler already provides `-dump-into-files` and `-dump-dir` to output
  debugging output to files. The compiler library makes it possible to redirect
  warnings independently of errors.
  
5. Easily derivable specification for the file format

  One way to make it easier for external tools to use compiler diagnostics is to
  ensure that we can easily derive specifications for the diagnostics.

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
  | Custom: {
      id :'b extension;
      pull: (Version.t -> 'b -> 'a);
      default: 'a typ} -> 'b typ
```

Fresh record or sum type can be then generated by generative functors,
and populated after the definition to allow recursive definitions.


A structured diagnostic can them be seen as a simple ADT described by the EDSL.

The `Custom` constructor and its extension `id` makes it possible to both register
specialized printers for some fields. For instance, defining the `msg` type
for error report can be done with
```
module Msg = New_record(Vl)(struct let name="error_msg" let update=v1 end)()
let msg = Msg.new_field v1 "msg" doc
let msg_loc = Msg.new_field v1 "loc" Loc.ctyp
let () = Msg.seal v1
let msg_typ =
  let pull v m = Log.Record.(make v [ msg ^= m.txt; msg_loc ^= m.loc ]) in
  Custom { id = Msg; pull; default = Msg.raw_type }
```
and logging an error report on the compiler logger can be done with

```ocaml
let log_report log report = Log.set log Error_log.key report
```
or
```ocaml
let log_report log report = log.Log.%[Error_log.key] <- report
```

Moreover, with this kind of description, the printers for specific formats can be written independently
using only compiler-libs. Similarly, we can ensure that the description is detailled enough 
to generate printers, serializers or schema.


## Versioning and subtyping rules for maintainability

An important requirement for structured compiler output is the ability to guarantee a
sufficient level of backward compatibility to other developer tools.

A basic measure to ensure this is to have versioned schemata and diagnostics.
An easy first is to add a metadata field to toplevel compiler diagnostics
which describes the version of the logger used to output the diagnostics:

```json
  "metadata" : { "version" : [1, 0], "valid" : "Full"},
```

However, since we are constructing the structured diagnostics incrementally, we
cannot guarantee at build time that all constructed diagnostics will satisfy the
expected schema. Nevertheless, we can check at runtime when writing the
diagnostics if it is valid according to the scheme for this version.

Adding a version to the type descriptions also makes it easier to enforce a
update policy between version. The policy that I propose is that

- minor updates only only refine schemata into a subtype of the previous schema
- major update may delete deprecated fields or add new constructors
- logger should support as much as possible the previous major version
- only one update of diagnostic versions by compiler minor compiler version.

In other, a minor update may:

0. create new record or sum types
1. add new fields from a record type
2. deprecate a field from a record type
3. promotes an optional field from a record type to a required field
4. remove constructors from a sum type
5. expand a non-record argument type of a constructor to a record type
  whose field "contents" contains the previous value

while a major update may:

0. delete unused records and sum types
1. remove a deprecated field from a record type
2. add a new constructor to a sum type

With this policy, we can be sure that a logger at version `major.minor` can output
a structured diagnostic which is valid for any version `major.m` where `m<=minor`.
Indeed this only requires to:

1. erase fields from future version
2. project expanded constructor arguments to their previous core contents.

Enforcing compatibility between major versions is more delicate.

First, to avoid reuse of previous name, we must keep a set of tombstone for
deleted nominal types, record fields, and sum constructors.

Second, in the record case, the compiler code will need to keep `deprecated` path to
output the now deleted fields. This means that a record field may progress from
versions to versions between:

1. optional field
2. mandatory field
3. deprecated field
4. freshly deleted field
5. ancient deleted field

Note that this policy already requires than any new diagnostic field is supported
by three OCaml minor versions (and thus one year and a half).

Finally, there are no good options from constructor.

If there is a new constructor in version `major.0`, it is not always possible
to translate this new constructor to the diagnostic in the previous version.
We could replace future constructor by a special constructor `<future>`, but it is
unclear if this would help tools more than having an unknown constructor name.

Another option would be to require that new constructor can be approximated by previous constructor.

This means that developer tools will have to consider that any sum type are
extensible.

This is the reason why minor updates are allowed to expand the type of constructor
arguments. Without this subtyping rule, amending a constructor argument which
was not a record would require a major version change. With this rule, we have
at least some flexibility for constructor arguments.

For instance, one of the sum type presents in compiler diagnostics is the error kind,
which is represented as:

```ocaml
type report_kind =
  | Report_error
  | Report_warning of string
  | Report_warning_as_error of string
  | Report_alert of string
  | Report_alert_as_error of string
```

It seems possible in the future that we may want to expand the `Report_warning`
argument to contain the warning number, name and error status while deleting the
`Report_warning_as_error` constructor. With the deletion and expansion rule, this
could be done in a minor update.
