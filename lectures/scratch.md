this

---

```nasm
section .text
extern error
extern print
global our_code_starts_here
our_code_starts_here:
  mov eax, 2
  mov [ebp - 4], eax
  mov eax, 4
  mov [ebp - 8], eax
  mov eax, 6
  mov [ebp - 12], eax
  mov eax, [ebp - 4]
  add eax, [ebp - 8]
  add eax, [ebp - 12]
  ret
```

```nasm
section .text
extern error
extern print
global our_code_starts_here
our_code_starts_here:
  push ebp
  mov ebp, esp
  mov eax, 2
  mov [ebp - 4], eax
  mov eax, 4
  mov [ebp - 8], eax
  mov eax, 6
  mov [ebp - 12], eax
  mov eax, [ebp - 4]
  add eax, [ebp - 8]
  add eax, [ebp - 12]
  sub esp, 12
  push eax
  call print
  add esp, 16
  mov esp, ebp
  pop  ebp
  ret
```
