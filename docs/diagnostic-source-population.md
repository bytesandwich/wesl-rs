# WESL Diagnostic Source Population: Technical Deep-Dive

**Document Type:** Internal Engineering Note  
**Target Audience:** Tooling Engineers, Compiler Contributors  
**Last Updated:** 2025-12-14  
**Status:** Living Document

## Executive Summary

This document explains how source-location information (source, span, module_path, display_name, declaration) is populated inside the WESL compiler pipeline. It is designed to help tooling engineers understand:
- Which errors have spans vs. which do not
- Why certain diagnostics lack location information
- What invariants can be relied upon when building CLI/IDE tools
- How sourcemaps enrich diagnostic metadata

## Core Types

### `Diagnostic<E: std::error::Error>` (`src/error.rs:49-52`)

```rust
pub struct Diagnostic<E: std::error::Error> {
    pub error: Box<E>,
    pub detail: Box<Detail>,
}
```

The `Diagnostic` wrapper enriches any error with contextual metadata stored in `Detail`.

### `Detail` (`src/error.rs:54-62`)

```rust
pub struct Detail {
    pub source: Option<String>,       // Source code contents
    pub output: Option<String>,        // Generated output (even if failed)
    pub module_path: Option<ModulePath>, // Logical path (e.g., package::foo::bar)
    pub display_name: Option<String>,  // Human-readable name (e.g., /path/to/file.wesl)
    pub declaration: Option<String>,   // Declaration name where error occurred
    pub span: Option<Span>,            // Byte-offset range in source
}
```

All fields are optional. Population depends on:
1. The error type
2. Which compiler phase produced the error
3. Whether a SourceMapper is used

---

## Error Categories and Span Availability

### 1. Parse Errors (`wgsl_parse::Error`)

**Files:** `crates/wgsl-parse/src/error.rs`

**Span-bearing:** ✅ Always

Parse errors are generated during lexing/parsing and **always** include a `Span`. The span is populated directly by the parser using token positions.

```rust
// src/error.rs:64-70
impl From<wgsl_parse::Error> for Diagnostic<Error> {
    fn from(error: wgsl_parse::Error) -> Self {
        let span = error.span;
        let mut res = Self::new(Error::ParseError(error));
        res.detail.span = Some(span);
        res
    }
}
```

**Span Guarantee:** Parse errors ALWAYS have a span pointing to the invalid token or location.

**Fields Typically Present:**
- `span`: ✅ Always
- `source`: ⚠️ Only if `with_source()` called
- `module_path`: ⚠️ Only if `with_module_path()` called
- `declaration`: ❌ Never (parse errors are module-level)

**Example Error:**
```
ErrorKind::UnexpectedToken { token: "}", expected: [")", ";"] }
```

---

### 2. Validation Errors (`ValidateError`)

**Files:** `crates/wesl/src/validate/mod.rs`

**Span-bearing:** ⚠️ Partial

Validation performs symbol checking, duplicate detection, and cycle detection. Spans are available **only for expression-based errors**.

#### Span-bearing validation errors:
- `UndefinedSymbol` – when checking expressions (line 163-167)
- `NotCallable` – function call validation (line 163-167)
- `ParamCount` – function call validation (line 163-167)

```rust
// validate/mod.rs:163-167
check_expr(expr, wesl).map_err(|e| {
    let mut err = Diagnostic::from(e);
    err.detail.span = Some(expr.span());
    err.detail.declaration = decl.ident().map(|id| id.name().to_string());
    err
})?;
```

#### Span-less validation errors:
- `Duplicate` – declaration-level, no specific token span
- `Cycle` – graph-level error across multiple declarations

**Why No Span?**
- `Duplicate` errors arise from comparing declaration names, not a single token
- `Cycle` errors represent a **relationship** between declarations, not a single location
- These are architectural: the error describes a property of the declaration graph

**Fields Typically Present:**
- `span`: ⚠️ Only for expression-based errors
- `declaration`: ✅ Usually present (added at line 73-75, 88-89, 165)
- `module_path`: ⚠️ Only when called via `compile_pre_assembly` (line 867-870)
- `source`: ⚠️ Only when passed through sourcemap

**Conceptual Anchoring:**
When a validation error lacks a span, tooling should anchor the error to:
- The declaration name (if present)
- Line 1, column 1 of the module (fallback)

