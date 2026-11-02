## Этап 1. Статическая библиотека (.a)
Скомпилируйте код и свяжите со статической библиотекой.
Подсказка: используйте ar rcs и g++ -L. для статической компоновки.
```
 gcc -c vector_math.c -o vector_math.o
user@Admin:~/lab3$ g++ -c main.cpp -o main.o
user@Admin:~/lab3$ ar rcs libmath_static.a vector_math.o
user@Admin:~/lab3$ g++ main.o -L. -lmath_static -o app_static
user@Admin:~/lab3$ ./app_static
Library version: v2.4.1-release
Dot product result: 32
user@Admin:~/lab3$ nm main.o
                 U _GLOBAL_OFFSET_TABLE_
                 U _ZNSolsEPFRSoS_E
                 U _ZNSolsEi
                 U _ZSt21ios_base_library_initv
                 U _ZSt4cout
                 U _ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_
                 U _ZStlsISt11char_traitsIcEERSt13basic_ostreamIcT_ES5_PKc
                 U __stack_chk_fail
                 U dot_product
                 U get_lib_version
0000000000000000 T main
user@Admin:~/lab3$    ar tv libmath_static.a
rw-r--r-- 0/0   1880 Jan  1 03:00 1970 vector_math.o
user@Admin:~/lab3$    ar rcs libmath_static.a vector_math.o
user@Admin:~/lab3$    ar tv libmath_static.a
rw-r--r-- 0/0   1880 Jan  1 03:00 1970 vector_math.o
user@Admin:~/lab3$
```

Вопросы:
### Коротко опишите, что делают флаги утилиты ar: r, c, s . Что произойдёт, если запустить команду второй раз с тем же именем архива?
Ошибки нет, архив не раздваивается: r заменяет модуль с тем же именем, поэтому в архиве по-прежнему один vector_math.o. Размер и содержимое те же, потому что объектный файл ты не менялась. Если бы ты пересобрала vector_math.o после правки кода и снова запустила ar rcs, в архиве оказалась бы новая версия.


### С помощью утилиты nm посмотрите, что получилось в main.o. Как вы думаете, почему функция main не подверглась name mangling (имеет "чистое" имя), хотя файл скомпилирован компилятором C++?
В nm main.o видно две группы символов:

Искажённые имена вроде _ZSt4cout и _ZNSolsEi: это функции и объекты стандартной библиотеки). C++ допускает перегрузку, поэтому компилятор кодирует в имя пространство имён и типы аргументов.
Имя main не искажается. Потому что код, который запускает программу и вызывает main, написан на C и ищет символ с точным именем main. Ну она и не перегружается
 U dot_product, U get_lib_version: они тоже с чистыми именами, потому что в заголовке стоит extern "C".

## Этап 2. Динамическая библиотека (.so)
Попробуйте собрать .so без флага -fPIC и зафиксируйте результат:
gcc -c vector_math.c -o vector_math_nopic.o
gcc -shared -o libmath_broken.so vector_math_nopic.o
ничего не вывелось
Скомпилируйте C-код с флагом PIC и соберите разделяемую библиотеку:
gcc -c -fPIC vector_math.c -o vector_math_pic.o
gcc -shared -o libmath_dynamic.so vector_math_pic.o
```
у меня
gcc -v 2>&1 | grep -o -- '--enable-default-pie'
--enable-default-pie
user@Admin:~/lab3$ gcc -c -fno-pic -fno-pie vector_math.c -o vector_math_nopic.o
user@Admin:~/lab3$ gcc -shared -o libmath_broken.so vector_math_nopic.o
user@Admin:~/lab3$
```
Слинкуйте C++-код с динамической библиотекой:
g++ main.o -L. -lmath_dynamic -o app_dynamic
Вопросы:
Приведите текст полученной на шаге 1 ошибки. Комментарий: для создания .a флаг -fPIC не нужен, а для .so — обязателен. Подробное объяснение механизма, стоящего за этим ограничением — таблицы GOT и позиционно-независимый код, — будет дано на четвёртой лекции.

У меня .so собрался без ошибки....
```
у меня
gcc -v 2>&1 | grep -o -- '--enable-default-pie'
--enable-default-pie
user@Admin:~/lab3$ gcc -c -fno-pic -fno-pie vector_math.c -o vector_math_nopic.o
user@Admin:~/lab3$ gcc -shared -o libmath_broken.so vector_math_nopic.o
user@Admin:~/lab3$
```
вывод readelf -r
```
 readelf -r vector_math_nopic.o

Relocation section '.rela.text' at offset 0x268 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
00000000000b  000300000002 R_X86_64_PC32     0000000000000000 .data - 4

Relocation section '.rela.data' at offset 0x280 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  000400000001 R_X86_64_64       0000000000000000 .rodata + 0

Relocation section '.rela.eh_frame' at offset 0x298 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000020  000200000002 R_X86_64_PC32     0000000000000000 .text + 0
000000000040  000200000002 R_X86_64_PC32     0000000000000000 .text + 11
```

