# Structured diagnostics

A limitation of the current compiler diagnostics (error messages, warnings, time
profile information and other compiler developer debugging output) is that it
cannot be parsed easily and reliably by developer tools.

This means that not only those tools have the choice to either parse the
diagnostics produced by the compiler, which is brittle; or lose meaningful
information.

Consider for instance, a moderately large OCaml error message:
```ocaml
File "functor_example.ml", line 7, characters 11-15:
7 | module M = F(Y)
               ^^^^
Error: This application of the functor F is ill-typed.
       These arguments:
         Y
       do not match these parameters:
         functor (X : x) (Y : y) -> ...
       1. An argument appears to be missing with module type x
       2. Module Y matches the expected module type y
```
The internal compiler report data types for this error messages contains
one explicit location, one main message
```ocaml
Error: This application of the functor F is ill-typed.
       These arguments:
         Y
       do not match these parameters:
         functor (X : x) (Y : y) -> ...
```

and two submessages
```ocaml
       1. An argument appears to be missing with module type x
```

```ocaml
       2. Module Y matches the expected module type y
```
decorated by three distinct semantic tags.

All this information is lost when printed as a plain text. Even worse for tools,
the compiler is using `Format` to shape the output to adjust it to a 78-column
width and semantic tags to control the styling of the text through ANSI terminal
escape sequences. All this formatting is counterproductive for developer tools
that would rather render the error message in another text shape, with another
styling context.

The aim of this RFCs is to add structured diagnostics to the compiler as a way
to render the internal compiler ADT in any structured file formats. For
instance, if rendered to s-expressions, the following error message should look
like

```lisp
((metadata ((version (1 0)) (valid Full)))
((kind Report_error)
(main
  ((msg
     ((Open_box (HV 0)) (Text "This application of the functor ")
     (Open_tag Inline_code) (Text "F") Close_tag (Text " is ill-typed.")
     (Simple_break (1 0)) (Text "These arguments:") (Simple_break (1 2))
     (Open_box (B 0)) (Open_tag Preservation) (Text "Y") Close_tag Close_box
     (Simple_break (1 0)) (Text "do not match these parameters:")
     (Simple_break (1 2)) (Open_box (B 0)) (Text "functor")
     (Simple_break (1 0)) (Open_tag Insertion) (Text "(") (Text "X")
     (Text " : ") (Open_box (B 2)) (Text "x") Close_box (Text ")") Close_tag
     (Simple_break (1 0)) (Open_tag Preservation) (Text "(") (Text "Y")
     (Text " : ") (Open_box (B 2)) (Text "y") Close_box (Text ")") Close_tag
     (Simple_break (1 0)) (Text "-> ...") Close_box Close_box))
  ((file "functor_example.ml") (start_line 7) (stop_line 7)
  (characters (11 15)))))
(sub
  (((msg
      ((Tab_break (0 0)) Open_tbox (Open_tag Insertion) (Text "1")
      (Text ". ") Close_tag Set_tab (Open_box (HV 2))
      (Text "An argument appears to be missing with module type")
      (Simple_break (1 2)) (Open_box (B 0)) (Open_box (B 2)) (Text "x")
      Close_box Close_box Close_box Close_tbox)))
  ((msg
     ((Tab_break (0 0)) Open_tbox (Open_tag Preservation) (Text "2")
     (Text ". ") Close_tag Set_tab (Open_box (HV 2)) (Text "Module ")
     (Text "Y") (Text " matches the expected module type") (Text " ")
     (Open_box (B 2)) (Text "y") Close_box Close_box Close_tbox)))))
(quotable_locs
  (((file "functor_example.ml") (start_line 7) (stop_line 7)
   (characters (11 15)))))))
```

Moreover, this kind of output is useful beyond error messages. Currently, the
compiler already has debugging flags for outputting a textual representation of
the parsetree, typedtree, lambda IR, and more.
All these flags could benefits being cleanly separated rather than being mixed on stdout.

Similarly, the output of the REPL could be split by kind of contents, or the `-config`
flag could benefit available in alternate file formats.


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
  quite similar when seen as text-based serialization format for the algebraic data
  types that already exist internally in the compiler.

4. Redirectable path to specific files

  The compiler already provides `-dump-into-files` and `-dump-dir` to output
  debugging output to files. The compiler library makes it possible to redirect
  warnings independently of errors. The structured diagnostic implementation
  should be able to redirect fields.
  