---

### 3. Import/Resolution Errors (`ImportError`, `ResolveError`)

**Files:** `crates/wesl/src/import.rs`, `crates/wesl/src/resolve.rs`

**Span-bearing:** ❌ Never

Import resolution operates at the **module graph level**, not the syntax level. These errors occur when:
- A module file is not found
- An imported declaration doesn't exist in the target module
- Visibility constraints are violated

```rust
// import.rs:29-41
pub enum ImportError {
    DuplicateSymbol(String),
    ResolveError(ResolveError),
    MissingDecl(ModulePath, String),  // No span!
    Private(String, ModulePath),
}
```

**Why No Span?**

1. **`MissingDecl`**: The error describes the **absence** of a declaration in a remote module. There is no single source location that represents "where the missing thing should be."

2. **`ResolveError::FileNotFound`** / **`ModuleNotFound`**: These are filesystem-level errors. There is no source code to point at.

3. **Architectural Reason**: Import resolution happens **after** parsing but **during graph traversal**. The algorithm walks the module dependency graph, and errors arise from graph topology, not from specific tokens.

**Could a span theoretically exist?**

Yes, but it is not currently stored:
- `MissingDecl` could store the span of the `import` statement that attempted to import the missing item
- This would require threading span information through the import resolution algorithm
- Current design prioritizes simplicity: import statements are stripped during assembly, so the span would be from a removed AST node

**Fields Typically Present:**
- `span`: ❌ Never
- `module_path`: ✅ Present for module-level errors
- `display_name`: ⚠️ Only if resolver provides it
- `declaration`: ❌ Rarely (only for `MissingDecl` referring to a specific symbol)
- `source`: ⚠️ Only via sourcemap

**Fallback Strategy:**
Anchor to the module path's file:line 1, column 1.

---

### 4. Conditional Compilation Errors (`CondCompError`)

**Files:** `crates/wesl/src/condcomp.rs`

**Span-bearing:** ⚠️ Partial

Conditional compilation evaluates `@if(expr)` attributes. Errors are span-bearing when they can be traced to a specific attribute expression.

```rust
// condcomp.rs:64-66
fn eval_attr(expr: &ExpressionNode, features: &Features) -> Result<Expression, E> {
    eval_attr_impl(expr, features).map_err(|e| Diagnostic::from(e).with_span(expr.span()).into())
}
```

**Span-bearing:**
- `InvalidExpression` – when expression is invalid
- `InvalidFeatureFlag` – when evaluating a specific flag

**Span-less:**
- `NoPrecedingIf` / `DuplicateIf` – structural validation across multiple attributes

**Fields Typically Present:**
- `span`: ⚠️ Only for expression-based errors (line 66, 204-209)
- `declaration`: ⚠️ When error originates in a function/struct (line 388, 391)
- `module_path`: ⚠️ Propagated if called through compile pipeline
- `source`: ⚠️ Via sourcemap

---

### 5. Evaluation Errors (`EvalError`)

**Files:** `crates/wesl/src/eval/error.rs` (requires `eval` feature)

**Span-bearing:** ⚠️ Partial

Evaluation errors occur during const-expression evaluation or shader execution simulation. They are enriched with context via the `Context` type.

```rust
// eval/mod.rs context tracking
pub struct Context {
    cur_decl: Option<Ident>,     // Currently evaluated declaration
    cur_span: Option<Span>,      // Span within that declaration
    // ...
}
```

Spans are added via `with_ctx()`:

```rust
// error.rs:188-193
#[cfg(feature = "eval")]
pub fn with_ctx(mut self, ctx: &Context) -> Self {
    let (decl, span) = ctx.err_ctx();
    self.detail.declaration = decl.map(|id| id.to_string());
    self.detail.span = span;
    self
}
```

**Span-bearing:**
Most evaluation errors that arise during expression evaluation have a span (traced through `cur_span` in eval context).

**Span-less:**
Errors arising from:
- Module-scope semantic violations (e.g., `OverrideInConst`, `LetInMod`)
- Type mismatches without a specific expression (e.g., return type checking)

**Fields Typically Present:**
- `span`: ⚠️ When expression is being evaluated
- `declaration`: ✅ Usually present via `with_ctx()`
- `module_path`: ⚠️ Via sourcemap
- `source`: ⚠️ Via sourcemap

