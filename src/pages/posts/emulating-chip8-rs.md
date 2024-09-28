---
layout: ../../layouts/post.astro
title: 'Emulating Chip-8 in Rust'
author: 'Todd'
pubDate: 2023-03-04
---

Taking a little bit of a break from other projects, I decided to start a new short-lived project to play around with. Something I had been wanting to do for awhile. Writing most to all of an emulator from nothing. My experience with this was in a class, doing homework assignments with an emulator where large portions were pre-built by the instructor. The homework assignments were usually something along the lines of, "implement floating point using ieee 754 in the emulator." Which was a fun assignment, but doesnt give the same satisfaction of starting with writing an emulator starting from a blank source file.

For writing my first emulator, I have decided to go with the Chip-8. This is considered the "hello world" of writing emulators. There are tons of test roms out there and the instruction set is fairly simple. You also don't need to worry too much about implement multiple parts, such as emulating the TIA chip that exists in the Atari 2600. The Chip-8 has about 35-opcodes, one of which is not usually implemented by emulators (0NNN). From research there is a site that exists as probably the best resource for finding a [specification](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM#00E0) for the processor.

The Chip-8 can also be described as an "interpreted programming language" since it appears that a Chip-8 processor was never actually produced. Instead, it was commonly emulated on other computers. In the early days, it was often used on the COSMAC VIP computer made by RCA that has an RCA 1802 processor. Also taking a look at the specification, the address 0x000 to 0x1FF was the block where the interpreters were usually packaged in. However, I will be implementing this as an emulator with rust, so needed an interpreter packaged inside of the ROM is not really necessary.

Before diving into the technical details of the code, here is the repository where my code lives.

