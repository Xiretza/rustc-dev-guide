# Diagnostic and subdiagnostic structs
rustc has two diagnostic derives that can be used to create simple diagnostics,
which are recommended to be used when they are applicable:
`#[derive(SessionDiagnostic)]` and `#[derive(SessionSubdiagnostic)]`.

Diagnostics created with the derive macros can be translated into different
languages and each has a slug that uniquely identifies the diagnostic.

## `#[derive(SessionDiagnostic)]`
Instead of using the `DiagnosticBuilder` API to create and emit diagnostics,
the `SessionDiagnostic` derive can be used. `#[derive(SessionDiagnostic)]` is
only applicable for simple diagnostics that don't require much logic in
deciding whether or not to add additional subdiagnostics.

Consider the [definition][defn] of the "field already declared" diagnostic
shown below:

```rust,ignore
#[derive(SessionDiagnostic)]
#[error(typeck::field_already_declared, code = "E0124")]
pub struct FieldAlreadyDeclared {
    pub field_name: Ident,
    #[primary_span]
    #[label]
    pub span: Span,
    #[label(typeck::previous_decl_label)]
    pub prev_span: Span,
}
```

`SessionDiagnostic` can only be applied to structs. Every `SessionDiagnostic`
has to have one attribute applied to the struct itself: either `#[error(..)]`
for defining errors, or `#[warning(..)]` for defining warnings.

If an error has an error code (e.g. "E0624"), then that can be specified using
the `code` sub-attribute. Specifying a `code` isn't mandatory, but if you are
porting a diagnostic that uses `DiagnosticBuilder` to use `SessionDiagnostic`
then you should keep the code if there was one.

Both `#[error(..)]` and `#[warning(..)]` must provide a slug as the first
positional argument (a path to an item in `rustc_errors::fluent::*`). A slug
uniquely identifies the diagnostic and is also how the compiler knows what
error message to emit (in the default locale of the compiler, or in the locale
requested by the user). See [translation documentation](./translation.md) to
learn more about how translatable error messages are written and how slug
items are generated.

In our example, the Fluent message for the "field already declared" diagnostic
looks like this:

```fluent
typeck_field_already_declared =
    field `{$field_name}` is already declared
    .label = field already declared
    .previous_decl_label = `{$field_name}` first declared here
```

`typeck_field_already_declared` is the slug from our example and is followed
by the diagnostic message.

Every field of the `SessionDiagnostic` which does not have an annotation is
available in Fluent messages as a variable, like `field_name` in the example
above. Fields can be annotated `#[skip_arg]` if this is undesired.

Using the `#[primary_span]` attribute on a field (that has type `Span`)
indicates the primary span of the diagnostic which will have the main message
of the diagnostic.

Diagnostics are more than just their primary message, they often include
labels, notes, help messages and suggestions, all of which can also be
specified on a `SessionDiagnostic`.

