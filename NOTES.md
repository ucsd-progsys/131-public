# Miscellaneous Notes 

* Stack Alignment on MacOS
* Running `ghcid`



## Stack Alignment on Mac OS

See details [here](http://stackoverflow.com/questions/5649/x86-assembly-on-a-mac/357113#357113)

1. **Allocating Stack** space:

After allocating stack space `push ebp; mov ebp, esp`

```
sub esp, 4 * N
and esp, 0xFFFFFFF0
```

2. **Making a Call**

To pass N parameters, you should first *pad* with `4 * K` bytes,
where `K = (4 - (N mod 4))` bytes, and then `push` the `N`
parameters backwards.

```
sub esp, (4 * K)
push 24
call print
```

3. After the call, restore ESP (to "pop" off the params and padding) by:

```
add esp, 4 * (N + K)
```

Complete worked out example.


```
section .text
extern error
extern print
global our_code_starts_here
our_code_starts_here:
  push ebp
  mov ebp, esp
  sub esp, 0
  and esp, 0xFFFFFFF0
  sub esp, 12
  push 24
  call print
  add esp, 16
  mov esp, ebp
  pop ebp
  ret
```

### Links

+ [x86 interpreter](http://carlosrafaelgn.com.br/Asm86/)

### Setup GhcI

Make this your `.ghci` file

```
:set -fwarn-unused-binds -fwarn-unused-imports
:set -isrc
:load Main
```

Invoke `ghcid` thus:

```
stack exec -- ghcid --command="stack ghci"
```

Run a subset of tests thus:

```
stack test --test-arguments "-p Errors"
```
