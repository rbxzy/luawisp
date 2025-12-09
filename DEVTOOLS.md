# DEVTOOLS.md

## LuaWisp Developer Tools & Debugging

This guide covers debugging tools, error handling, and syntax tree inspection available in LuaWisp.

---

## Setup

```javascript
const { LuaWisp, getTotalErrorCount, highlightErrs, printAST, stringify } = require('@codewisp/lua-transpiler');

const compiler = new LuaWisp();

// Register functions with optional argument type checking
compiler.registerFunction("print", -1);  // Variadic, no type checking
compiler.registerFunction("split", 2);   // 2 arguments, no type checking
compiler.registerFunction("random", 2, ["number", "number"]);  // Type-checked arguments

compiler.defineBoilerplate(`
function print(...args) {
    console.log(args.join(" "));
}
function random(min, max) { 
    return Math.random() * (max - min) + min; 
}
`);
```

---

## Compilation Result

The `compile()` method returns a `CompilationResult` object:

```javascript
const result = compiler.compile(sourceCode);
```

### Result Structure

```javascript
{
    success: boolean,      // Whether compilation succeeded
    output: string,        // Final code with boilerplate (legacy)
    raw: string,          // Raw transpiled code WITHOUT boilerplate (success only)
    final: string,        // Final code WITH boilerplate (success only)
    tokens: Token[],      // Lexer tokens
    ast: Stmt[] | null,   // Abstract Syntax Tree
    errors: {
        lexer: Err[],
        parser: Err[],
        transpiler: Err[]
    },
    source: string        // Original source code
}
```

### Basic Usage

```javascript
const sourceCode = `
local x = 10
print(x)
`;

const result = compiler.compile(sourceCode);

if (result.success) {
    console.log("✓ Compilation successful!");
    console.log(result.final);  // Code with boilerplate
    console.log(result.raw);    // Code without boilerplate
} else {
    console.log("✗ Compilation failed!");
}
```

---

## Error Handling

### Counting Errors

```javascript
const { getTotalErrorCount } = require('@codewisp/lua-transpiler');

const totalErrors = getTotalErrorCount(result.errors);
console.log(`Total errors: ${totalErrors}`);
console.log(`Lexer errors: ${result.errors.lexer.length}`);
console.log(`Parser errors: ${result.errors.parser.length}`);
console.log(`Transpiler errors: ${result.errors.transpiler.length}`);
```

### Highlighting Errors

```javascript
const { highlightErrs } = require('@codewisp/lua-transpiler');

if (!result.success) {
    highlightErrs(result.errors, sourceCode);
}
```

**Example Output:**

```
local str = "hello there!
            ^-- Unterminated string literal.

x = unknown_var + 5
    ^-- Variable 'unknown_var' is not defined.

sprite.pointTowards(0, "hello")
                       ^^^^^^^-- Function 'pointTowards' expected 'number'.
```

**Note:** `highlightErrs()` logs directly to the console. For debugging only.

---

## Syntax Tree Inspection

### Printing the AST

```javascript
const { printAST } = require('@codewisp/lua-transpiler');

if (result.success && result.ast) {
    printAST(result.ast);
}
```

**Example Output:**

```json
[
    {
        "type": "VariableStmt",
        "name": {
            "type": "IDENTIFIER",
            "lexeme": "x",
            "literal": null,
            "loc": {
                "line": 1,
                "col": 6,
                "len": 1
            }
        },
        "value": {
            "type": "LiteralExpr",
            "token": {
                "type": "NUMBER",
                "lexeme": "42",
                "literal": 42,
                "loc": {
                    "line": 1,
                    "col": 10,
                    "len": 2
                }
            },
            "loc": {
                "line": 1,
                "col": 10,
                "len": 2
            }
        },
        "isLocal": true,
        "loc": {
            "line": 1,
            "col": 0,
            "len": 12
        }
    },
    {
        "type": "ExpressionStmt",
        "expression": {
            "type": "CallExpr",
            "callee": {
                "type": "VarExpr",
                "name": {
                    "type": "IDENTIFIER",
                    "lexeme": "print",
                    "literal": null,
                    "loc": {
                        "line": 2,
                        "col": 0,
                        "len": 5
                    }
                },
                "loc": {
                    "line": 2,
                    "col": 0,
                    "len": 5
                }
            },
            "args": [
                {
                    "type": "VarExpr",
                    "name": {
                        "type": "IDENTIFIER",
                        "lexeme": "x",
                        "literal": null,
                        "loc": {
                            "line": 2,
                            "col": 6,
                            "len": 1
                        }
                    },
                    "loc": {
                        "line": 2,
                        "col": 6,
                        "len": 1
                    }
                }
            ],
            "loc": {
                "line": 2,
                "col": 0,
                "len": 8
            }
        },
        "loc": {
            "line": 2,
            "col": 0,
            "len": 8
        }
    }
]
```

