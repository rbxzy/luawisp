# LuaWisp Syntax Reference

Complete syntax guide for the LuaWisp DSL (Lua-style syntax that transpiles to JavaScript/TypeScript).

---

## Table of Contents

- [Variables](#variables)
- [Functions](#functions)
- [Classes and Inheritance](#classes-and-inheritance)
- [Objects](#objects)
- [Lists](#lists)
- [Property Access](#property-access)
- [Data Types](#data-types)
- [Expressions](#expressions)
- [Comments](#comments)
- [Strings](#strings)
- [Control Flow](#control-flow)

---

## Variables

### Local Variables

Local variables are scoped to their block and transpile to `var` declarations.

```lua
local x = 10
local name = "Alice"
local isActive = true
```

**Transpiles to:**
```javascript
var x:any = 10;
var name:any = "Alice";
var isActive:any = true;
```

### Global Variables

Variables declared without `local` are added to the `globals` object.

```lua
x = 10
name = "Alice"
```

**Transpiles to:**
```javascript
globals.x = 10;
globals.name = "Alice";
```

**Note:** It's recommended to use `local` for most variables to avoid polluting the global scope.

---

## Functions

### Local Functions

Local functions are scoped to their block.

```lua
local function greet(name)
    print("Hello, " .. name)
end

greet("World")
```

**Transpiles to:**
```javascript
function greet(name:any) {
console.log(("Hello, " + name));

}
greet("World");
```

### Global Functions

Functions declared without `local` are added to the `globals` object.

```lua
function greet(name)
    print("Hello, " .. name)
end

greet("World")
```

**Transpiles to:**
```javascript
globals.greet = function(name:any) {
console.log(("Hello, " + name));

}
globals.greet("World");
```

### Function Expressions

Functions can be assigned to variables.

```lua
local sayHello = function(name)
    return "Hello, " .. name
end

print(sayHello("Alice"))
```

---

## Classes and Inheritance

### Basic Class

Classes must have an `init` method (constructor).

```lua
class Person
    function init(name, age)
        this.name = name
        this.age = age
    end
    
    function greet()
        print("Hi, I'm " .. this.name)
    end
end

local person = Person("Alice", 30)
person.greet()
```

**Transpiles to:**
```javascript
function Person(name:any,age:any) {
this.name = name;
this.age = age;
}
Person.prototype.greet = function() {
console.log(("Hi, I'm " + this.name));
};
var person:any = new Person("Alice",30);
person.greet();
```

### Class Inheritance

Use `implements` for inheritance (single inheritance only).

```lua
class Animal
    function init(name)
        this.name = name
    end
    
    function speak()
        print(this.name .. " makes a sound")
    end
end

class Dog implements Animal
    function init(name, breed)
        this.name = name
        this.breed = breed
    end
    
    function speak()
        print(this.name .. " barks")
    end
end

local dog = Dog("Buddy", "Golden Retriever")
dog.speak()
```

**Note:** The parent class constructor is automatically called with the same arguments.

---

## Objects

### Object Literals

Objects use `=` for key-value pairs (not `:` like JavaScript).

```lua
local person = {
    name = "Alice",
    age = 30,
    isStudent = false
}

print(person.name)
```

**Transpiles to:**
```javascript
var person:any = { name: "Alice", age: 30, isStudent: false };
console.log((person.name));
```

### Nested Objects

Objects can be nested arbitrarily.

```lua
local person = {
    name = "Alice",
    address = {
        street = "123 Main St",
        city = "Springfield",
        coordinates = {
            lat = 40.7128,
            lng = -74.0060
        }
    }
}

print(person.address.city)
print(person.address.coordinates.lat)
```

### Objects with Methods

Methods can reference the object by name (not `this` - that's only for classes).

```lua
local person = {
    name = "Jim",
    sayName = function()
        print(person.name)
    end
}

person.sayName()
```

**Transpiles to:**
```javascript
var person:any = { name: "Jim", sayName: function() {
console.log((person.name));

} };
person.sayName();
```

---

## Lists

### List Literals

Lists use the same `{}` braces as objects, but without keys.

```lua
local numbers = { 1, 2, 3, 4, 5 }
local names = { "Alice", "Bob", "Charlie" }
local mixed = { 1, "two", 3, true }
```

**Transpiles to:**
```javascript
var numbers:any = [1,2,3,4,5];
var names:any = ["Alice","Bob","Charlie"];
var mixed:any = [1,"two",3,true];
```

### Nested Lists

Lists can be nested.

```lua
local matrix = {
    { 1, 2, 3 },
    { 4, 5, 6 },
    { 7, 8, 9 }
}

print(matrix[0][0])  -- Prints 1
```

### List Indexing

Lists are zero-indexed (like JavaScript arrays).

```lua
local fruits = { "apple", "banana", "orange" }

print(fruits[0])  -- apple
print(fruits[1])  -- banana
print(fruits[2])  -- orange
```

### Lists of Objects

```lua
local people = {
    { name = "Alice", age = 30 },
    { name = "Bob", age = 25 },
    { name = "Charlie", age = 35 }
}

print(people[0].name)  -- Alice
print(people[1].age)   -- 25
```

**Important:** You cannot mix list syntax and object syntax in the same literal:
```lua
-- ❌ ERROR: Cannot mix
local invalid = { 1, 2, name = "test" }

-- ✅ Correct: Choose one
local list = { 1, 2, 3 }
local obj = { a = 1, b = 2 }
```

---

## Property Access

### Member Access (Dot Notation)

```lua
local person = { name = "Alice", age = 30 }
print(person.name)
print(person.age)
```

### Index Access (Bracket Notation)

```lua
local list = { 10, 20, 30 }
print(list[0])
print(list[1])

local x = 1
print(list[x])
```

### Chained Access

```lua
local data = {
    user = {
        profile = {
            name = "Alice"
        }
    }
}

print(data.user.profile.name)
```

### Mixed Access

```lua
local data = {
    users = {
        { name = "Alice" },
        { name = "Bob" }
    }
}

print(data.users[0].name)  -- Alice
```

---

## Data Types

### Numbers

```lua
local integer = 42
local float = 3.14
local negative = -10
```

### Strings

See [Strings](#strings) section for details.

### Booleans

```lua
local isTrue = true
local isFalse = false
```

### Nil

```lua
local empty = nil
```

**Transpiles to:** `null`

---

## Expressions

### Arithmetic Operators

```lua
local x = 10 + 5   -- Addition
local y = 10 - 5   -- Subtraction
local z = 10 * 5   -- Multiplication
local w = 10 / 5   -- Division
local m = 10 % 3   -- Modulo
local p = 2 ^ 8    -- Exponentiation
```

### String Concatenation

```lua
local greeting = "Hello, " .. "World!"
local message = "Count: " .. 42
```

**Transpiles to:** JavaScript `+` operator

### Comparison Operators

```lua
local a = 10 == 10   -- Equal
local b = 10 ~= 5    -- Not equal
local c = 10 > 5     -- Greater than
local d = 10 < 5     -- Less than
local e = 10 >= 10   -- Greater than or equal
local f = 10 <= 5    -- Less than or equal
```

### Logical Operators

```lua
local a = true and false   -- Logical AND
local b = true or false    -- Logical OR
local c = not true         -- Logical NOT
```

### Grouping

```lua
local result = (10 + 5) * 2
local check = (x > 5) and (y < 10)
```

---

## Comments

### Single-Line Comments

```lua
-- This is a single-line comment
local x = 10  -- Comment after code
```

### Multi-Line Comments

Use `--[[` and `]]` for multi-line comments.

```lua
--[[
This is a multi-line comment.
It can span multiple lines.
Useful for documentation.
]]
local x = 10
```

### Long Comments with Equals

Use `--[==[` and `]==]` for comments that need to contain `]]`.

```lua
--[==[
This comment can contain ]] without issues
because it uses the long bracket syntax.
]==]
local x = 10
```

---

## Strings

### Double-Quoted Strings

```lua
local greeting = "Hello, World!"
local message = "She said, \"Hello!\""
```

### Single-Quoted Strings

```lua
local greeting = 'Hello, World!'
local message = 'It\'s a nice day'
```

### Escape Sequences

Both single and double-quoted strings support escape sequences:

```lua
local newline = "Line 1\nLine 2"
local tab = "Column1\tColumn2"
local quote = "He said \"Hi\""
local backslash = "Path: C:\\Users\\Name"
```

Supported escape sequences:
- `\n` - Newline
- `\t` - Tab
- `\r` - Carriage return
- `\\` - Backslash
- `\"` - Double quote
- `\'` - Single quote
- `\0` - Null character
- `\a` - Bell
- `\b` - Backspace
- `\f` - Form feed
- `\v` - Vertical tab

### Multi-Line Strings (Long Strings)

Use `[[` and `]]` for multi-line strings.

```lua
local text = [[
This is a multi-line string.
It preserves line breaks and spacing.
No escape sequences needed for quotes: "Hello" 'World'
]]
```

### Long Strings with Equals

Use `[=[` and `]=]` (or more equals) for strings that need to contain `]]`.

```lua
local code = [==[
This string can contain ]] without issues
because it uses the long bracket syntax with equals.
]==]
```

**Note:** The number of `=` signs must match in opening and closing brackets.

---

## Control Flow

### If Statements

```lua
if x > 10 then
    print("Greater than 10")
elseif x > 5 then
    print("Greater than 5")
else
    print("5 or less")
end
```

### While Loops

```lua
local i = 0
while i < 5 do
    print(i)
    i = i + 1
end
```

### Repeat-Until Loops

```lua
local i = 0
repeat
    print(i)
    i = i + 1
until i >= 5
```

### For Loops

```lua
for i = 1, i < 10, i = i + 1 do
    print(i)
end
```

**Transpiles to:**
```javascript
for (var i:any = 1; i < 10; i = i + 1) {
    console.log((i));
}
```

### Break Statement

```lua
local i = 0
while true do
    print(i)
    i = i + 1
    if i >= 5 then
        break
    end
end
```

---

## Special Features

### Print Statement

Built-in `print` function.

```lua
print("Hello, World!")
print(x + y)
print("Value:", x)
```

**Transpiles to:** `console.log(...)`

### Return Statement

```lua
function add(a, b)
    return a + b
end
```

### This Keyword

`this` is only available inside class methods.

```lua
class Dog
    function init(name)
        this.name = name  -- ✅ Valid in class
    end
end

local obj = {
    name = "test",
    greet = function()
        -- ❌ Cannot use 'this' here
        print(obj.name)  -- ✅ Use object name instead
    end
}
```

---

## Reserved Keywords

The following keywords are reserved and cannot be used as variable names:

- `local`
- `function`
- `class`
- `implements`
- `this`
- `if`, `then`, `else`, `elseif`, `end`
- `while`, `do`, `repeat`, `until`
- `for`
- `break`, `return`
- `and`, `or`, `not`
- `true`, `false`, `nil`

---

## Examples

### Complete Example: Todo List

```lua
class Todo
    function init(title)
        this.title = title
        this.completed = false
    end
    
    function toggle()
        this.completed = not this.completed
    end
end

local todos = {
    Todo("Learn LuaWisp"),
    Todo("Build a project"),
    Todo("Deploy to production")
}

-- Mark first todo as complete
todos[0].toggle()

-- Print all todos
for i = 0, i < 2, i = i + 1 do
    local status = "[ ]"
    if todos[i].completed then
        status = "[x]"
    end
    print(status .. " " .. todos[i].title)
end
```

---

## Best Practices

1. **Use `local` for most variables** - Avoid polluting the global scope
2. **Prefer classes for complex objects** - Use `this` for better method context
3. **Use object names in object methods** - Since `this` isn't available in object literals
4. **Be consistent with quotes** - Pick single or double quotes and stick with it
5. **Use long strings for multi-line text** - Cleaner than escape sequences
6. **Comment your code** - Use `--` for single lines, `--[[]]` for blocks
7. **Zero-index your lists** - Remember lists start at 0, not 1

---

## Differences from Standard Lua

LuaWisp is inspired by Lua but has some differences:

1. **Lists are zero-indexed** (like JavaScript), not one-indexed (like Lua)
2. **`this` keyword for classes** instead of self parameter
3. **Unified `{}` syntax** for both objects and lists (determined by content)
4. **No ipairs/pairs** - Use standard for loops
5. **No metatables** - Use classes for OOP
6. **Classes use `implements`** for inheritance, not Lua's metatable system
7. **Built-in `print` function** transpiles to `console.log`

---

