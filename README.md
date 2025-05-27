# Ход работы

## 1. Установка и подготовка среды

Работа выполнялась в среде **Ubuntu 22.04**. Установлены:

- `clang`
- `llvm`
- `opt`
- `graphviz`

Установка:
```bash
sudo apt install clang llvm graphviz
```
2. Исходный код
main.c:
```
#include <stdio.h>

int square(int x) {
    return x * x;
}

int main() {
    int a = 5;
    int b = square(a);
    printf("%d\n", b);
    return 0;
}
```