Сравните размеры сегментов text для app_static и app_dynamic (через size). Куда делся код функций библиотеки в динамическом варианте?
```
user@Admin:~/lab3$ size app_static app_dynamic
   text    data     bss     dec     hex filename
   2545     656     280    3481     d99 app_static
   2514     680     280    3474     d92 app_dynamic
user@Admin:~/lab3$ ldd app_dynamic
        linux-vdso.so.1 (0x000070e382aa3000)
        libmath_dynamic.so => not found
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x000070e382600000)
        libc.so.6 => /usr/lib/x86_64-linux-gnu/libc.so.6 (0x000070e382200000)
        libm.so.6 => /usr/lib/x86_64-linux-gnu/libm.so.6 (0x000070e38296b000)
        /lib64/ld-linux-x86-64.so.2 (0x000070e382aa5000)
        libgcc_s.so.1 => /usr/lib/x86_64-linux-gnu/libgcc_s.so.1 (0x000070e38293d000)
```
Сегмент text у динамической версии меньше всего на 31 байт. Это ожидаемо: функции dot_product и get_lib_version крошечные (несколько десятков байт кода), так что выигрыш почти незаметен. Остальной размер занимает код самой программы и служебные секции. Если бы в библиотеке была большая математика, разница была бы существенной.

Заодно видно, что data у динамической версии чуть выросла (656 → 680). Это служебные таблицы для динамической линковки (GOT и подобное), которых в статическом варианте нет.
Куда делся код библиотеки

size показывает, что text у app_dynamic меньше всего на 31 байт - обе функции маленькие. В app_static их код скопирован в .text на этапе компоновки.
В app_dynamic этого кода нет: в main.o по nm только U dot_product и U get_lib_version, а сам код лежит в libmath_dynamic.so (nm там даёт T).
ldd app_dynamic показывает, какие .so нужны программе. У нас в списке есть строка libmath_dynamic.so. Без LD_LIBRARY_PATH=. там написано not found - загрузчик не ищет библиотеки в текущей папке, а только в системных. С LD_LIBRARY_PATH=. та же строка становится ./libmath_dynamic.so - мы указали, что искать надо рядом с программой.

Когда программа запускается, загрузчик делает три вещи:

1) Смотрит в app_dynamic, какие библиотеки ему нужны (там записано имя libmath_dynamic.so).

2) Находит этот файл (по тем же путям, что и ldd).

3) Загружает его в память и подставляет в программу настоящие адреса функций dot_product и get_lib_version - чтобы вызовы из main попадали куда надо.

 в app_dynamic самих функций нет, есть только «обещание» их вызвать.
Перезапустим ldd
```
:~/lab3$ ldd app_dynamic
LD_LIBRARY_PATH=. ldd app_dynamic
        linux-vdso.so.1 (0x00007c6b24d0c000)
        libmath_dynamic.so => not found
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007c6b24a00000)
        libc.so.6 => /usr/lib/x86_64-linux-gnu/libc.so.6 (0x00007c6b24600000)
        libm.so.6 => /usr/lib/x86_64-linux-gnu/libm.so.6 (0x00007c6b248da000)
        /lib64/ld-linux-x86-64.so.2 (0x00007c6b24d0e000)
        libgcc_s.so.1 => /usr/lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007c6b24ccc000)
        linux-vdso.so.1 (0x00007a84cefa2000)
        libmath_dynamic.so => ./libmath_dynamic.so (0x00007a84cef90000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007a84cec00000)
        libc.so.6 => /usr/lib/x86_64-linux-gnu/libc.so.6 (0x00007a84ce800000)
        libm.so.6 => /usr/lib/x86_64-linux-gnu/libm.so.6 (0x00007a84ceada000)
        /lib64/ld-linux-x86-64.so.2 (0x00007a84cefa4000)
        libgcc_s.so.1 => /usr/lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007a84cef5d000)
user@Admin:~/lab3$
```
Первый блок (ldd app_dynamic): строка libmath_dynamic.so => not found. Загрузчик не ищет библиотеки в текущей папке, поэтому не находит нашу.
Второй блок (LD_LIBRARY_PATH=. ldd app_dynamic): та же строка превратилась в libmath_dynamic.so => ./libmath_dynamic.so. Теперь загрузчик знает, где искать, и находит библиотеку.
## Этап 3. Ошибка порядка компоновки
Попробуйте слинковать программу, поменяв аргументы местами (библиотека указана до объектного файла):
g++ -L. -lmath_dynamic main.o -o app_wrong_order
Соберите программу в правильном порядке:
g++ main.o -L. -lmath_dynamic -o app_correct

