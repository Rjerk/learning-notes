# 使用 objump 生成易读的汇编代码

ref.cpp:

```
#include <cstdio>

int main()
{
    int x = 1;
    int y = 2;
    int &z = x;

    printf ("x = %d, y = %d, z = %d\n", x, y, z);

    ++z;
    printf ("x = %d, y = %d, z = %d\n", x, y, z);

    z += 2;
    printf ("x = %d, y = %d, z = %d\n", x, y, z);
}
```

```
g++ -g -c ref.cpp
objdump -d -M intel -S ref.o > ref.s
```

生成的汇编代码 ref.s:

```

ref.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <main>:
#include <cstdio>

int main()
{
   0:	55                   	push   rbp
   1:	48 89 e5             	mov    rbp,rsp
   4:	48 83 ec 20          	sub    rsp,0x20
   8:	64 48 8b 04 25 28 00 	mov    rax,QWORD PTR fs:0x28
   f:	00 00 
  11:	48 89 45 f8          	mov    QWORD PTR [rbp-0x8],rax
  15:	31 c0                	xor    eax,eax
    int x = 1;
  17:	c7 45 e8 01 00 00 00 	mov    DWORD PTR [rbp-0x18],0x1
    int y = 2;
  1e:	c7 45 ec 02 00 00 00 	mov    DWORD PTR [rbp-0x14],0x2
    int &z = x;
  25:	48 8d 45 e8          	lea    rax,[rbp-0x18]
  29:	48 89 45 f0          	mov    QWORD PTR [rbp-0x10],rax

    printf ("x = %d, y = %d, z = %d\n", x, y, z);
  2d:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  31:	8b 08                	mov    ecx,DWORD PTR [rax]
  33:	8b 45 e8             	mov    eax,DWORD PTR [rbp-0x18]
  36:	8b 55 ec             	mov    edx,DWORD PTR [rbp-0x14]
  39:	89 c6                	mov    esi,eax
  3b:	bf 00 00 00 00       	mov    edi,0x0
  40:	b8 00 00 00 00       	mov    eax,0x0
  45:	e8 00 00 00 00       	call   4a <main+0x4a>

    ++z;
  4a:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  4e:	8b 00                	mov    eax,DWORD PTR [rax]
  50:	8d 50 01             	lea    edx,[rax+0x1]
  53:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  57:	89 10                	mov    DWORD PTR [rax],edx
    printf ("x = %d, y = %d, z = %d\n", x, y, z);
  59:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  5d:	8b 08                	mov    ecx,DWORD PTR [rax]
  5f:	8b 45 e8             	mov    eax,DWORD PTR [rbp-0x18]
  62:	8b 55 ec             	mov    edx,DWORD PTR [rbp-0x14]
  65:	89 c6                	mov    esi,eax
  67:	bf 00 00 00 00       	mov    edi,0x0
  6c:	b8 00 00 00 00       	mov    eax,0x0
  71:	e8 00 00 00 00       	call   76 <main+0x76>

    z += 2;
  76:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  7a:	8b 00                	mov    eax,DWORD PTR [rax]
  7c:	8d 50 02             	lea    edx,[rax+0x2]
  7f:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  83:	89 10                	mov    DWORD PTR [rax],edx
    printf ("x = %d, y = %d, z = %d\n", x, y, z);
  85:	48 8b 45 f0          	mov    rax,QWORD PTR [rbp-0x10]
  89:	8b 08                	mov    ecx,DWORD PTR [rax]
  8b:	8b 45 e8             	mov    eax,DWORD PTR [rbp-0x18]
  8e:	8b 55 ec             	mov    edx,DWORD PTR [rbp-0x14]
  91:	89 c6                	mov    esi,eax
  93:	bf 00 00 00 00       	mov    edi,0x0
  98:	b8 00 00 00 00       	mov    eax,0x0
  9d:	e8 00 00 00 00       	call   a2 <main+0xa2>
}
  a2:	b8 00 00 00 00       	mov    eax,0x0
  a7:	48 8b 75 f8          	mov    rsi,QWORD PTR [rbp-0x8]
  ab:	64 48 33 34 25 28 00 	xor    rsi,QWORD PTR fs:0x28
  b2:	00 00 
  b4:	74 05                	je     bb <main+0xbb>
  b6:	e8 00 00 00 00       	call   bb <main+0xbb>
  bb:	c9                   	leave  
  bc:	c3                   	ret     
```