5. Easily derivable specifications

  One way to make it easier for external tools to use compiler diagnostics is to
  ensure that we can easily derive specifications for the diagnostics.

## Versioning and subtyping rules for maintainability

An important requirement for structured compiler output is the ability to guarantee a
sufficient level of backward compatibility to other developer tools.

An easy first step is to add a metadata field to toplevel compiler diagnostics
which describes the version of the logger used to output the diagnostics:

```json
  "metadata" : { "version" : [1, 0] }
```

However, since we are constructing the structured diagnostics incrementally, we
cannot guarantee at build time that all constructed diagnostics will satisfy the
expected schema. 

Thus, I propose to add a runtime check, and a metadata field which confirms that the diagnostic
is well typed

```json
  "metadata" : { "version" : [1, 0], "valid" : "Full"},
```


Beyond a version number, we should enforce a clear update policy for schamata between.
The policy that I propose is a data-oriented version of semantic versioning:

- minor updates can only refine schemata into a subtype of the previous schema
- major update can delete deprecated fields or add new constructors
- only one update of diagnostic versions by compiler minor compiler version.

In other, a minor update may:

0. create new record or sum types

1. add new fields to a record type
2. promote an optional field from a record type to a required field
3. deprecate a field from a record type

4. remove constructors from a sum type
5. expand the argument type of a variant constructor
6. introduce a new derived variant constructor

while a major update may:

0. delete unused records and sum types
1. remove a deprecated field from a record type
2. add a new variant constructor to a variant type

With this policy, we can be sure that a logger at version `major.minor` can output
a structured diagnostic which is valid for any version `major.m` where `m<=minor`.
Indeed this only requires to:

1. erase fields from future versions
2. project constructors to their previous representation

### Record subtyping rule

The subtyping rules for records straigthforward

1. add new fields to a record type
2. promote an optional field to a required field
3. deprecate a field

### Variant constructors subtyping rules

The constructor rules can be divided in two parts,
the dual rule for record field addition

1. remove constructors from a sum type

and the derived constructor rules.

2. expand the argument type of a variant constructor
3. introduce a new derived variant constructor

The first rule is straigthforward and unsufficient. Indeed, this rule only
allows us to remove unused clutter, it is not enough to evolve constructor
fields.

A hackish solution to this issue consists in 

1. duplicate the sum type and update it
2. deprecate the field containing the offending sum type
3. add a new field containing the updated sum type.

To avoid this complex dance, we propose to allo reversible update for variant
constructors: we should be able to expres that a constructor from a version `v2` is in fact
derived from a constructor from an older version `v1<v2`.
There two cases that seem relatively straigthforward to handle:

1. A constructor exists in both `v1` and `v2` with only a change of its argument type,
   with the new type being a subtype of the previous one.

For maximal backward compatibibility, we propose to require that the `v2` argument type
is necessary a record, which contains a `contents` field holding the previous argument.


2. A new constructor in `v2` can be approximated by an older constructor in `v1`

With those possibility, the life cycles of variant constructors is then restricted to

1. derived inception
2. publication
3. expansion
4. deletion

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
`Report_warning_as_error` constructor:


```ocaml
type report_kind =
  | Report_error
  | Report_warning of { contents:string; name:string; number:string; as_error:bool}
  | Report_alert of string
  | Report_alert_as_error of string
```

With the deletion and expansion rule, this change can be done in a minor update.


### Beyond major versions

Enforcing compatibility between major versions is not always possible,
in particular for the variant constructor side.

First, to avoid reuse of previous name with different meaning, we must keep a
set of tombstones for deleted nominal types, record fields, and sum
constructors.

Second, in the record case, the compiler code will need to keep `deprecated` paths to
output the now deleted fields. This means that a record field may progress from
versions to versions between:

1. field publication as option
2. field publication as mandatory
3. field deprecation
4. field deletion

Note that this policy already requires than any new diagnostic field is supported
by at least two OCaml minor versions (and thus one year).

Finally, backward-compatibility for variant constructor is more complex.
If there is a new constructor in version `major.0`, it is not always possible
to translate this new constructor to the diagnostic in the previous version.
We could replace future constructor by a special constructor `<future>`, but it is
unclear if this would help tools more than having an unknown constructor name.
Thus reliable deserializer that expect to work across major version 
would need to parse variant sum as if they were always extensible.