**Example:**
```rust
// lib.rs:707-708
.map_err(|e| Diagnostic::from(e)
    .with_source(source.to_string())
    .with_ctx(&ctx))
```

---

### 6. Generics Errors (`GenericsError`)

**Files:** `crates/wesl/src/generics/mod.rs` (requires `generics` feature)

**Span-bearing:** ❌ Unknown (experimental, not analyzed)

Generics are experimental. Assume similar patterns to validation errors.

---

## Compilation Pipeline: Diagnostic Enrichment Flow

### Pre-Assembly Phase (`compile_pre_assembly`)

**File:** `src/lib.rs:834-873`

```rust
pub fn compile_pre_assembly(
    root: &ModulePath,
    resolver: &impl Resolver,
    opts: &CompileOptions,
) -> Result<(import::Resolutions, HashSet<Ident>), Error>
```

1. **Conditional compilation** (if enabled):
   - Wraps resolver in `Preprocessor` to run `condcomp::run()`
   - Errors enriched with `with_module_path()` (line 841-843)

2. **Module resolution**:
   - `resolver.resolve_module(root)` may fail with `ResolveError`
   - Parse errors caught and enriched:
     ```rust
     // resolve.rs:39-43
     let wesl: TranslationUnit = source.parse().map_err(|e| {
         Diagnostic::from(e)
             .with_module_path(path.clone(), self.display_name(path))
             .with_source(source.to_string())
     })?;
     ```

3. **Import resolution**:
   - `import::resolve_lazy()` or `import::resolve_eager()`
   - Errors propagated, enriched at call site (line 109-113 in import.rs)

4. **WESL validation** (if enabled):
   - Iterates over all modules
   - Enriches with `with_module_path()` (line 867-870)
   ```rust
   validate_wesl(&module.source).map_err(|d| {
       d.with_module_path(module.path.clone(), resolver.display_name(&module.path))
   })?;
   ```

### Post-Assembly Phase (`compile_post_assembly`)

**File:** `src/lib.rs:876-895`

```rust
fn compile_post_assembly(
    wesl: &mut TranslationUnit,
    options: &CompileOptions,
    keep: &HashSet<Ident>,
) -> Result<(), Error>
```

1. **Generics** (if enabled): Generate variants, replace calls
2. **WGSL validation** (if enabled): `validate_wgsl(wesl)`
3. **Lowering** (if enabled): Transform output
4. **Stripping**: Dead code elimination

**Note:** Post-assembly errors typically have **no module context**, since the code is merged into a single `TranslationUnit`.

### Sourcemap Integration (`compile_sourcemap`)

**File:** `src/lib.rs:920-956`

```rust
pub fn compile_sourcemap(
    root: &ModulePath,
    resolver: &impl Resolver,
    mangler: &impl Mangler,
    options: &CompileOptions,
) -> Result<CompileResult, Error>
```

The sourcemap variant wraps the resolver and mangler in a `SourceMapper`:

```rust
let sourcemapper = SourceMapper::new(root, resolver, mangler);
```

**When does the sourcemap enrich diagnostics?**

1. **Pre-assembly errors** (line 928-955):
   ```rust
   Err(e) => {
       let sourcemap = sourcemapper.finish();
       Err(Diagnostic::from(e)
           .with_sourcemap(&sourcemap)
           .unmangle(Some(&sourcemap), Some(&mangler))
           .into())
   }
   ```

2. **Post-assembly errors** (line 934-941):
   ```rust
   compile_post_assembly(&mut assembly, options, &keep)
       .map_err(|e| {
           Diagnostic::from(e)
               .with_output(assembly.to_string())
               .with_sourcemap(&sourcemap)
               .unmangle(Some(&sourcemap), Some(&mangler))
               .into()
       })
   ```

**What does `with_sourcemap()` do?**

**File:** `src/error.rs:197-221`