[https://github.com/martintc/chp8-rust-emulator](Chip-8 emulator in rust)

As of the writing of this post, the Chip-8 emulator does not support input, but from testing, it all works. I utilized SDL2 for drawing the display and made use of tinyrand for generating a random number as will be seen later. The meat of this program is located in `src/cpu.rs`.

Chip-8 has only 4096 bytes of RAM ranging from `0x0000` to `0xFFFF`. The memory layout is fairly straight forward. As mentioned before, addresses `0x0000` to `0x01FFF` are reserved for the interpreter. Most Chip-8 programs start at address `0x0200` and go to `0xFFFF`. To represent this, using an array of 4096 of type `u8` is sufficient for emulating the RAM.

There are 16 general purpose registers which are 8-bits in length. Much like the RAM, these can be implemented in code as an array of type `u8` with a length of 16. The special registers are the stack pointer, delay timer, sound timer, program counter and the index register. The index and program counter can be presented as `u16` since they have a 16-it length with the reamining special registers being represented as `u8.`

For those who are not familiar with what that means for registers. The general purpose registers are gor any kind of general use. There is a caveat here. Register 16 (0xf) is not really that 'general purpose.' This register is used for any flags such as carry on add or borrow for subtraction. The index register is primary used to point to a location in memory where a sprite begins (graphic). Program counter keeps our current place in RAM where we fetch instructions to run. The stack pointer allows us to call functions/subroutines and return back to previous thread of execution once that is complete by calling to return. These are the registers we will primarily worry about. The delay timer and sound timer are fairly straight forward based on thier name.

In order to allow for subroutines/functions, the Chip-8 has a stack as alluded to in the previous paragraph with the stack pointer. The stack for the Chip-8 can store 16 addresses on the stack and these addresses can be captured as a `u16`. For those not familiar with the stack we are talking about here. A call can be made to jump to an address that serves as the start of a routine, we can think of this as a function in rust.
```rust
  fn add_numbers() {
   let x = 1 + 1;
  }

  fn main () {
   println ("hello");
   add_numbers();
  }
```
While the code above isn't very useful it can help demonstrate the concept. When executing, we print a statement. Then we call a function. The simple version is, when the function is called, we store a where we were in the main function when executing then jump to the address where the `add_numbers` function starts. This is done by setting the program counter register to the address that starts the `add_numbers` function. Once that function is done executing, there is a call to return (here it would be implicit), which would take a look at the stack and recall where we jumped from. Once again, by getting this address and jumping back to it by setting the program counter register to that address.

The other big system to cover is the display. Chip-8's display is 64 by 32 pixels. Here is where we supply out own abstraction that drifts a little bit from the implementation. I represent this as a two dimensional array of `u8`s. Each `u8` 'entry' will be considered a pixel. The Chip-8 draws based on sprites, so we are always drawing sprite. Sprite have a limitation, that we will discuss. But just understand that the display is 64 by 32 and we focus on drawing sprites unlike other retro cpus.

The CPU can be captured in a data structure. For this emulator, this is how that is structured.
```rust
  pub struct Cpu {
    ram: [u8; 0xfff],
    pub vram: [[u8; 32]; 64],
    reg: [u8; 0x10], // registers
    i: u16,          // index register
    pc: u16,         // program counter
    stack: [u16; 0x10],
    sp: u8, // stack pointer
    dt: u8, // delay timer
    st: u8, // sound timer
    keypad: [u8; 0x10],
    rand: StdRand,
  }
```
Then, initializing the structure is as follows.
```rust
  pub fn new() -> Self {
      Self {
          ram: [0x0; 0xfff],
          vram: [[0x0; 32]; 64],
          reg: [0x0; 0x10],
          i: 0x0,
          pc: 0x200, // initial start adress once ROM is loaded
          stack: [0x0; 0x10],
          sp: 0x0,
          dt: 0x0,
          st: 0x0,
          keypad: [0x0; 0x10],
          rand: StdRand::default(),
      }
  }
```
Initialize all the values to zero except for the program counter. As mentioned earlier, programs start at address `0x0200`, so this is where we want to go ahead and set the program counter.

With that out of the way, we can talk about loading the instructions (ROM), which is really simple. When given a ROM, we can simply read the file. Since all ROMs start at `0x200` and we know that anything below that is reserved for an interpreter and we don't need the interpreter, we read in the file as a series of bytes and copy them into the RAM array starting at index `0x200` in the array.
```rust
   pub fn load_rom(&mut self, input: Vec<u8>) {
       let mut address: usize = 0x200;
       for byte in input.iter() {
           self.ram[address] = *byte;
           address += 1;
       }
   }
```
With that out of the way, we are ready to begin executing.

Executing an instructions starts by interpreting what the instruction is. We know that the first instruction executed is at address `0x0200`. The CHIP-8 is a [big-endian](https://www.techtarget.com/searchnetworking/definition/big-endian-and-little-endian) system. So an instruction `0x8121` is stored just like that in memory. The first digit `8` will give us the first clue as to what opcode we are looking to run. Looking at the specification, there are several instructions that begin with an `8.` We have `8xy0` and `8xy1` and `8xyE` and so one. The big differentiator here is now the last nibble (4 bits) on the opcode. So once we see it starts with `8`, we need to see what the last nibble is, which in `0x8121`, it is `1`. So looking at the specification again, this is `OR Vx, Vy`.

Back to reading it in. RAM is an array of `u8`s and we know an instruction in CHIP-8 is 16-bits, which we can call a `u16`. The way we can read this in is by first reading in the byte as address `0x0200`. Then we read in the byte at `0x0201` We can construct a `u16` with the first byte shifted to the upper 8 bits of that and the second byte occupying the lower 8 bits.
```rust
  let msb = self.ram[self.pc as usize] as u16;
  self.pc += 1;
  let lsb = self.ram[self.pc as usize] as u16;
  let mut inst: u16 = msb << 8;
  inst |= lsb;
  self.pc += 1;
```
Now it is ready to intrepet and we can do so by first checking the upper 4 bits (nibble). Then depending on that decide if it leads us right to an opcode or if we need to interpret it some more such as reading the last 4 bits (nibble) such as we did above in `0x8121`.
```rust
        match inst & 0xf000 {
            0x0000 => match inst & 0x00ff {
                0x00e0 => self.op_00e0(inst),
                0x00ee => self.op_00ee(inst),
                _ => panic!("Instruction not valid: {}", inst),
            },
            0x1000 => self.op_1nnn(inst),
            0x2000 => self.op_2nnn(inst),
            0x3000 => self.op_3xkk(inst),
            0x4000 => self.op_4xkk(inst),
            0x5000 => self.op_5xy0(inst),
            0x6000 => self.op_6xkk(inst),
            0x7000 => self.op_7xkk(inst),
            0x8000 => match inst & 0x000f {
                0x0000 => self.op_8xy0(inst),
                0x0001 => self.op_8xy1(inst),
                0x0002 => self.op_8xy2(inst),
                0x0003 => self.op_8xy3(inst),
                0x0004 => self.op_8xy4(inst),
                0x0005 => self.op_8xy5(inst),
                0x0006 => self.op_8xy6(inst),
                0x0007 => self.op_8xy7(inst),
                0x000e => self.op_8xye(inst),
                _ => panic!("Instruction not valid: {}", inst),
            },
            0x9000 => self.op_9xy0(inst),
            0xa000 => self.op_annn(inst),
            0xb000 => self.op_bnnn(inst),
            0xc000 => self.op_cxkk(inst),
            0xd000 => self.op_dxyn(inst),
            0xe000 => match inst & 0x00ff {
                0x009e => self.op_ex9e(inst),
                0x00a1 => self.op_exa1(inst),
                _ => panic!("Instruction not valid: {}", inst),
            },
            0xf000 => match inst & 0x00ff {
                0x0007 => self.op_fx07(inst),
                0x000a => self.op_fx0a(inst),
                0x0015 => self.op_fx15(inst),
                0x0018 => self.op_fx18(inst),
                0x001e => self.op_fx1e(inst),
                0x0029 => self.op_fx29(inst),
                0x0033 => self.op_fx33(inst),
                0x0055 => self.op_fx55(inst),
                0x0065 => self.op_fx65(inst),
                _ => panic!("Instruction not valid: {}", inst),
            },
            _ => panic!("Instruction not valid: {}", inst),
        }
```
This is essentially sort of like a decision tree happening or a state machine. Drilling down until we find the proper opcode to run.

Now it is time to talk a little more about opcode formatting. In the program, I have kept the functions close to a representation of the spec. So as an example the instructions `0x8122` maps to `0x8xy2`. The `x` and `y` here specify registers we can pass in to the instruction. Looking at the spec of `0x8xy2`, we can see this resolves to `AND Vx, Vy`. We will perform a logical `AND` operation on the registers x and y, these registers can be any of the 16 general purpose registers. 
```rust
  // and - bit wise an on rgisters vx and vy with the result going into vx
  // 8xy2
  pub fn op_8xy2(&mut self, inst: u16) {
      let vx = ((inst & 0x0f00) >> 8) as usize;
      let vy = ((inst & 0x00f0) >> 4) as usize;
      self.reg[vx] &= self.reg[vy];
  }
```
Looking at the instruction above the `0x8122` we pass the full 16-bit instruction. Now we seperate out the `x` and `y` registers using some bitewise operations. Then we perform the logical `AND`.

From there, I think that is enough information on how the code itself works. Now I will talk about the experience of writing it in Rust.

First, unit testing came in handy. As you notice, most of the `cpu.rs` file isn't event code for implementing the emulator, most of it (as of this writing 900 LOC) is dedicated to writing unit tests. which helped troubleshoot. It can also be a good resource to understand how it works since I hand write the roms with instructions and performs tests on them.

One of my biggest snags was forgetting some of Rust's safety features. So it would mess up the emulator because Rust protects against arthimetic overflow. This is why in some implementations of instructions, you see me upcast everything to `u16` then downcase to `u8` at the end. 