```
 g++ -L. -lmath_dynamic main.o -o app_wrong_order
/usr/bin/x86_64-linux-gnu-ld.bfd: main.o: in function `main':
main.cpp:(.text+0x63): undefined reference to `get_lib_version'
/usr/bin/x86_64-linux-gnu-ld.bfd: main.cpp:(.text+0xb4): undefined reference to `dot_product'
collect2: error: ld returned 1 exit status

g++ main.o -L. -lmath_dynamic -o app_correct

```

### Вопросы:
Шаг 1 упадёт с ошибкой undefined reference. Объясните, почему это происходит, используя концепцию множеств компоновщика: U (неразрешённые ссылки), D (определённые символы) и E (извлечённые файлы).

```
 LD_LIBRARY_PATH=. ./app_correct
Library version: v2.4.1-release
Dot product result: 32
user@Admin:~/lab3$
```
```
g++ -L. -lmath_dynamic main.o -o app_wrong_order

main.cpp:(.text+0x63): undefined reference to `get_lib_version'
main.cpp:(.text+0xb4): undefined reference to `dot_product'
```
Упало из-за порядка аргументов. Компоновщик читает их слева направо один раз и ведёт три множества: E - файлы, уже взятые в программу, D - символы, которые они определяют, U - символы, которые нужны, но пока не определены.

Что произошло:

Дошло до -lmath_dynamic. main.o ещё не прочитан, U пусто -> библиотеке нечего закрывать, ни один её объект в E не попадает. Библиотека пройдена и больше не рассматривается.

Дальше main.o. Он идёт в E, а dot_product и get_lib_version в U.

Аргументы кончились. U не пуст - undefined reference.

В правильном порядке (g++ main.o -L. -lmath_dynamic -o app_correct) сначала читается main.o, его неразрешённые символы попадают в U, а затем библиотека их закрывает: нужные объекты переходят в E, символы в D, U очищается.



## Этап 4. Name mangling и extern "C"
Откройте vector_math.h и закомментируйте блок #ifdef __cplusplus ... extern "C" ... #endif.
Попробуйте снова собрать программу.
### Вопросы:
Линковка упадёт с ошибкой undefined reference. Выполните nm main.o | grep dot и nm libmath_dynamic.so | grep dot. Как теперь выглядят имена функций dot_product и get_lib_version в объектном файле C++?

```
cat vector_math.h
#ifndef VECTOR_MATH_H
#define VECTOR_MATH_H

// #ifdef __cplusplus
// extern "C" {
// #endif

extern const char* get_lib_version(void);
int dot_product(const int* a, const int* b, int n);

// #ifdef __cplusplus
// }
// #endif

#endif
user@Admin:~/lab3$ g++ -c main.cpp -o main.o
user@Admin:~/lab3$ g++ main.o -L. -lmath_dynamic -o app_mangled
/usr/bin/x86_64-linux-gnu-ld.bfd: main.o: in function `main':
main.cpp:(.text+0x63): undefined reference to `get_lib_version()'
/usr/bin/x86_64-linux-gnu-ld.bfd: main.cpp:(.text+0xb4): undefined reference to `dot_product(int const*, int const*, int)'
collect2: error: ld returned 1 exit status


user@Admin:~/lab3$ nm main.o | grep dot
                 U _Z11dot_productPKiS0_i

user@Admin:~/lab3$ nm libmath_dynamic.so | grep dot
000000000000110a T dot_product
user@Admin:~/lab3$
```
В main.o: _Z11dot_productPKiS0_i. Без extern "C" компилятор с++ считает её обычной функцией и кодирует в имя типы аргументов.
В libmath_dynamic.so: символ называется просто dot_product. Библиотеку собирал компилятор с, а тут перегрузки нет, поэтому имена не меняются.
Результат: main.o ищет _Z11dot_productPKiS0_i, а в библиотеке такого символа нет, есть только dot_product. Отсюда undefined reference. dot_product(int const*, int const*, int).

в libmath_dynamic.so: dot_product и get_lib_version (обычные имена, значит, они определены в секции кода).