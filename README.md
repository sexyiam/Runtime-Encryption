# Basic Idea
The basic idea of this trick is to create an encrypted function, where only the current function being executed can be decrypted. This is done by modifying the .text section of the executable to contain the encrypted code. Then, before executing each function, we decrypt it. After executing the function, we re-encrypt it to keep the rest of the program encrypted.

# Specific Implementation
To implement this trick, we can use the gcc **-finstrument-functions** option. This option causes a special function (**__cyg_profile_func_enter** and **__cyg_profile_func_exit**) to be called before and after each function call, respectively. By using these functions, we can modify the code to perform the encryption and decryption tasks.


Basically something like this,
```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <windows.h>

// Hook function
void __cyg_profile_func_enter(void *this_fn, void *call_site) {
}

void __cyg_profile_func_exit(void *this_fn, void *call_site) {
}

// Main function
int main() {
}
```

```c++
int getfunc(void* func, int mark) {
    int i = 0;
    
    while (1) {
        unsigned char currentByte = *(unsigned char*)(func + i);
        
        if (currentByte == 0xe8) {
            int relativeOffset = *(int*)(func + i + 1);
            int targetAddress = (int)func + i + relativeOffset + 5;
            
            if (targetAddress == mark) {
                break;
            }
        }
        
        i++;
    }
    
    return i;
}
```

This allows for an easy calculation of the size of the function. We look for the E8 [offset], or call in x86 asm, opcodes, allowing us to trace and find the absolute address of the resultant location.

```c++
int keylookup(void* func) {
    int j = 0;
    while (keys[j] != 0 && keys[j] != (int)func)
        j += 2;
    if (keys[j] == 0)
        keys[j] = (int)func;
    return j;
}

void encfn(void* func, int i, const char* key) {
    DWORD a;
    unsigned char* code = func;
    VirtualProtect(code, i, PAGE_EXECUTE_READWRITE, &a);
    for (int j = 0; j < i; j++)
        code[j] ^= key[j % 4];
    VirtualProtect(code, i, PAGE_EXECUTE_READ, &a);
}
```

In the code, keys is an array that holds the encrypted keys for the functions. keylookup is a function that searches for a function's key in the keys array. If it does not find a key, it generates one.

encfn is a function that performs XOR encryption and decryption on the function code. It takes a function pointer, an integer that represents the length of the function code, and a string that serves as the key for the encryption.

So, the basic idea is -

The function __cyg_profile_func_enter is called before each function call. In this function, we first decrypt the current function (__cyg_profile_func_enter is actually a pointer to the address of the decrypted function). Then, we add the current function to the callstack and call the function using the **__cyg_profile_func_exit** function pointer.

The function **__cyg_profile_func_exit** is called after each function call. In this function, we first call the function using the __cyg_profile_func_enter function pointer. Then, we remove the current function from the callstack and re-encrypt the current function.

```c++
int exitMark, enterMark;

int fakeEntryPoint();// defined somewhere

int main() {
    srand(time(NULL));
    memset(keysArray, 0, 100 * sizeof(int));
    memset(callstackArray, 0, 100 * sizeof(void*));
    
    exitMark = ((int)__cyg_profile_func_exit);
    enterMark = ((int)__cyg_profile_func_enter);

    printf("%d", fakeEntryPoint()); // Rename original main to avoid conflict
    return 0;
}
```

[img]https://i.imgur.com/cAzGPf1.png[/img]

And our address are called successfully.
