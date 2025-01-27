---
title: "Implementing an interpreter for Brainfuck language Part-1"
date: 2025-01-27
draft: false
---

# Building a Brainfuck Interpreter in Rust

Since compilers are in my radar for a while, I think starting simple and iterating from there might be a go idea for me to fresh my memeory and learn something, also maybe you will learn something from it who knows!

Lets go :)

Brainfuck is a minimalist programming language known for its simplicity and esoteric nature. It consists of only eight commands (`>`, `<`, `+`, `-`, `.`, `,`, `[`, `]`) and operates on a simple memory array. Despite its simplicity, Brainfuck is Turing-complete and capable of expressing any computation.

---

# Brainfuck Interpreter Code in Rust

In this section, we will break down the code step by step, explaining each block in detail.

---

## 1. Core structure Token

### Code

To be honest, `Token` is a bit off in this topic because we are not actually dealing with tokens instead character but it does not really matter...

```rust
#[derive(Debug, PartialEq)]
pub enum Token {
    MoveRight, // >
    MoveLeft,  // <
    Increment, // +
    Decrement, // -
    Output,    // .
    Input,     // ,
    LoopStart, // [
    LoopEnd,   // ]
}
```

### Explanation

The `Token` enum represents each of the eight commands in the Brainfuck language:

- `MoveRight`: Corresponds to `>` (move the pointer one cell to the right).
- `MoveLeft`: Corresponds to `<` (move the pointer one cell to the left).
- `Increment`: Corresponds to `+` (increment the value at the pointer).
- `Decrement`: Corresponds to `-` (decrement the value at the pointer).
- `Output`: Corresponds to `.` (output the value at the pointer as a character).
- `Input`: Corresponds to `,` (accept input from the user).
- `LoopStart` and `LoopEnd`: Represent the start (`[`) and end (`]`) of loops.

We derive `Debug` for debugging and `PartialEq` to allow comparison between tokens.

---

## 2. The `BrainFuckInterpreter` Struct

### Code

The brainfuck language has its own unique context, which basically a memory unit which we go over it like a roller coaster :D 

```rust
pub struct BrainFuckInterpreter {
    memory: Vec<u8>,
    pointer: usize,
    loop_mapping: HashMap<usize, usize>,
}
```

### Explanation

The `BrainFuckInterpreter` struct encapsulates the state of the Brainfuck interpreter:

- `memory`: A vector of 30,000 bytes initialized to `0`. This serves as the Brainfuck memory array.
- `pointer`: A `usize` value pointing to the current memory cell being manipulated.
- `loop_mapping`: A `HashMap` mapping the indices of matching `[` and `]`. This allows efficient jumping between the start and end of loops.

---

## 3. Initializing the Interpreter

### Code

```rust
impl BrainFuckInterpreter {
    pub fn new() -> Self {
        BrainFuckInterpreter {
            memory: vec![0; 30000],
            pointer: 0,
            loop_mapping: HashMap::new(),
        }
    }
}
```

### Explanation

The `new` function initializes a `BrainFuckInterpreter` instance:

- The `memory` is initialized as a vector of 30,000 zeroed bytes.
- The `pointer` is set to the starting position (0).
- `loop_mapping` is initialized as an empty `HashMap`.

This creates a clean slate for the interpreter.

---

## 4. Tokenizing Brainfuck Code

### Code

```rust
pub fn tokenize(&self, code: &str) -> Vec<Token> {
    code.chars().filter_map(|c| match c {
        '>' => Some(Token::MoveRight),
        '<' => Some(Token::MoveLeft),
        '+' => Some(Token::Increment),
        '-' => Some(Token::Decrement),
        '.' => Some(Token::Output),
        ',' => Some(Token::Input),
        '[' => Some(Token::LoopStart),
        ']' => Some(Token::LoopEnd),
        _ => None,
    }).collect()
}
```

### Explanation

The `tokenize` method converts a Brainfuck program (a string) into a vector of `Token`s:

1. `code.chars()`: Iterates through each character in the input string.
2. `filter_map`: Filters out invalid characters and maps valid ones to `Token` variants.
3. `collect()`: Collects the filtered tokens into a vector.