### Location Information

Every node in the AST includes a `loc` (location) object with:
- `line`: Line number (1-indexed)
- `col`: Column number (0-indexed)
- `len`: Length of the token/expression in characters

This makes it easy to trace errors back to the exact position in source code.

**Use Case: Highlighting Errors in CodeMirror**

With basic math using `line`, `col`, and `len`, you can easily highlight syntax errors in text editors like CodeMirror:

```javascript
// Example: Highlight an error in CodeMirror
function highlightError(editor, error) {
    const { line, col, len } = error.loc;
    
    editor.markText(
        { line: line - 1, ch: col },        // Start position (line is 1-indexed, convert to 0-indexed)
        { line: line - 1, ch: col + len },  // End position
        { className: 'syntax-error' }       // CSS class for styling
    );
}
```

This location data enables real-time error highlighting, go-to-definition features, and other IDE functionality.

---

## Complete Example

```javascript
const { 
    LuaWisp, 
    getTotalErrorCount, 
    highlightErrs, 
    printAST,
    stringify 
} = require('@codewisp/lua-transpiler');

const compiler = new LuaWisp();

// Register functions with type checking
compiler.registerFunction("print", -1);
compiler.registerFunction("split", 2);
compiler.registerFunction("move", 2);
compiler.registerFunction("rotate", 1);
compiler.registerFunction("reset", 0);
compiler.registerFunction("sqrt", 1);
compiler.registerFunction("round", 1);
compiler.registerFunction("random", 2, ["number", "number"]);

// Register built-in objects with typed methods
compiler.registerBuiltinObject("sprite", {
    // Properties
    x: { isFunction: false },
    y: { isFunction: false },
    visible: { isFunction: false },
    size: { isFunction: false },
    width: { isFunction: false },
    height: { isFunction: false },
    costume: { isFunction: false },
    layer: { isFunction: false },
    transparency: { isFunction: false },
    brightness: { isFunction: false },
    
    // Methods with type checking
    setCostume: { isFunction: true, argsLen: 1, argTypes: ["string"] },
    nextCostume: { isFunction: true, argsLen: 0 },
    prevCostume: { isFunction: true, argsLen: 0 },
    getCostume: { isFunction: true, argsLen: 0 },
    pointTowards: { isFunction: true, argsLen: 2, argTypes: ["number", "number"] }
});

compiler.defineBoilerplate(`
function move(x, y) { sprite.x += x; sprite.y += y; }
function rotate(z) { sprite.rotation = z; }
function reset() { sprite.x = 0; sprite.y = 0; }
function round(n) { return Math.round(n); }
function sqrt(n) { return Math.sqrt(n); }
function random(min, max) { return Math.random() * (max - min) + min; }

let sprite = {
    x: 0,
    y: 0,
    visible: true,
    setCostume: (name) => { console.log("Setting costume:", name); },
    pointTowards: (x, y) => { console.log("Pointing to:", x, y); }
};

function print(...args) {
    console.log(args.join(" "));
}
`);

const sourceCode = `
local x = 10
sprite.pointTowards(x, 20)
sprite.setCostume("hero")
print("Position:", sprite.x, sprite.y)
`;

const result = compiler.compile(sourceCode);

if (result.success) {
    console.log("✓ Compilation successful!");
    
    console.log("\n=== Tokens ===");
    console.log(`Token count: ${result.tokens.length}`);
    
    console.log("\n=== AST ===");
    console.log(stringify(result.ast, 4));
    
    console.log("\n=== Output ===");
    console.log("Raw (no boilerplate):");
    console.log(result.raw);
    console.log("\nFinal (with boilerplate):");
    console.log(result.final);
    
} else {
    console.log("✗ Compilation failed!");
    console.log(`\nTotal errors: ${getTotalErrorCount(result.errors)}`);
    console.log(`- Lexer: ${result.errors.lexer.length}`);
    console.log(`- Parser: ${result.errors.parser.length}`);
    console.log(`- Transpiler: ${result.errors.transpiler.length}`);
    
    console.log("\n=== Error Details ===");
    highlightErrs(result.errors, sourceCode);
}
```