`#[label]`, `#[help]` and `#[note]` can all be applied to fields which have the
type `Span`. Applying any of these attributes will create the corresponding
subdiagnostic with that `Span`. These attributes will look for their
diagnostic message in a Fluent attribute attached to the primary Fluent
message. In our example, `#[label]` will look for
`typeck_field_already_declared.label` (which has the message "field already
declared"). If there is more than one subdiagnostic of the same type, then
these attributes can also take a value that is the attribute name to look for
(e.g. `previous_decl_label` in our example).

Other types have special behavior when used in a `SessionDiagnostic` derive:

- Any attribute applied to an `Option<T>` and will only emit a
  subdiagnostic if the option is `Some(..)`.
- Any attribute applied to a `Vec<T>` will be repeated for each element of the
  vector.

`#[help]` and `#[note]` can also be applied to the struct itself, in which case
they work exactly like when applied to fields except the subdiagnostic won't
have a `Span`. These attributes can also be applied to fields of type `()` for
the same effect, which when combined with the `Option` type can be used to
represent optional `#[note]`/`#[help]` subdiagnostics.

Suggestions can be emitted using one of four field attributes:

- `#[suggestion(slug, code = "...", applicability = "...")]`
- `#[suggestion_hidden(slug, code = "...", applicability = "...")]`
- `#[suggestion_short(slug, code = "...", applicability = "...")]`
- `#[suggestion_verbose(slug, code = "...", applicability = "...")]`

Suggestions must be applied on either a `Span` field or a `(Span,
MachineApplicability)` field. Similarly to other field attributes, the slug
specifies the Fluent attribute with the message and defaults to the equivalent
of `.suggestion`. `code` specifies the code that should be suggested as a
replacement and is a format string (e.g. `{field_name}` would be replaced by
the value of the `field_name` field of the struct), not a Fluent identifier.
`applicability` can be used to specify the applicability in the attribute, it
cannot be used when the field's type contains an `Applicability`.

In the end, the `SessionDiagnostic` derive will generate an implementation of
`SessionDiagnostic` that looks like the following:

```rust,ignore
impl SessionDiagnostic<'_> for FieldAlreadyDeclared {
    fn into_diagnostic(self, sess: &'_ rustc_session::Session) -> DiagnosticBuilder<'_> {
        let mut diag = sess.struct_err(rustc_errors::fluent::typeck::field_already_declared);
        diag.set_span(self.span);
        diag.span_label(
            self.span,
            rustc_errors::fluent::typeck::label
        );
        diag.span_label(
            self.prev_span,
            rustc_errors::fluent::typeck::previous_decl_label
        );
        diag
    }
}
```

Now that we've defined our diagnostic, how do we [use it][use]? It's quite
straightforward, just create an instance of the struct and pass it to
`emit_err` (or `emit_warning`):

```rust,ignore
tcx.sess.emit_err(FieldAlreadyDeclared {
    field_name: f.ident,
    span: f.span,
    prev_span,
});
```

### Reference
`#[derive(SessionDiagnostic)]` and `#[derive(LintDiagnostic)]` support the
following attributes:

- `#[error(slug, code = "...")]`, `#[warning(slug, code = "...")]`,
  `#[fatal(slug, code = "...")]` or `#[lint(slug, code = "...")]`
  - _Applied to struct._
  - _Mandatory_
  - Defines the struct to be representing an error, fatal error, a warning or a
    lint. Errors, fatal errors and warnings only supported by
    `SessionDiagnostic`, and lints by `LintDiagnostic`.
  - Slug (_Mandatory_)
    - Uniquely identifies the diagnostic and corresponds to its Fluent message,
      mandatory.
    - A path to an item in `rustc_errors::fluent`. Always in a module starting
      with a Fluent resource name (which is typically the name of the crate
      that the diagnostic is from), e.g.
      `rustc_errors::fluent::typeck::field_already_declared`
      (`rustc_errors::fluent` is implicit in the attribute, so just
      `typeck::field_already_declared`).
    - See [translation documentation](./translation.md).
  - `code = "..."` (_Optional_)
    - Specifies the error code.
- `#[note]` or `#[note(slug)]` (_Optional_)
  - _Applied to struct or `Span`/`()` fields._
  - Adds a note subdiagnostic.
  - Value is a path to an item in `rustc_errors::fluent` for the note's
    message.
    - Defaults to equivalent of `.note`.
  - If applied to a `Span` field, creates a spanned note.
- `#[help]` or `#[help(slug)]` (_Optional_)
  - _Applied to struct or `Span`/`()` fields._
  - Adds a help subdiagnostic.
  - Value is a path to an item in `rustc_errors::fluent` for the note's
    message.
    - Defaults to equivalent of `.help`.
  - If applied to a `Span` field, creates a spanned help.
- `#[label]` or `#[label(slug)]` (_Optional_)
  - _Applied to `Span` fields._
  - Adds a label subdiagnostic.
  - Value is a path to an item in `rustc_errors::fluent` for the note's
    message.
    - Defaults to equivalent of `.label`.
- `#[warn_]` or `#[warn_(slug)]` (_Optional_)
  - _Applied to `Span` fields._
  - Adds a warning subdiagnostic.
  - Value is a path to an item in `rustc_errors::fluent` for the note's
    message.
    - Defaults to equivalent of `.warn`.
- `#[suggestion{,_hidden,_short,_verbose}(slug, code = "...", applicability = "...")]`
  (_Optional_)
  - _Applied to `(Span, MachineApplicability)` or `Span` fields._
  - Adds a suggestion subdiagnostic.
  - Slug (_Mandatory_)
    - A path to an item in `rustc_errors::fluent`. Always in a module starting
      with a Fluent resource name (which is typically the name of the crate
      that the diagnostic is from), e.g.
      `rustc_errors::fluent::typeck::field_already_declared`
      (`rustc_errors::fluent` is implicit in the attribute, so just
      `typeck::field_already_declared`). Fluent attributes for all messages
      exist as top-level items in that module (so `typeck_message.attr` is just
      `typeck::attr`).
    - See [translation documentation](./translation.md).
    - Defaults to `rustc_errors::fluent::_subdiag::suggestion` (or
    - `.suggestion` in Fluent).
  - `code = "..."` (_Mandatory_)
    - Value is a format string indicating the code to be suggested as a
      replacement.
  - `applicability = "..."` (_Optional_)
    - String which must be one of `machine-applicable`, `maybe-incorrect`,
      `has-placeholders` or `unspecified`.
- `#[subdiagnostic]`
  - _Applied to a type that implements `AddToDiagnostic` (from
    `#[derive(SessionSubdiagnostic)]`)._
  - Adds the subdiagnostic represented by the subdiagnostic struct.
- `#[primary_span]` (_Optional_)
  - _Applied to `Span` fields._
  - Indicates the primary span of the diagnostic.
- `#[skip_arg]` (_Optional_)
  - _Applied to any field._
  - Prevents the field from being provided as a diagnostic argument.

## `#[derive(SessionSubdiagnostic)]`
It is common in the compiler to write a function that conditionally adds a
specific subdiagnostic to an error if it is applicable. Oftentimes these
subdiagnostics could be represented using a diagnostic struct even if the
overall diagnostic could not. In this circumstance, the `SessionSubdiagnostic`
derive can be used to represent a partial diagnostic (e.g a note, label, help or
suggestion) as a struct.

Consider the [definition][subdiag_defn] of the "expected return type" label
shown below:

```rust
#[derive(SessionSubdiagnostic)]
pub enum ExpectedReturnTypeLabel<'tcx> {
    #[label(typeck::expected_default_return_type)]
    Unit {
        #[primary_span]
        span: Span,
    },
    #[label(typeck::expected_return_type)]
    Other {
        #[primary_span]
        span: Span,
        expected: Ty<'tcx>,
    },
}
```

Unlike `SessionDiagnostic`, `SessionSubdiagnostic` can be applied to structs or
enums. Attributes that are placed on the type for structs are placed on each
variants for enums (or vice versa). Each `SessionSubdiagnostic` should have one
attribute applied to the struct or each variant, one of:

- `#[label(..)]` for defining a label
- `#[note(..)]` for defining a note
- `#[help(..)]` for defining a help
- `#[suggestion{,_hidden,_short,_verbose}(..)]` for defining a suggestion

All of the above must provide a slug as the first positional argument (a path
to an item in `rustc_errors::fluent::*`). A slug uniquely identifies the
diagnostic and is also how the compiler knows what error message to emit (in
the default locale of the compiler, or in the locale requested by the user).
See [translation documentation](./translation.md) to learn more about how
translatable error messages are written and how slug items are generated.

In our example, the Fluent message for the "expected return type" label
looks like this:

```fluent
typeck_expected_default_return_type = expected `()` because of default return type

typeck_expected_return_type = expected `{$expected}` because of return type
```

Using the `#[primary_span]` attribute on a field (with type `Span`) will denote
the primary span of the subdiagnostic. A primary span is only necessary for a
label or suggestion, which can not be spanless.

Every field of the type/variant which does not have an annotation is available
in Fluent messages as a variable. Fields can be annotated `#[skip_arg]` if this
is undesired.

Like `SessionDiagnostic`, `SessionSubdiagnostic` supports `Option<T>` and
`Vec<T>` fields.

Suggestions can be emitted using one of four attributes on the type/variant:

- `#[suggestion(..., code = "...", applicability = "...")]`
- `#[suggestion_hidden(..., code = "...", applicability = "...")]`
- `#[suggestion_short(..., code = "...", applicability = "...")]`
- `#[suggestion_verbose(..., code = "...", applicability = "...")]`

Suggestions require `#[primary_span]` be set on a field and can have the
following sub-attributes:

- The first positional argument specifies the path to a item in
  `rustc_errors::fluent` corresponding to the Fluent attribute with the message
  and defaults to the equivalent of `.suggestion`.
- `code` specifies the code that should be suggested as a replacement and is a
  format string (e.g. `{field_name}` would be replaced by the value of the
  `field_name` field of the struct), not a Fluent identifier.
- `applicability` can be used to specify the applicability in the attribute, it
  cannot be used when the field's type contains an `Applicability`.

Applicabilities can also be specified as a field (of type `Applicability`)
using the `#[applicability]` attribute.

In the end, the `SessionSubdiagnostic` derive will generate an implementation
of `SessionSubdiagnostic` that looks like the following:

```rust
impl<'tcx> AddToDiagnostic for ExpectedReturnTypeLabel<'tcx> {
    fn add_to_diagnostic(self, diag: &mut rustc_errors::Diagnostic) {
        use rustc_errors::{Applicability, IntoDiagnosticArg};
        match self {
            ExpectedReturnTypeLabel::Unit { span } => {
                diag.span_label(span, rustc_errors::fluent::typeck::expected_default_return_type)
            }
            ExpectedReturnTypeLabel::Other { span, expected } => {
                diag.set_arg("expected", expected);
                diag.span_label(span, rustc_errors::fluent::typeck::expected_return_type)
            }

        }
    }
}
```

Once defined, a subdiagnostic can be used by passing it to the `subdiagnostic`
function ([example][subdiag_use_1] and [example][subdiag_use_2]) on a
diagnostic or by assigning it to a `#[subdiagnostic]`-annotated field of a
diagnostic struct.

### Reference
`#[derive(SessionSubdiagnostic)]` supports the following attributes:

- `#[label(slug)]`, `#[help(slug)]` or `#[note(slug)]`
  - _Applied to struct or enum variant. Mutually exclusive with struct/enum variant attributes._
  - _Mandatory_
  - Defines the type to be representing a label, help or note.
  - Slug (_Mandatory_)
    - Uniquely identifies the diagnostic and corresponds to its Fluent message,
      mandatory.
    - A path to an item in `rustc_errors::fluent`. Always in a module starting
      with a Fluent resource name (which is typically the name of the crate
      that the diagnostic is from), e.g.
      `rustc_errors::fluent::typeck::field_already_declared`
      (`rustc_errors::fluent` is implicit in the attribute, so just
      `typeck::field_already_declared`).
    - See [translation documentation](./translation.md).
- `#[suggestion{,_hidden,_short,_verbose}(slug, code = "...", applicability = "...")]`
  - _Applied to struct or enum variant. Mutually exclusive with struct/enum variant attributes._
  - _Mandatory_
  - Defines the type to be representing a suggestion.
  - Slug (_Mandatory_)
    - A path to an item in `rustc_errors::fluent`. Always in a module starting
      with a Fluent resource name (which is typically the name of the crate
      that the diagnostic is from), e.g.
      `rustc_errors::fluent::typeck::field_already_declared`
      (`rustc_errors::fluent` is implicit in the attribute, so just
      `typeck::field_already_declared`). Fluent attributes for all messages
      exist as top-level items in that module (so `typeck_message.attr` is just
      `typeck::attr`).
    - See [translation documentation](./translation.md).
    - Defaults to `rustc_errors::fluent::_subdiag::suggestion` (or
    - `.suggestion` in Fluent).
  - `code = "..."` (_Mandatory_)
    - Value is a format string indicating the code to be suggested as a
      replacement.
  - `applicability = "..."` (_Optional_)
    - _Mutually exclusive with `#[applicability]` on a field._
    - Value is the applicability of the suggestion.
    - String which must be one of:
      - `machine-applicable`
      - `maybe-incorrect`
      - `has-placeholders`
      - `unspecified`
- `#[primary_span]` (_Mandatory_ for labels and suggestions; _optional_ otherwise)
  - _Applied to `Span` fields._
  - Indicates the primary span of the subdiagnostic.
- `#[applicability]` (_Optional_; only applicable to suggestions)
  - _Applied to `Applicability` fields._
  - Indicates the applicability of the suggestion.
- `#[skip_arg]` (_Optional_)
  - _Applied to any field._
  - Prevents the field from being provided as a diagnostic argument.

[defn]: https://github.com/rust-lang/rust/blob/bbe9d27b8ff36da56638aa43d6d0cdfdf89a4e57/compiler/rustc_typeck/src/errors.rs#L65-L74
[use]: https://github.com/rust-lang/rust/blob/eb82facb1626166188d49599a3313fc95201f556/compiler/rustc_typeck/src/collect.rs#L981-L985

[subdiag_defn]: https://github.com/rust-lang/rust/blob/e70c60d34b9783a2fd3171d88d248c2e0ec8ecdd/compiler/rustc_typeck/src/errors.rs#L220-L233
[subdiag_use_1]: https://github.com/rust-lang/rust/blob/e70c60d34b9783a2fd3171d88d248c2e0ec8ecdd/compiler/rustc_typeck/src/check/fn_ctxt/suggestions.rs#L556-L560
[subdiag_use_2]: https://github.com/rust-lang/rust/blob/e70c60d34b9783a2fd3171d88d248c2e0ec8ecdd/compiler/rustc_typeck/src/check/fn_ctxt/suggestions.rs#L575-L579
