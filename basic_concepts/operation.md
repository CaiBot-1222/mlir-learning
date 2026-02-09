# Operation thoughts

> “An MLIR operation is: optional results, then the operation itself (in either generic or custom syntax), then an optional location.”
```mlir
operation             ::= op-result-list? (generic-operation | custom-operation)
                          trailing-location?
generic-operation     ::= string-literal `(` value-use-list? `)`  successor-list?
                          dictionary-properties? region-list? dictionary-attribute?
                          `:` function-type
custom-operation      ::= bare-id custom-operation-format
op-result-list        ::= op-result (`,` op-result)* `=`
op-result             ::= value-id (`:` integer-literal)?
successor-list        ::= `[` successor (`,` successor)* `]`
successor             ::= caret-id (`:` block-arg-list)?
dictionary-properties ::= `<` dictionary-attribute `>`
region-list           ::= `(` region (`,` region)* `)`
dictionary-attribute  ::= `{` (attribute-entry (`,` attribute-entry)*)? `}`
trailing-location     ::= `loc` `(` location `)`
```

## 1. Top Level shape

```text
operation ::= op-result-list? (generic-operation | custom-operation)
              trailing-location?
```

should look like :
```mlir
%res0, %res1:2 = "mydialect.my_op"(%arg0, %arg1) ... : (i32, f32) -> (i1, f32) loc("file.mlir":12:3)

%res = arith.addi %a, %b : i32
```
## 2. Results
```
op-result-list ::= op-result (`,` op-result)* `=`
op-result      ::= value-id (`:` integer-literal)?
```
example:
```mlir
%r = some.op ...        // single result, named %r

%r0, %r1 = some.op ...  // two separate result names (%r0 and %r1)

%r:3 = some.op ...      // three results: %r#0, %r#1, %r#2
```

## 3. Generic vs Custom operations
### 3.1 **Generic form**
```
generic-operation ::= string-literal `(` value-use-list? `)` successor-list?
                      dictionary-properties? region-list? dictionary-attribute?
                      `:` function-type
```
```mlir
%r:2 = "mydialect.my_op"(%a, %b)
        [^bb1(%x), ^bb2] 
        <{prop1 = 42, prop2 = "foo"}>
        (%region0, %region1)
        {attr1 = 1 : i32, attr2 = "bar"}
        : (i32, f32) -> (f32, i1)
      loc("file.mlir":12:3)
```
**successor-list**
```
successor-list ::= `[` successor (`,` successor)* `]`
successor      ::= caret-id (`:` block-arg-list)?
```
```mlir
^bb0(%x: i32, %y: i1):
  cf.cond_br %y, ^bb1(%x), ^bb2

// for example
cf.cond_br %cond, ^then(%x), ^else
// has the list
[^then(%x), ^else]
```
**dictionary-properties**
```
dictionary-properties ::= `<` dictionary-attribute `>`
dictionary-attribute  ::= `{` (attribute-entry (`,` attribute-entry)*)? `}`
```
```mlir
<{prop1 = 42 : i32, prop2 = "foo"}>
{attr1 = 1 : i32, attr2 = "bar"}
```
**region-list**
```
region-list ::= `(` region (`,` region)* `)`
```
```mlir
"scf.if"(%cond) ({
  // then region
  ...
}, {
  // else region
  ...
}) : (i1) -> ()
```

### 3.2 **Custom form**
```mlir
// Custom form (human-friendly)
%sum = arith.addi %a, %b : i32

// Roughly equivalent generic form
%sum = "arith.addi"(%a, %b) : (i32, i32) -> i32
```

### 3.3 Putting together
```mlir
%r:2 = "mydialect.my_op"(%a, %b)
        [^succ0(%x), ^succ1]
        <{prop1 = 42 : i32}>
        ({  // region 0
           ^bb0(%arg0: i32):
             ...
         },
         {  // region 1
           ^bb1:
             ...
         })
        {attr1 = "hello", attr2 = 1.0 : f64}
        : (i32, i32) -> (f32, i1)
        loc("file.mlir":12:3)
```