### Type Checking Example

```javascript
// This will produce a type error during compilation
const badCode = `
sprite.pointTowards(0, "hello")  -- Error: Expected number, got string
sprite.setCostume(123)           -- Error: Expected string, got number
random("min", "max")             -- Error: Expected number arguments
`;

const badResult = compiler.compile(badCode);

// badResult.success will be false
// badResult.errors.transpiler will contain type mismatch errors

if (!badResult.success) {
    highlightErrs(badResult.errors, badCode);
    // Output:
    // sprite.pointTowards(0, "hello")
    //                        ^^^^^^^-- Function 'pointTowards' expected 'number'.
    //
    // sprite.setCostume(123)
    //                   ^^^-- Function 'setCostume' expected 'string'.
    //
    // random("min", "max")
    //        ^^^^^-- Function 'random' expected 'number'.
}
```

---

## API Reference

| Function | Description |
|----------|-------------|
| `getTotalErrorCount(errors)` | Returns total number of errors across all stages |
| `highlightErrs(errors, source)` | Logs all errors with visual highlighting |
| `printAST(ast)` | Logs the syntax tree as formatted JSON |
| `stringify(obj, indent)` | Custom JSON stringification utility |

### Compiler Methods

| Method | Description |
|--------|-------------|
| `registerFunction(name, argsLen, argTypes?)` | Register a function with optional type checking |
| `registerBuiltinObject(name, properties)` | Register an object with properties and typed methods |
| `registerReservedDeclaration(name)` | Register a reserved declaration name |
| `registerReservedFunction(dslName, jsName)` | Map DSL function name to JS output name |
| `defineBoilerplate(code)` | Set boilerplate code prepended to output |
| `compile(source)` | Compile source code and return result |

### Type Checking

When registering functions or object methods, you can specify argument types using the `argTypes` parameter:

**Supported Types:**
- `"string"` - String literals and expressions
- `"number"` - Number literals and expressions  
- `"boolean"` - Boolean literals (true/false)
- `"null"` - Null literal
- `"unknown"` - Expressions whose type cannot be determined at compile time

**Examples:**

```javascript
// Function with type checking
compiler.registerFunction("random", 2, ["number", "number"]);

// Object method with type checking
compiler.registerBuiltinObject("sprite", {
    setCostume: { 
        isFunction: true, 
        argsLen: 1, 
        argTypes: ["string"] 
    },
    pointTowards: { 
        isFunction: true, 
        argsLen: 2, 
        argTypes: ["number", "number"] 
    }
});
```

**Type Errors:**

Type mismatches are caught during compilation and reported in `result.errors.transpiler`:

```javascript
// This code will fail type checking
sprite.pointTowards(0, "hello")  // Error: Argument 2 expects number, got string
sprite.setCostume(42)            // Error: Argument 1 expects string, got number
```

### Result Properties

| Property | Type | Description |
|----------|------|-------------|
| `success` | `boolean` | Whether compilation succeeded |
| `output` | `string` | Final code with boilerplate (legacy) |
| `raw` | `string` | Code without boilerplate (success only) |
| `final` | `string` | Code with boilerplate (success only) |
| `tokens` | `Token[]` | Array of lexer tokens |
| `ast` | `Stmt[]` | Abstract syntax tree |
| `errors` | `object` | Errors from lexer, parser, transpiler |
| `source` | `string` | Original source code |

---

## Tips

- Always check `result.success` before accessing output
- Use `getTotalErrorCount()` for quick error summary
- Use `highlightErrs()` during development for debugging
- Use `printAST()` or `stringify(result.ast, 4)` to understand how code is parsed
- Use `result.raw` when you don't want boilerplate
- Use `result.final` for complete runnable code
- Add `argTypes` to functions/methods for compile-time type checking
- Type checking only works for registered functions and built-in object methods
- Use `"unknown"` type for arguments that accept any expression type
