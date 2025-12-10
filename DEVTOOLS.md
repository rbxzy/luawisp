# DEVTOOLS.md

## LuaWisp Developer Tools & Debugging

This guide covers debugging tools, error handling, and syntax tree inspection available in LuaWisp.

### Table of Contents
- [Setup](#setup)
- [Compilation Result](#compilation-result)
- [Error Handling](#error-handling)
- [Syntax Tree Inspection](#syntax-tree-inspection)
- [Compiler Configuration](#compiler-configuration)
  - [Registering Functions](#registering-functions)
  - [Registering Built-in Objects](#registering-built-in-objects)
  - [Registering Reserved Declarations](#registering-reserved-declarations)
  - [Registering Reserved Functions](#registering-reserved-functions)
  - [Defining Boilerplate](#defining-boilerplate)
- [Complete Example](#complete-example)
- [API Reference](#api-reference)
- [Tips](#tips)

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

## Compiler Configuration

The LuaWisp compiler provides several methods to customize its behavior and define your DSL's semantics.

### Registering Functions

Use `registerFunction()` to make functions available globally in your DSL code.

**Signature:**
```javascript
compiler.registerFunction(name, argsLen, argTypes?)
```

**Parameters:**
- `name` (string) - Function name as it appears in DSL code
- `argsLen` (number) - Number of expected arguments, or `-1` for variadic (any number)
- `argTypes` (array, optional) - Array of type strings for compile-time type checking

**Examples:**

```javascript
// Simple function with 2 arguments, no type checking
compiler.registerFunction("split", 2);

// Variadic function (accepts any number of arguments)
compiler.registerFunction("print", -1);

// Function with type-checked arguments
compiler.registerFunction("random", 2, ["number", "number"]);

// Single-argument function with type checking
compiler.registerFunction("sqrt", 1, ["number"]);

// No-argument function
compiler.registerFunction("getCurrentTime", 0);
```

**Usage in DSL:**
```lua
local result = split("hello,world", ",")
print("Hello", "World", 123)
local num = random(1, 100)
```

**Important Notes:**
- Functions must be registered before compilation
- Type checking is optional but recommended for catching errors early
- Variadic functions (`argsLen: -1`) cannot have type checking
- You must implement the actual function in your boilerplate or runtime

---

### Registering Built-in Objects

Use `registerBuiltinObject()` to define objects with properties and methods that are always available.

**Signature:**
```javascript
compiler.registerBuiltinObject(name, properties)
```

**Parameters:**
- `name` (string) - Object name as it appears in DSL code
- `properties` (object) - Map of property/method definitions

**Property Definition Format:**
```javascript
{
    propertyName: {
        isFunction: boolean,      // true for methods, false for properties
        argsLen?: number,         // Required if isFunction is true
        argTypes?: string[]       // Optional type checking for methods
    }
}
```

**Examples:**

```javascript
// Simple object with properties only
compiler.registerBuiltinObject("game", {
    score: { isFunction: false },
    level: { isFunction: false },
    paused: { isFunction: false }
});

// Object with methods and properties
compiler.registerBuiltinObject("sprite", {
    // Properties
    x: { isFunction: false },
    y: { isFunction: false },
    visible: { isFunction: false },
    
    // Methods without type checking
    show: { isFunction: true, argsLen: 0 },
    hide: { isFunction: true, argsLen: 0 },
    
    // Methods with type checking
    moveTo: { isFunction: true, argsLen: 2, argTypes: ["number", "number"] },
    setCostume: { isFunction: true, argsLen: 1, argTypes: ["string"] },
    say: { isFunction: true, argsLen: 1, argTypes: ["string"] }
});

// Math utilities object
compiler.registerBuiltinObject("Math", {
    PI: { isFunction: false },
    E: { isFunction: false },
    abs: { isFunction: true, argsLen: 1, argTypes: ["number"] },
    max: { isFunction: true, argsLen: 2, argTypes: ["number", "number"] },
    min: { isFunction: true, argsLen: 2, argTypes: ["number", "number"] }
});
```

**Usage in DSL:**
```lua
-- Properties
sprite.x = 100
sprite.y = 200
local currentX = sprite.x

-- Methods
sprite.show()
sprite.moveTo(50, 75)
sprite.setCostume("hero_walk")
sprite.say("Hello!")

-- Math object
local distance = Math.abs(-10)
local highest = Math.max(score, highScore)
```

**Important Notes:**
- Built-in objects are automatically available without declaration
- You must implement the actual object and its methods in your boilerplate
- Type checking helps catch errors like `sprite.moveTo("50", "75")` at compile time
- Properties can be read and written: `sprite.x = 10` or `local x = sprite.x`

---

### Registering Reserved Declarations

Use `registerReservedDeclaration()` to make variable names always available without requiring declaration.

**Signature:**
```javascript
compiler.registerReservedDeclaration(name)
```

**Parameters:**
- `name` (string) - Variable name to reserve

**Examples:**

```javascript
// Reserve common game objects
compiler.registerReservedDeclaration("player");
compiler.registerReservedDeclaration("enemy");
compiler.registerReservedDeclaration("background");

// Reserve utility objects
compiler.registerReservedDeclaration("Input");
compiler.registerReservedDeclaration("Audio");
compiler.registerReservedDeclaration("Storage");
```

**Usage in DSL:**
```lua
-- These can be used without 'local' declaration
player.health = 100
enemy.spawn(10, 20)
background.color = "blue"

Input.onKeyPress("space", function()
    player.jump()
end)
```

**Use Cases:**
- Global game objects (player, world, camera)
- Singleton services (Input, Audio, Network)
- API namespaces (UI, Physics, Storage)
- Constants or configuration objects

**Important Notes:**
- Reserved declarations are treated as always defined
- They cannot be redeclared with `local`
- You must provide the actual implementation in your runtime

---

### Registering Reserved Functions

Use `registerReservedFunction()` to map DSL function names to different JavaScript output names.

**Signature:**
```javascript
compiler.registerReservedFunction(dslName, jsName)
```

**Parameters:**
- `dslName` (string) - Function name in DSL code
- `jsName` (string) - Function name in transpiled JavaScript

**Examples:**

```javascript
// Map DSL event handlers to runtime functions
compiler.registerReservedFunction("_onStart", "onGameStart");
compiler.registerReservedFunction("_onUpdate", "onGameUpdate");
compiler.registerReservedFunction("_onCollision", "onCollisionDetected");

// Map DSL control flow to runtime wrappers
compiler.registerReservedFunction("_forever", "foreverLoop");
compiler.registerReservedFunction("_repeat", "repeatTimes");
compiler.registerReservedFunction("_wait", "waitAsync");

// Map DSL sugar to JavaScript equivalents
compiler.registerReservedFunction("_clone", "structuredClone");
compiler.registerReservedFunction("_typeof", "typeof");
```

**Usage in DSL:**
```lua
_onStart(function()
    print("Game started!")
end)

_onUpdate(function(deltaTime)
    player.update(deltaTime)
end)

_forever(function()
    enemy.patrol()
    _wait(1)
end)

_repeat(10, function()
    spawnEnemy()
end)
```

**Transpiles to:**
```javascript
onGameStart(() => {
    print("Game started!");
});

onGameUpdate((deltaTime) => {
    player.update(deltaTime);
});

foreverLoop(() => {
    enemy.patrol();
    waitAsync(1);
});

repeatTimes(10, () => {
    spawnEnemy();
});
```

**Use Cases:**
- Event handler registration (onClick, onStart, onCollision)
- Async/await wrappers (wait, sleep, delay)
- Custom control flow (forever, repeat, until)
- Language bridges (mapping DSL syntax to framework APIs)

**Important Notes:**
- Reserved functions are purely a transpilation mapping
- You must implement the actual JavaScript function in your runtime
- Useful for creating DSL-friendly syntax that maps to your game engine's API

---

### Defining Boilerplate

Use `defineBoilerplate()` to prepend JavaScript/TypeScript code to every compilation output.

**Signature:**
```javascript
compiler.defineBoilerplate(code)
```

**Parameters:**
- `code` (string) - JavaScript/TypeScript code to prepend

**Examples:**

**Basic Runtime Functions:**
```javascript
compiler.defineBoilerplate(`
function print(...args) {
    console.log(args.join(" "));
}

function wait(seconds) {
    return new Promise(resolve => setTimeout(resolve, seconds * 1000));
}

function random(min, max) {
    return Math.random() * (max - min) + min;
}
`);
```

**Game Engine Integration:**
```javascript
compiler.defineBoilerplate(`
// Import game engine
import { Sprite, Input, Audio } from './gameEngine';

// Global game objects
const sprite = new Sprite();
const player = new Sprite();
const enemy = new Sprite();

// Helper functions
function move(x, y) {
    sprite.x += x;
    sprite.y += y;
}

function rotate(angle) {
    sprite.rotation = angle;
}

// Event handlers
const eventHandlers = {
    onStart: [],
    onUpdate: []
};

function onGameStart(callback) {
    eventHandlers.onStart.push(callback);
}

function onGameUpdate(callback) {
    eventHandlers.onUpdate.push(callback);
}

// Start game loop
gameEngine.run(eventHandlers);
`);
```

**Async Wrapper:**
```javascript
compiler.defineBoilerplate(`
(async function() {
    // Helper functions
    function wait(seconds) {
        return new Promise(resolve => setTimeout(resolve, seconds * 1000));
    }
    
    function forever(callback) {
        return (async function loop() {
            while (true) {
                await callback();
            }
        })();
    }
    
    // User code starts here
    try {
`);

// Note: You might need to close the try-catch and async function
// in your transpiler's output
```

**Type Definitions (TypeScript):**
```javascript
compiler.defineBoilerplate(`
interface Sprite {
    x: number;
    y: number;
    visible: boolean;
    moveTo(x: number, y: number): void;
    show(): void;
    hide(): void;
}

interface GameAPI {
    score: number;
    level: number;
    pause(): void;
    resume(): void;
}

declare const sprite: Sprite;
declare const game: GameAPI;

function print(...args: any[]): void {
    console.log(args.join(" "));
}
`);
```

**Use Cases:**
- Import statements for external libraries
- Polyfills and helper functions
- Global object initialization
- Type definitions for TypeScript
- Async/await wrappers
- Runtime setup code
- API bridges to game engines or frameworks

**Important Notes:**
- Boilerplate is prepended to EVERY compilation
- Access the output with boilerplate via `result.output` or `result.final`
- Access raw output without boilerplate via `result.raw`
- Keep boilerplate lightweight to avoid bloating output
- Consider using ES modules and imports instead of copying code

**Boilerplate Strategy:**
```javascript
// Minimal boilerplate - just imports
compiler.defineBoilerplate(`
import * as runtime from './runtime';
const { print, wait, sprite, player } = runtime;
`);

// Full boilerplate - self-contained
compiler.defineBoilerplate(`
// Everything defined inline
function print(...args) { console.log(args.join(" ")); }
const sprite = { x: 0, y: 0, show() {}, hide() {} };
// ... more code
`);
```

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
