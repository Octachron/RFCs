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

1. Unobstrusive: the new logger should be easy to use
2. Versioned with a specified update policy
3. Independent of specific format
4. Partially redirectable to specific files
5. Easily derivable specification for the produced file format

## Structured logging as incremental algebraic data type construction

To better cooperate with other tools, the compiler itself should provide
machine-readable structured diagnostics. This idea has been
adopted(\cite{rust},\cite{gcc9},\cite{MSVC},\cite{llvm}) or is under
consideration (\cite{haskell-9.4},\cite{haskell-json}, \cite{golang}) in many
compilers.

However, is the file format that important?
What really matters is that the data is well-structured, in other words
well-typed. And with GADTs, we can create an embedded versioned algebraic type
system for diagnostics:
```ocaml
type 'a typ =
  | Int: int typ | Doc: doc typ | Option: 'a typ -> 'a option typ
  | Sum: 'a def -> 'a sum typ | Record: 'id def -> 'id record typ | ...
```

From that description, we will show how to build various printers for
diagnostics, enforce a versioning policy, provide logging functions that can
replace transparently `Format` printers and more.

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

## Versioning and covariance for maintainability

Another requirement for structured compiler output is the ability to guarantee a
sufficient level of backward compatibility to other developer tools.

We shall show how by weaving a notion of sequence of versions through the
creations and deletions of both record fields and variant constructors, we can
ensure that minor updates are covariant: a minor update may only create record
fields or delete variant constructors. The reverse breaking changes can then be
relegated to major updates.

Similarly, with our versioned eDSL for compiler diagnostics, we can check at runtime
that a diagnostic is well-typed for a given version.
