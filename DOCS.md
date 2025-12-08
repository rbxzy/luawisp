# LuaWisp API Documentation

Documentation for the LuaWisp compiler class and registration system.

---

## Table of Contents

- [LuaWisp Class](#luawisp-class)
- [Registering Functions](#registering-functions)
- [Registering Objects](#registering-objects)
- [Registering Reserved Declarations](#registering-reserved-declarations)
- [Registering Reserved Functions](#registering-reserved-functions)
- [Defining Boilerplate](#defining-boilerplate)
- [Compiling Code](#compiling-code)
- [Property Info Schema](#property-info-schema)

---

## LuaWisp Class

The `LuaWisp` class is the main interface for compiling DSL code to JavaScript.

```typescript
import { LuaWisp } from '@codewisp/lua-transpiler';

const compiler = new LuaWisp();
```

Each instance maintains its own registrations (functions, objects, boilerplate, etc.).

---

## Registering Functions

You can register functions that will be available in your DSL with compile-time argument validation.

### Method: `registerFunction(name, argsLen, isLocal?)`

**Parameters:**
- `name` (string): Function name
- `argsLen` (number): Expected number of arguments, or `-1` for variadic (accepts any number)
- `isLocal` (boolean, optional): Default is `true`

**Why register functions?**
- Enables compile-time argument validation
- Prevents runtime errors from incorrect function calls
- Provides clear error messages during compilation

### Examples

```typescript
// Function that expects exactly 1 argument
compiler.registerFunction('wait', 1);

// Function that expects exactly 2 arguments
compiler.registerFunction('setPosition', 2);

// Variadic function (accepts any number of arguments)
compiler.registerFunction('print', -1);
compiler.registerFunction('log', -1);
```

### Using Registered Functions in DSL

```lua
-- Registered with argsLen: 1
wait(1)              -- ✅ Compiles successfully
wait()               -- ❌ Compile error: expects 1 argument
wait(1, 2)           -- ❌ Compile error: expects 1 argument

-- Registered with argsLen: -1 (variadic)
print()              -- ✅ Compiles successfully
print("Hello")       -- ✅ Compiles successfully
print(1, 2, 3, 4)    -- ✅ Compiles successfully
```

**Note:** Registered functions are recognized by the compiler but need implementations. See [Defining Boilerplate](#defining-boilerplate) for providing runtime implementations.

---

## Registering Objects

You can register objects with properties and methods that have compile-time validation.

### Method: `registerBuiltinObject(name, properties)`

**Parameters:**
- `name` (string): Object name
- `properties` (object): Property definitions following the [Property Info Schema](#property-info-schema)

**Why register objects?**
- Compile-time validation prevents typos in property names
- Validates method argument counts
- Provides clear error messages for invalid property/method usage
- Documents your API at the type level

### Property Info Schema

Each property must be defined with:

```typescript
{
    propertyName: {
        isFunction: boolean,    // true for methods, false for properties
        argsLen?: number       // Required if isFunction=true (-1 for variadic)
    }
}
```

### Examples

```typescript
// Register sprite object with properties and methods
compiler.registerBuiltinObject('sprite', {
    // Properties (not functions)
    x: { isFunction: false },
    y: { isFunction: false },
    visible: { isFunction: false },
    rotation: { isFunction: false },
    
    // Methods with fixed argument counts
    setCostume: { isFunction: true, argsLen: 1 },
    moveTo: { isFunction: true, argsLen: 2 },
    
    // Methods with no arguments
    hide: { isFunction: true, argsLen: 0 },
    show: { isFunction: true, argsLen: 0 },
    
    // Variadic method (accepts any number of arguments)
    emit: { isFunction: true, argsLen: -1 }
});

// Register Audio API
compiler.registerBuiltinObject('Audio', {
    volume: { isFunction: false },
    muted: { isFunction: false },
    play: { isFunction: true, argsLen: 1 },
    stop: { isFunction: true, argsLen: 0 }
});
```

### Using Registered Objects in DSL

```lua
-- Valid property access
sprite.x = 100
sprite.visible = true

-- Valid method calls
sprite.setCostume("idle")
sprite.moveTo(10, 20)
sprite.hide()

-- Compile-time errors
sprite.invalidProp = 1         -- ❌ Error: property doesn't exist
sprite.setCostume()            -- ❌ Error: expects 1 argument
sprite.moveTo(10)              -- ❌ Error: expects 2 arguments

-- Variadic method works with any number of args
sprite.emit("click")           -- ✅ Valid
sprite.emit("click", data, 123) -- ✅ Valid
```

---

## Registering Reserved Declarations

You can register object/variable names that are always available without property validation.

### Method: `registerReservedDeclaration(name)`

**Parameters:**
- `name` (string): Name of the reserved declaration

**Why use reserved declarations?**
- For objects with dynamic properties (can't enumerate all properties)
- For wrapping native JavaScript objects (like `Math`, `console`)
- When you don't need compile-time validation
- For third-party APIs with flexible interfaces

### Examples

```typescript
compiler.registerReservedDeclaration('Keyboard');
compiler.registerReservedDeclaration('Mouse');
compiler.registerReservedDeclaration('Game');
compiler.registerReservedDeclaration('Math');
compiler.registerReservedDeclaration('console');
```

### Using Reserved Declarations in DSL

```lua
-- No validation - any property/method allowed
local pressed = Keyboard.isPressed("Space")
local mouseX = Mouse.x
Game.start()
Math.floor(10.5)
console.log("debug")
```

**Difference from `registerBuiltinObject`:**
- `registerBuiltinObject`: Validates properties/methods at compile-time
- `registerReservedDeclaration`: No validation, more flexible but less safe

---

## Registering Reserved Functions

You can map DSL function declarations to JavaScript function calls.

### Method: `registerReservedFunction(dslName, jsName)`

**Parameters:**
- `dslName` (string): Function name in the DSL
- `jsName` (string): Function name in JavaScript output

**Why use reserved functions?**
- Transform DSL function declarations into runtime function calls
- Useful for event handlers, game loop functions, callbacks
- Allows cleaner DSL syntax

**Important:** Reserved functions are for function _declarations_, not direct function calls.

### Examples

```typescript
compiler.registerReservedFunction('_forever', 'forever');
compiler.registerReservedFunction('_on_clone_start', 'onCloneStart');
compiler.registerReservedFunction('_on_key_press', 'onKeyPress');
```

### Using Reserved Functions in DSL

```lua
-- Declare a function using reserved name
local function _forever()
    print("This loops forever")
end

local function _on_key_press()
    sprite.x = sprite.x + 1
end
```

**Transpiles to:**

```javascript
forever(() => {
    console.log("This loops forever");
});

onKeyPress(() => {
    sprite.x = sprite.x + 1;
});
```

---

## Defining Boilerplate

You can define JavaScript code that gets prepended to all successful compilations.

### Method: `defineBoilerplate(code)`

**Parameters:**
- `code` (string): JavaScript/TypeScript code to prepend

**Why use boilerplate?**
- Provide runtime implementations for registered functions
- Include utility functions, polyfills, async wrappers
- Add setup code needed by the transpiled output

**Important:** Only include functions that are NOT part of your game engine runtime. If objects like `sprite`, `Keyboard`, etc. are already available in your game engine, don't include them in the boilerplate.

### What to Include

✅ **Include:** Custom function implementations you registered
```typescript
compiler.defineBoilerplate(`
// Implementations for registered functions
function print(...args) {
    console.log(...args);
}

async function wait(seconds) {
    return new Promise(resolve => setTimeout(resolve, seconds * 1000));
}
`);
```

❌ **Don't Include:** Game engine objects that already exist
```typescript
// DON'T do this if sprite is provided by your game engine
compiler.defineBoilerplate(`
const sprite = { x: 0, y: 0 };  // ❌ Game engine provides this
`);
```

### Complete Example

```typescript
// Register custom functions
compiler.registerFunction('print', -1);
compiler.registerFunction('wait', 1);
compiler.registerFunction('asyncMultiplayer', 2);

// Provide implementations
compiler.defineBoilerplate(`
// Custom print function
function print(...args) {
    console.log(...args);
}

// Async wait function
async function wait(seconds) {
    return new Promise(resolve => setTimeout(resolve, seconds * 1000));
}

// Async multiplayer API wrapper
async function asyncMultiplayer(action, data) {
    const response = await fetch('/api/multiplayer', { // Replace with await Mulitplayer.connect or however it's already defined in your engine
        method: 'POST',
        body: JSON.stringify({ action, data })
    });
    return await response.json();
}
`);
```

---

## Compiling Code

Compile your DSL source code to JavaScript.

### Method: `compile(source)`

**Parameters:**
- `source` (string): DSL source code

**Returns:** `CompilationResult` object

```typescript
interface CompilationResult {
    success: boolean;        // True if compilation succeeded
    output: string;          // Final code with boilerplate
    raw?: string;            // Raw code without boilerplate (only on success)
    final?: string;          // Final code with boilerplate (only on success)
    tokens: Token[];         // Lexer tokens
    ast: Stmt[] | null;      // Abstract syntax tree
    errors: {
        lexer: Err[];        // Lexical errors
        parser: Err[];       // Parse errors
        transpiler: Err[];   // Semantic/validation errors
    };
    source: string;          // Original source
}
```

### Output Fields

- **`result.output`**: Final code with boilerplate (always use this)
- **`result.raw`**: Raw transpiled code without boilerplate (available on success)
- **`result.final`**: Same as `output` (available on success)

### Example

```typescript
const code = `
local x = 10
print(x)
`;

const result = compiler.compile(code);

if (result.success) {
    // Use result.output - includes boilerplate
    console.log(result.output);
    
    // Or access raw transpiled code
    console.log(result.raw);
} else {
    // Handle errors
    result.errors.lexer.forEach(err => {
        console.log(`Lexer error at ${err.line}:${err.col} - ${err.error}`);
    });
    result.errors.parser.forEach(err => {
        console.log(`Parser error at ${err.line}:${err.col} - ${err.error}`);
    });
    result.errors.transpiler.forEach(err => {
        console.log(`Transpiler error at ${err.line}:${err.col} - ${err.error}`);
    });
}
```

---

## Property Info Schema

When registering objects with `registerBuiltinObject`, each property follows this schema:

```typescript
{
    isFunction: boolean,      // true = method, false = property
    argsLen?: number         // Required if isFunction=true
                             // Use -1 for variadic functions
}
```

### Examples

```typescript
{
    // Property (read/write)
    x: { isFunction: false },
    
    // Method with no arguments
    reset: { isFunction: true, argsLen: 0 },
    
    // Method with 2 arguments
    moveTo: { isFunction: true, argsLen: 2 },
    
    // Variadic method (any number of arguments)
    emit: { isFunction: true, argsLen: -1 }
}
```

---

## Complete Usage Example

```typescript
import { LuaWisp } from '@codewisp/lua-transpiler';

// Create compiler
const compiler = new LuaWisp();

// 1. Register game engine objects (with validation)
compiler.registerBuiltinObject('sprite', {
    x: { isFunction: false },
    y: { isFunction: false },
    setCostume: { isFunction: true, argsLen: 1 },
    hide: { isFunction: true, argsLen: 0 }
});

// 2. Register objects without validation
compiler.registerReservedDeclaration('Keyboard');
compiler.registerReservedDeclaration('Game');

// 3. Register custom functions
compiler.registerFunction('print', -1);      // Variadic
compiler.registerFunction('wait', 1);        // 1 argument

// 4. Register reserved functions
compiler.registerReservedFunction('_forever', 'forever');

// 5. Define boilerplate (only custom functions)
compiler.defineBoilerplate(`
function print(...args) {
    console.log(...args);
}

async function wait(seconds) {
    return new Promise(resolve => setTimeout(resolve, seconds * 1000));
}
`);

// 6. Compile DSL code
const dslCode = `
sprite.x = 100
sprite.setCostume("idle")

local function _forever()
    if Keyboard.isPressed("Space") then
        sprite.y = sprite.y - 10
    end
end

print("Game started!")
wait(1)
`;

const result = compiler.compile(dslCode);

if (result.success) {
    // result.output contains boilerplate + transpiled code
    eval(result.output);
} else {
    console.error('Compilation failed:', result.errors);
}
```

---

## Other Methods

### `clearCustomRegistrations()`

Clears all registrations (functions, objects, reserved declarations, reserved functions, and boilerplate).

```typescript
compiler.clearCustomRegistrations();
// All registrations cleared - fresh state
```

---

For DSL syntax documentation, see [SYNTAX.md](./SYNTAX.md).
