# DEVTOOLS.md

## LuaWisp Developer Tools & Debugging

This guide covers debugging tools, error handling, and syntax tree inspection available in LuaWisp.

---

## Setup

```javascript
const { LuaWisp, getTotalErrorCount, highlightErrs, printAST } = require('@codewisp/lua-transpiler');

const compiler = new LuaWisp();
compiler.registerFunction("print", -1);

compiler.defineBoilerplate(`
function print(...args) {
    console.log(args.join(" "));
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
        "type": "VarDeclStmt",
        "isLocal": true,
        "name": "x",
        "initializer": {
            "type": "NumberLiteral",
            "value": 42
        }
    },
    {
        "type": "FunctionCallStmt",
        "name": "print",
        "args": [
            {
                "type": "Identifier",
                "name": "x"
            }
        ]
    }
]
```

---

## Complete Example

```javascript
const { 
    LuaWisp, 
    getTotalErrorCount, 
    highlightErrs, 
    printAST 
} = require('@codewisp/lua-transpiler');

const compiler = new LuaWisp();
compiler.registerFunction("print", -1);
compiler.defineBoilerplate(`
function print(...args) {
    console.log(args.join(" "));
}
`);

const sourceCode = `
local greeting = "Hello, World!"
print(greeting)
`;

const result = compiler.compile(sourceCode);

if (result.success) {
    console.log("✓ Compilation successful!");
    
    console.log("\n=== Tokens ===");
    console.log(`Token count: ${result.tokens.length}`);
    
    console.log("\n=== AST ===");
    printAST(result.ast);
    
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

---

## API Reference

| Function | Description |
|----------|-------------|
| `getTotalErrorCount(errors)` | Returns total number of errors across all stages |
| `highlightErrs(errors, source)` | Logs all errors with visual highlighting |
| `printAST(ast)` | Logs the syntax tree as formatted JSON |

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
- Use `printAST()` to understand how code is parsed
- Use `result.raw` when you don't want boilerplate
- Use `result.final` for complete runnable code
