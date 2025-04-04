This is a small patch on top of [slimcc](https://github.com/fuhsnn/slimcc) to enable compiler-level `defer` functionality.

Based on the [n3199 proposal](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3199.htm).

UPDATE: The feature has been merged in upstream slimcc, use `_Defer` or enable `defer` with `-fdefer-ts` flag.

## Building

On recent-ish glibc x86-64 Linux, simply `make` or `cc *.c`.

## Testing

Example from [n3199](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n3199.htm):
```
#include <stdbool.h>
#include <stdio.h>

int ex2(void) {
  int r = 4;
  int* p = &r;
  defer { *p = 5; }
  return *p; // return 4;
}

int ex5(void) {
  int r = 0;
  {
    defer {
      defer r *= 4;
      r *= 2;
      defer {
        r += 3;
      }
    }
    defer r += 1;
  }
  return r; // return 20;
}

int main () {
  printf("%d\n", ex2());
  printf("%d\n", ex5());

  // ex4
  {
    defer {
      printf(" meow");
    }
    if (true)
      defer printf("cat");
    printf(" says");
  }
  printf("\n");
  return 0;
}
```
The output should be:
```
4
20
cat says meow
```
## Implementation Experience

slimcc had implemented C99 VLA de-allocation, therefore a basic structure was already in place for tracking VLA declarations (and their effect on stack pointer) across blocks and gotos; it was trivial to extend it to defer statements.

`__attribute__((cleanup(fn)))` was implemented next, as hidden `defer fn(&obj)`s following each declaration. Other than parsing the attribute, most of the work was in code generation, dealing with return values potentially clobbered by cleanup functions.

Lastly, since the compiler got away with not handling secondary-blocks as having their own scope, that had to be fixed.

The actual `defer` parsing bits slid-in gracefully thanks to [chibicc](https://github.com/rui314/chibicc)'s neat design, which slimcc is based on.