```rust
pub fn with_sourcemap(mut self, sourcemap: &impl SourceMap) -> Self {
    // 1. If declaration is set, look up module_path, display_name, and source
    if let Some(decl) = &self.detail.declaration {
        if let Some((path, decl)) = sourcemap.get_decl(decl) {
            self.detail.module_path = Some(path.clone());
            self.detail.declaration = Some(decl.to_string());
            self.detail.display_name = sourcemap
                .get_display_name(path)
                .map(|name| name.to_string());
            self.detail.source = sourcemap
                .get_source(path)
                .map(|s| s.to_string())
                .or(self.detail.source);
        }
    }

    // 2. If source is still missing, use module_path or default
    if self.detail.source.is_none() {
        if let Some(path) = &self.detail.module_path {
            self.detail.source = sourcemap.get_source(path).map(|s| s.to_string());
        } else {
            self.detail.source = sourcemap.get_default_source().map(|s| s.to_string());
        }
    }

    self
}
```

**Sourcemap population order:**
1. **Declaration → Path**: If a mangled declaration name exists, reverse-map to module path and original name
2. **Path → Source**: Look up source code by module path
3. **Fallback**: Use default source (root module) if no path is known

---

## Sourcemap Architecture

### `SourceMap` Trait (`src/sourcemap.rs:17-28`)

```rust
pub trait SourceMap {
    fn get_decl(&self, decl: &str) -> Option<(&ModulePath, &str)>;
    fn get_source(&self, path: &ModulePath) -> Option<&str>;
    fn get_display_name(&self, path: &ModulePath) -> Option<&str>;
    fn get_default_source(&self) -> Option<&str> { None }
}
```

### `SourceMapper` (`src/sourcemap.rs:112-137`)

A proxy that implements both `Resolver` and `Mangler`, recording:
- All loaded modules (path → source)
- All mangled identifiers (mangled → (path, original_name))

```rust
impl Resolver for SourceMapper<'_> {
    fn resolve_source(&'a self, path: &ModulePath) -> Result<Cow<'a, str>, ResolveError> {
        let res = self.resolver.resolve_source(path)?;
        let mut sourcemap = self.sourcemap.borrow_mut();
        sourcemap.add_source(path.clone(), self.resolver.display_name(path), res.clone().into());
        Ok(res)
    }
}

impl Mangler for SourceMapper<'_> {
    fn mangle(&self, path: &ModulePath, item: &str) -> String {
        let res = self.mangler.mangle(path, item);
        let mut sourcemap = self.sourcemap.borrow_mut();
        sourcemap.add_decl(res.clone(), path.clone(), item.to_string());
        res
    }
}
```

**Key Insight:** The sourcemap is built incrementally during compilation by intercepting all module loads and identifier manglings.

---

## Diagnostic Field Guarantees and Invariants

### `span: Option<Span>`

**Guarantee:**
- **Always present** for parse errors
- **Often present** for expression-based validation/eval errors
- **Never present** for import/resolve errors
- **Never present** for post-assembly errors in merged code

**Span Format:**
- Byte-based offsets (not char-based)
- `Span::range()` returns `std::ops::Range<usize>`

**Invariant:**
```
IF span.is_some() THEN source.is_some()
```
A span is meaningless without the source code to anchor it. Tooling must check both.

### `source: Option<String>`

**Guarantee:**
- Present when `with_source()` was called
- Present when `with_sourcemap()` was called AND the sourcemap has the module loaded

**Invariant:**
```
IF span.is_some() THEN source SHOULD be Some
```
(Not enforced, but expected. Diagnostic rendering checks this at runtime – see `error.rs:574-578`)

### `module_path: Option<ModulePath>`

**Guarantee:**
- Present for import/resolve errors
- Present when propagated through `compile_pre_assembly` (for validation errors)
- Absent for post-assembly errors (code is merged, no per-module tracking)

**Format:**
- Logical path: `package::submodule::item`
- OR absolute filesystem path (converted to ModulePath)

### `display_name: Option<String>`

**Guarantee:**
- Present when resolver provides `display_name(path)`
- Typically a filesystem path (e.g., `/home/user/shaders/main.wesl`)

**When is it absolute vs. logical?**
- **Absolute:** `FileResolver` returns filesystem paths
- **Logical:** `VirtualResolver` and `PkgResolver` do not provide display names (return `None`)

**Invariant:**
```
display_name.is_some() IFF resolver.display_name(module_path) is Some
```

### `declaration: Option<String>`