This ensures only valid Brainfuck commands are processed.

---

## 5. Parsing Loops

### Code

```rust
pub fn parse_loop(&mut self, tokens: &Vec<Token>) {
    let mut loop_stack = Vec::new();
    for (i, token) in tokens.iter().enumerate() {
        match token {
            Token::LoopStart => loop_stack.push(i),
            Token::LoopEnd => {
                let start = loop_stack.pop().unwrap();
                self.loop_mapping.insert(start, i);
                self.loop_mapping.insert(i, start);
            },
            _ => (),
        }
    }
    if !loop_stack.is_empty() {
        panic!("Unmatched loop start");
    }
}
```

### Explanation

This method maps matching `[` and `]` commands for efficient loop handling:

1. **Stack-Based Loop Matching**:
   - Use a stack (`loop_stack`) to track the indices of unmatched `[` tokens.
   - When a `]` is encountered, the most recent unmatched `[` is popped, and a mapping is created.

2. **Error Handling**:
   - If `loop_stack` is not empty after processing, it means there's an unmatched `[` token, so the program panics.

---

## 6. Executing Brainfuck Code

### Code

```rust
pub fn execute<W: Write>(&mut self, tokens: &Vec<Token>, output: &mut W) {
    let mut i = 0;
    while i < tokens.len() {
        match tokens[i] {
            Token::MoveRight => self.pointer += 1,
            Token::MoveLeft => self.pointer -= 1,
            Token::Increment => self.memory[self.pointer] += 1,
            Token::Decrement => self.memory[self.pointer] -= 1,
            Token::Output => write!(output, "{}", self.memory[self.pointer] as char).unwrap(),
            Token::Input => {
                let mut buffer = [0];
                std::io::stdin().read_exact(&mut buffer).unwrap();
                self.memory[self.pointer] = buffer[0];
            },
            Token::LoopStart => {
                if self.memory[self.pointer] == 0 {
                    i = self.loop_mapping[&i];
                }
            },
            Token::LoopEnd => {
                if self.memory[self.pointer] != 0 {
                    i = self.loop_mapping[&i];
                }
            },
        }
        i += 1;
    }
}
```

### Explanation

The `execute` method processes the Brainfuck tokens and performs the following:

1. **Pointer Manipulation**:
   - `Token::MoveRight` and `Token::MoveLeft` adjust the memory pointer.
   
2. **Memory Manipulation**:
   - `Token::Increment` and `Token::Decrement` modify the byte at the current pointer.

3. **I/O Operations**:
   - `Token::Output` writes the current byte as a character to the output.
   - `Token::Input` reads a single byte of input and stores it at the current pointer.

4. **Loop Handling**:
   - On `Token::LoopStart`, the interpreter jumps to the matching `]` if the byte at the pointer is `0`.
   - On `Token::LoopEnd`, the interpreter jumps back to the matching `[` if the byte at the pointer is non-zero.

---

## 7. Testing the Interpreter

### Code

```rust
fn main() {
    let code = "++++[>++++<-]>.";
    let mut interpreter = BrainFuckInterpreter::new();
    let tokens = interpreter.tokenize(code);
    interpreter.parse_loop(&tokens);

    let mut output = Vec::new();
    interpreter.execute(&tokens, &mut output);
    println!("{}", String::from_utf8(output).unwrap());
}
```

### Explanation

This test program does the following:

1. `++++`: Increment the first cell to `4`.
2. `[>++++<-]`: Create a loop that moves to the next cell, increments it by `4`, then decrements the first cell. This loop repeats until the first cell becomes `0`.
3. `>.`: Move to the second cell and output its value (`16`).

The program outputs the character corresponding to ASCII value `16`.

So this was easy because we almost showed no effort buuuuut, its extremely usefull to digest this since it is a core concept! 

We will keep improving an my idea is to use `cranelift` to turn this into a compiler and optimize as much as we can.

Code: [Part 1](https://github.com/SinanGncgl/brainfuck-interpreter)

---

