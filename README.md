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
![image](https://github.com/user-attachments/assets/810abddd-4fdb-4fca-9877-760c6628499e)

## 2. Исходный код
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
![image](https://github.com/user-attachments/assets/42d31780-cb49-4384-8adf-472c64f595e3)
## 3. Получение AST
```bash
clang -Xclang -ast-dump -fsyntax-only main.c
```
![image](https://github.com/user-attachments/assets/21d338eb-40e0-4b83-a5ae-bd30bcef3795)
Функция square принята, содержит параметр x и возвращает x * x.
## 4. Генерация LLVM IR
```bash
clang -S -emit-llvm main.c -o main.ll
```
![image](https://github.com/user-attachments/assets/0af09854-a11b-426e-b85d-2b9d405bb7fa)
## 5. Оптимизация IR
```bash
clang -O0 -S -emit-llvm main.c -o main_O0.ll
```
Стоит отметить, что в файле с IR до оптимизации:
Все переменные (a, b, x.addr) размещены в памяти через alloca;
Множество операций load и store;
square вызывается как отдельная функция.
![image](https://github.com/user-attachments/assets/873957d1-4804-419e-b4d9-f52c4ef606ab)
```bash
clang -O2 -S -emit-llvm main.c -o main_O2.ll
```
Команда -O2 – комплексная оптимизация среднего уровня. Она
применяет более 30 различных оптимизаций:
-inline – встраивание небольших функций (встраивает square в
main, если она вызывается один раз);
-constprop – подставит значение square(5) → 25, если функция
встроена и всё известно на этапе компиляции;
-mem2reg – перевод переменных из памяти в регистры (SSA);
-instcombine – объединение и упрощение инструкций
(упростит арифметику, например x * x может быть преобразовано в shl при
x = 2^n);
-simplifycfg – оптимизирует структуру блоков (Упростит граф
управления, если после inlining останутся лишние блоки);
-reassociate, -gvn, -sroa, -dce и другие.
В файле с IR после оптимизации:
Вся функция square исчезла – она была встроена (-inline) и затем
вычислена (оптимизация -constprop);
Никаких переменных, alloca, store, load – всё удалено (оптимизации
-mem2reg, -dce);
Остался только вызов printf(25).
```bash
diff main_O0.ll main_O2.ll
```
Сравнение двух файлов:
![image](https://github.com/user-attachments/assets/034da53c-6285-4ebf-90cc-ca45134ae94f)
Стоит отметить, что после оптимизации произошли следующие
изменения:
-Переменные типа alloca были удалены;
-Код переведён в SSA-форму;
-Оптимизация улучшила читаемость и упростила поток
управления.
6. Граф потока управления программы
Команда для генерации оптимизированного LLVM IR: 
```bash
clang -O2 -S -emit-llvm main.c -o main.ll
```
Команда для генерации .dot-файлов CFG для функций: 
```bash
opt -dot-cfg -disable-output main.ll
```
Эта команда создаст DOT-файлы: .main.dot – для функции main;
.square.dot – для square, если она не была удалена оптимизацией.
Команда для установки библиотеки Graphviz: 
```bash
sudo apt install graphviz
```
Команды для преобразования файлов с расширением .dot в .png с
помощью Graphviz:
```bash
dot -Tpng .main.dot -o cfg_main.png
```
```bash
dot -Tpng .square.dot -o cfg_square.png
```
Команды для просмотра файлов с CGF:
```bash
xdg-open cfg_main.png
```
![image](https://github.com/user-attachments/assets/841b5647-a9f4-454d-9fa8-4607215626f2)
```bash
xdg-open cfg_square.png
```
![image](https://github.com/user-attachments/assets/4532de07-2dbf-4863-91ae-967eddb95fb3)
Стоит отметить, что в LLVM каждый граф потока управления (CFG)
строится на уровне функции, поскольку структура управления всегда
локальна для тела функции. Для получения полного представления о
программе, нужно построить CFG для всех функций и анализировать их
совокупность. Автоматическое объединение всех CFG в один граф не
предусмотрено в LLVM по умолчанию.
## Выводы
● С помощью Clang можно получить полную структуру AST и
IR, а также CGF;
● LLVM предоставляет гибкие инструменты анализа и
оптимизации;
● Промежуточное представление кода удобно для написания
компиляторных трансформаций.

# Дополнительно задание. Вариант 1. Объявление комплексного числа с инициализацией на языке C++
Напишите программу, в которой создается объект типа std::complex<double> z(3.0, 4.0);, и выведите его модуль. Постройте AST и
LLVM IR, проанализируйте, как компилятор обрабатывает вызов
конструктора и вычисление модуля.
## 1.  Исходный код 
complex_example.cpp
```bash
#include <iostream>
#include <complex>

int main() {
    std::complex<double> z(3.0, 4.0);
    std::cout << "Module: " << std::abs(z) << std::endl;
    return 0;
}
```
![image](https://github.com/user-attachments/assets/e8276856-3e0d-4e4c-b39d-247089c4ba5e)
## 2. Генерация AST
Для построения AST используем Clang:
```bash
clang -Xclang -ast-dump -fsyntax-only complex_example.cpp
```
![image](https://github.com/user-attachments/assets/a8901339-e2ce-4fa2-9d4a-c113a9be52c5)
## 3. Генерация LLVM IR
```bash
clang -O2 -S -emit-llvm complex_example.cpp -o complex_example.ll
```
![image](https://github.com/user-attachments/assets/0d2239a5-efb4-482e-b696-48605b273d9c)
##  4. Анализ оптимизаций
При -O2 компилятор выполняет:
- Вычисление модуля на этапе компиляции:
```
3.0² + 4.0² = 25.0
- sqrt(25.0) → 5.0
```

## 5. Визуализация CFG
```bash
opt -dot-cfg complex_example.ll
```
Создали два файла:
.main.dot — граф управления для функции main()
._GLOBAL__sub_I_complex_example.cpp.dot 
```bash
dot -Tpng .main.dot -o cfg_main.png
```
Преобразовали .main.dot в изображение cfg_main.png
![image](https://github.com/user-attachments/assets/fc68a545-43bd-4ec8-83da-f0602c7c2dd5)

## 6. Результат выполнения
Программа выведет:
Module: 5
## Выводы
- Компилятор эффективно оптимизирует операции с `std::complex`
- Конструктор и вычисление модуля раскрываются в элементарные операции
- При включенной оптимизации математические вычисления выполняются на этапе компиляции
- AST показывает полную структуру вызовов, включая неявные преобразования
- LLVM IR демонстрирует низкоуровневую реализацию операций с комплексными числами