**Guarantee:**
- Present for validation errors inside declarations (line 73-75, 88-89)
- Present for eval errors via `with_ctx()` (line 190)
- Present when set explicitly (e.g., condcomp errors in functions, line 388, 391)

**Mangling:**
- May be mangled (if post-assembly)
- Unmangled by `Diagnostic::unmangle()` (called in `compile_sourcemap`, line 939, 954)

### `output: Option<String>`

**Guarantee:**
- Only populated for post-assembly errors
- Contains the generated WGSL even if compilation failed

**Use case:** Debugging post-assembly errors (validation, stripping, lowering).

---

## Diagnostic Rendering: `annotate-snippets`

**File:** `src/error.rs:547-593`

```rust
impl<E: std::error::Error> Display for Diagnostic<E> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        use annotate_snippets::*;
        let msg = format!("{}", self.error);
        let title = Level::ERROR.primary_title(&msg);
        let mut group = Group::with_title(title);

        let orig = self.display_origin();
        let short_orig = self.display_short_origin();

        if let Some(span) = &self.detail.span {
            let source = self.detail.source.as_deref();

            if let Some(source) = source {
                if span.range().end <= source.len() {
                    let annot = AnnotationKind::Primary.span(span.range()).label(&msg);
                    let mut snip = Snippet::source(source).fold(true).annotation(annot);

                    if let Some(orig) = &short_orig {
                        snip = snip.path(orig);
                    }

                    group = group.element(snip);
                } else {
                    group = group.element(
                        Level::NOTE.message("cannot display snippet: invalid source location"),
                    )
                }
            } else {
                group = group
                    .element(Level::NOTE.message("cannot display snippet: missing source file"))
            }
        }

        let note;
        if let Some(decl) = &self.detail.declaration {
            note = format!("in declaration of `{decl}` in {orig}");
        } else {
            note = format!("in {orig}");
        }
        let group = group.element(Level::NOTE.message(&note));

        let renderer = Renderer::styled();
        let rendered = renderer.render(&[group]);
        write!(f, "{rendered}")
    }
}
```

### What is Rendered?

1. **Error message** (primary title)
2. **Source snippet with span annotation** (if `span` and `source` are present)
3. **Fallback notes** if span/source are missing
4. **Origin note:** `"in declaration of \`foo\` in package::bar (path/to/file.wesl)"`

### What is NOT Rendered?

**No machine-readable output:**
- No `path:line:col` format
- No JSON serialization
- No structured span metadata

**Why?**
The `annotate-snippets` renderer is **human-oriented**. It produces colorized, formatted text for terminal display.

**Implications for IDEs:**
IDEs **cannot parse** this output. They need:
- Structured error objects
- Separate `path`, `line`, `col` fields
- Optional: JSON serialization

---

## Tooling Recommendations

### Safe Fallback Strategies

When `span` is absent:

1. **Check `declaration`**:
   ```rust
   if let Some(decl) = &diagnostic.detail.declaration {
       // Anchor to declaration in module
       // (Requires scanning module source to find declaration)
   }
   ```

2. **Check `module_path` + `display_name`**:
   ```rust
   if let Some(path) = &diagnostic.detail.display_name {
       // Anchor to file:1:1
       report_at(path, 1, 1);
   }
   ```

3. **Fallback to root module**:
   ```rust
   // Use default_source from sourcemap
   // Anchor to line 1, column 1
   ```

**This is consistent with compiler practice:** Many compilers (rustc, clang) report module-level errors without spans.

### What Invariants Can Tooling Rely On?

✅ **Reliable:**
- Parse errors ALWAYS have spans
- `span.is_some()` implies the error is expression-level
- `module_path` is present for pre-assembly errors
- `declaration` is present for errors inside functions/structs

⚠️ **Conditional:**
- `source` is only present if sourcemap is used OR `with_source()` was called
- `display_name` depends on resolver implementation
- Post-assembly errors lack `module_path`

❌ **Not guaranteed:**
- Import errors never have spans
- Validation errors for `Duplicate`/`Cycle` have no spans
- Generics errors (experimental, unspecified)

### Span Indexing: Byte vs. Char

**Format:** Byte offsets into UTF-8 source string.

**Caution:** If tooling uses char-based indexing (e.g., JavaScript, LSP), conversion is required:

```rust
fn byte_offset_to_line_col(source: &str, byte_offset: usize) -> (usize, usize) {
    let mut line = 1;
    let mut col = 1;
    for (i, ch) in source.char_indices() {
        if i == byte_offset {
            return (line, col);
        }
        if ch == '\n' {
            line += 1;
            col = 1;
        } else {
            col += 1;
        }
    }
    (line, col)
}
```

---

## Future Work: Machine-Oriented Diagnostics

### What Would Need to Change?

#### 1. Separate Renderer

Implement a JSON renderer:

```rust
pub fn to_json(&self) -> serde_json::Value {
    json!({
        "message": format!("{}", self.error),
        "severity": "error",
        "span": self.detail.span.map(|s| {
            json!({
                "start": s.range().start,
                "end": s.range().end,
            })
        }),
        "module_path": self.detail.module_path.as_ref().map(|p| p.to_string()),
        "file": self.detail.display_name,
        "declaration": self.detail.declaration,
    })
}
```

#### 2. Expose Span Metadata

Add methods to convert spans to line/column:

```rust
impl Diagnostic<Error> {
    pub fn span_to_line_col(&self) -> Option<(usize, usize)> {
        let span = self.detail.span?;
        let source = self.detail.source.as_ref()?;
        Some(byte_offset_to_line_col(source, span.range().start))
    }
}
```

#### 3. Structured Error Codes

Assign error codes for stable machine parsing:

```rust
pub enum ErrorCode {
    E001,  // Parse error
    E002,  // Undefined symbol
    E003,  // Missing import
    // ...
}
```

---

## Reference: Compiler Phase Capability Matrix

| Phase                  | Parse | Validate | Import | CondComp | Eval | Post-Asm |
|------------------------|-------|----------|--------|----------|------|----------|
| **Span available?**    | ✅    | ⚠️       | ❌     | ⚠️       | ⚠️   | ❌       |
| **Module path?**       | ⚠️    | ✅       | ✅     | ⚠️       | ⚠️   | ❌       |
| **Declaration name?**  | ❌    | ✅       | ⚠️     | ⚠️       | ✅   | ⚠️       |
| **Source via map?**    | ✅    | ✅       | ✅     | ✅       | ✅   | ✅       |

**Legend:**
- ✅ Always available
- ⚠️ Sometimes available
- ❌ Never available

---

## Summary: Key Takeaways

1. **Parse errors are golden:** Always have spans, always have tokens.

2. **Import errors are intentionally span-less:** They describe graph topology, not token locations. This is by design.

3. **Sourcemaps are critical for post-assembly errors:** Without sourcemaps, mangled names and merged code make errors hard to trace.

4. **Validation errors are hybrid:** Expression-based errors have spans; declaration-level errors do not.

5. **Tooling must have fallbacks:** When span is absent, anchor to declaration or module (line 1, col 1). This matches `rustc` and `clang` behavior.

6. **Current rendering is human-only:** `annotate-snippets` is beautiful for terminals, but IDEs need structured JSON.

7. **Byte offsets matter:** LSP and some editors expect char-based indexing. Conversion is required.

---

## Appendix: Code Navigation Map

| Concept              | File                               | Key Lines       |
|----------------------|------------------------------------|-----------------|
| `Diagnostic` struct  | `crates/wesl/src/error.rs`         | 49-62           |
| Parse error span     | `crates/wesl/src/error.rs`         | 64-70           |
| Validation spans     | `crates/wesl/src/validate/mod.rs`  | 163-167         |
| Import errors        | `crates/wesl/src/import.rs`        | 29-41           |
| Sourcemap trait      | `crates/wesl/src/sourcemap.rs`     | 17-28           |
| `with_sourcemap()`   | `crates/wesl/src/error.rs`         | 197-221         |
| Diagnostic rendering | `crates/wesl/src/error.rs`         | 547-593         |
| Compilation pipeline | `crates/wesl/src/lib.rs`           | 834-956         |
| CondComp errors      | `crates/wesl/src/condcomp.rs`      | 8-20, 64-66     |
| Eval context         | `crates/wesl/src/error.rs`         | 188-193         |
| Resolver integration | `crates/wesl/src/resolve.rs`       | 37-44           |

---

**Document maintained by:** WESL Contributors  
**Questions?** Open an issue or discussion on GitHub.
