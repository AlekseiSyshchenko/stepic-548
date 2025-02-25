###1.2. Динамические библиотеки и линковка

####Статическое связывание объектного файла

Разобьём наше приложение на два модуля. За печать сообщения теперь будет
отвечать отдельная функция `hello_message()` и располагаться она будет
в файле [`hello.c`](hello.c), а функция `main()` из файла [`main.c`](main.c)
будет вызывать функцию `hello_message()` для вывода сообщения.

```c
#include "hello.h"
#include <stdio.h>

void hello_message(const char* name)
{
    printf("Hello %s!\n", name);
}
```

Для того, чтобы функция печати `hello_message()` была доступна в других файлах
исходного кода необходимо описать её в интерфейсном файле [`hello.h`](hello.h)

```c
#ifndef __HELLO__
#define __HELLO__

void hello_message(const char* name);

#endif
```

Команды `#ifndef`, `#define` и `#endif` нужны, чтобы при сложных зависимостях
модулей программы друг от друга не происходило многократного включения h-файла
в один и тот же c-файл.

Обратите внимание, что при подключении при помощи команды `#include` интерфейсного
файла стандартной библиотеки ввода-вывода имя файла `<stdio.h>` заключается в знаки
меньше и больше. А при подключении локального интерфейсного файла `"hello.h"` его
имя заключается в двойные кавычки.

Изменим теперь файл [`main.c`](main.c), заменив в нём вызов функции `printf()` на
вызов функции `hello_message()`

```c
#include "hello.h"

int main()
{
    hello_message("Vasya");
    return 0;
}
```
> Лектор сначала создавал пустые файлы для кода командой `touch`. Я не совсем понял
> для чего это было нужно, так как и `vim` и `nano` можно запустить с именем не
> существующего ещё файла и он будет автоматически создан в момент первого сохранения.

Соберём теперь наше приложение последовательно. Сначала выполним только компиляцию
обоих файлов исходного кода в объектные файлы. В Linux файлы объектного кода принято
обозначать расширением `.o`

```
gcc -o hello.o -c hello.c
gcc -o main.o -c main.c
```

Ключ `-c` указывает `gcc`, что ему нужно остановиться после стадии компиляции и не
нужно пытаться автоматически собрать конечное приложение.

Для того, чтобы связать _"слинковать"_ эти файлы друг с другом, а так же со стандартной
библиотекой, и создать исполнимый файл нужно указать `gcc` имя нашей программы
`-o hello` и перечислить объектные фалы, полученные на предыдущем шаге.

```
gcc -o hello main.o hello.o
```

Таким образом была получена модульная программа, но связывание модулей в этом случае
статическое и они располагаются в одном исполнимом файле программы.

####Создание динамически загружаемой библиотеки

Для того чтобы создать из нашего файла [`hello.c`](hello.c) не объектный файл, а
динамическую библиотеку достаточно изменить ключи компилятора

```
gcc -o libHello.so -shared -fPIC hello.c
```

В Linux принято именовать все динамические библиотеки так: сначала идёт приставка
`lib`, затем имя библиотеки, а потом расширение `.so`. Часто
после расширения ещё добавляется номер версии библиотеки.

Ключ `-shared` указывает `gcc` на то, что следует создавать динамическую библиотеку,
а ключ `-fPIC` даёт указание использовать при генерировании кода только команды с
относительной адресацией "Generate position-independent code (PIC)". Насколько я
понял это упрощает загрузку динамической библиотеки в произвольное место в адресном
пространстве вызывающего её процесса.
[Подробнее про -fPIC на StackOverflow](http://stackoverflow.com/questions/5311515/gcc-fpic-option)

Для того чтобы узнать, какие имена (так называемые _символы_) библиотека экспортирует,
импортирует или просто использует можно воспользоваться утилитой `nm`

```
nm libHello.so
```

Видно, что в библиотеке используются десятки символов, и среди них можно найти
и имя нашей функции `hello_message`.

Теперь необходимо пересобрать наше приложение так, чтобы оно искало функцию
`hello_message()` в динамической библиотеке `Hello`.

```
gcc main.c -fPIC -L. -lHello -o hello
```

Ключ `-lHello` указывает `gcc`, что следует подключить динамическую библиотеку
`Hello`, а ключ `-L.` то, что искать её следует в текущем каталоге.

Однако если теперь запустить наше приложение, то мы получим ошибку об отсутствии
библиотеки `libHello.so`.

В Linux за поиск и загрузку динамических библиотек отвечает специальный сервис
`ld` (динамический линковщик). И это сервис по умолчанию ищет динамические
библиотеки только в строго определенных местах файловой системы. Таким образом,
наша только что созданная библиотека libHello.so этим сервисом, ни найдена, ни
тем более загружена не будет.

Чтобы сервис `ld` смог находить и загружать нашу библиотеку можно либо
зарегистрировать ещё в системе, либо поместить её в каталог (например,
/usr/lib), который уже зарегистрирован в системе, как хранилище динамических
библиотек. Но если мы не хотим вмешиваться в настройки системы, то можно
перед запуском программы установить специальную переменную окружения
`LD_LIBRARY_PATH` в которой указать сервису `ld` на временные дополнительные
каталоги для поиска динамических библиотек.

Ещё одна переменная окружения связанная с сервисом `ld` это `LD_PRELOAD` при
помощи которой можно, например, [подменить системные библиотеки](https://habrahabr.ru/post/199090/)
которые будет использовать приложение.

Итак, установим переменную окружения `LD_LIBRARY_PATH` так, чтобы она указывала
на текущий каталог и наше приложение, наконец, заработает.

```
export LD_LIBRARY_PATH=.
./hello
```
Альтернативный вариант, чтобы не добавлять в переменную текущую директорию (из соображений безопасности):
1. Пишем в makefile для своего проекта (там где библиотека собирается) веточку post-build:
post-build:
 -mv libHello.so /path/to/your/library/folder/MyProjectLibs
2. Правим /etc/ld.so.conf, добавляя туда вышеобозначенный путь до каталога, в котором лежат наши библиотеки.
3. Делаем
#ldconfig

####Экспортируемые имена функций в программах на `C++`

В предыдущем примере и основная программа и динамическая библиотека были созданы
на языке `C`. Соглашения об именах функций в обоих частях программы совпали
и наш код заработал сам собой. Но если, наша библиотека была бы написана на `C++`,
то всё было бы уже не так просто.

Можно переименовать файл `hello.c` в `hello.cpp` и попытаться пересобрать
библиотеку Hello используя этот файл. При этом `gcc`, основываясь на расширении
файла, вызовет для него компилятор `C++`

```
mv hello.c hello.cpp
gcc -o libHello.so -shared -fPIC hello.cpp
```

Если теперь посмотреть на экспортируемые символы получившейся библиотеки, то
вместо `hello_message` мы увидим `_Z13hello_messagePKc`.

```
nm libHello.so
```

Это связано с тем, что в языке `C++` в отличие от `C` возможна перегрузка
функций. То есть могут существовать несколько функций с одним именем и разным
набором аргументов. Чтобы потом линковщик мог найти в объектном файле
каждую из этих функций на стадии компиляции происходит кодирование (mangling)
имён функций так, чтобы все они оказались уникальными. Это достигается
добавлением к имени каждой функции (даже если она и не перегружалась)
закодированной информации о типах её аргументов.

Для того, чтобы по закодированному имени функции узнать её исходный прототип
можно воспользоваться утилитой `c++filt`

```
c++filt _Z13hello_messagePKc
```

Для того, чтобы вернуть соответствие имён функций друг другу нужно в файле
[`hello.h`](hello.h) к объявлению функции `hello_message()` добавить модификатор
`extern "C"` который даст понять компилятору и линковщику, что для данной
функции следует использовать правила именования как в языке `C`.

```c
extern "C" void hello_message(const char* name);
```

Однако после такой модификации заголовочного файла он станет несовместим с
кодом написанным на `C`. То есть скомпилировать файл [`main.c`](main.c) у нас
уже не выйдет. Чтобы сделать заголовочный файл универсальным нужно подключать
различные объявления функции `hello_message()` для различных языков.

```c
#ifdef __cplusplus
extern "C" void hello_message(const char *name);
#else
void hello_message(const char *name);
#endif
```

####Базовые возможности утилиты `make`

[`make`](https://ru.wikipedia.org/wiki/Make) — утилита, автоматизирующая процесс
преобразования файлов из одной формы в другую. Чаще всего это компиляция исходного
кода в объектные файлы и последующая компоновка в исполняемые файлы или библиотеки.

Утилита использует специальные make-файлы, в которых указаны зависимости файлов
друг от друга и правила для их удовлетворения.

Каждая зависимость в make-файле выражается так: сначала пишется имя зависимого
файла или объекта, потом ставится двоеточие и пишутся имена объектов или файлов
от которых зависит объект указанный первым. Следующая строка файла должна
начаться с табуляции и содержать команду которая позволяет из файлов,
указанных в правой части создать файл указанный в левой части. Если для создания
зависимого объекта требуется несколько последовательных команд, то они могут
занимать несколько идущих подряд строк и все они должны начинаться с символа
табуляции.

Рассмотрим простейший пример файла [`Makefile`](Makefile) для сборки нашего
текущего проекта. Библиотека Hello зависит от файлов [`hello.c`](hello.c) и
[`hello.h`](hello.h) из которых она создаётся, а само приложение зависит от
файлов [`main.c`](main.c), [`hello.h`](hello.h) и библиотеки, которая к
моменту сборки приложения должна уже существовать.

```
hello: main.c hello.h libHello.so
	gcc main.c -fPIC -L. -lHello -o hello

libHello.so: hello.c hello.h
	gcc -shared hello.c -fPIC -o libHello.so
```

Здесь описаны зависимости только для наших целевых файлов. Левую часть (которая
стоит перед двоеточием) в строке описывающей зависимость часто называют целью
(target). 

Но можно иметь и цели не связанные с файлами. Например, мы хотим сделать очистку
нашего проекта от созданной программы, библиотеки и всех промежуточных файлов
образующихся при его сборке, чтобы начать всё _с чистого листа_. Для этого
можно объявить такую синтетическую цель, как `clean`. Она ни от чего не зависит,
но имеет команды для своего достижения.

```
clean:
	rm hello
	rm libHello.so
	rm *.o
```

Так как по умолчанию `make` пытается достичь первой цели среди указанных в
make-файле, то первой целью обычно назначают сборку всего приложения.

```
buildall: hello libHello.so
```

У этой цели есть зависимость, но нет команд для её достижения. Так как
достаточно создать все объекты от которых она зависит и цель будет достигнута.

Рассмотрим финальную версию нашего make-файла.

```
all: exe lib

exe: main.c hello.h lib
	gcc main.c -fPIC -L. -lHello -o hello

lib: hello.c hello.h
	gcc -shared hello.c -fPIC -o libHello.so

clean:
	-rm hello libHello.so 2>/dev/null
```

Цель `clean` ни от чего не зависит. Она лишь удаляет созданные библиотеку и приложение.
Однако если мы захотим очистить проект который и так "чист", то команда `rm` не найдёт
файлов, которые следует удалить, и выдаст сообщение об ошибке. Чтобы его подавить в
конец команды  добавлена часть `2>/dev/null` которая перенаправляет все сообщения об
ошибках в "никуда". Но кроме сообщений об ошибках команда `rm` имеет ещё и код возврата,
который анализирует утилита `make` чтобы понять была ли цель успешно достигнута.
И код возврата в случае отсутствия файлов также будет свидетельствовать об ошибке.
Для того, чтобы `make` игнорировал эту ошибку и в любом случае считал цель достигнутой
перед командой `rm` стоит знак минус.

Цели `lib` и `exe` предназначены для сборки библиотеки и приложения соответственно.
А синтетическая цель `all` которая будет достигаться по умолчанию предназначена
для создания всего приложения в целом.

Если запустить утилиту `make` без параметров, то она попытается найти в текущем
каталоге [`Makefile`](Makefile) и достичь первую из указанных там целей. Если же
мы хотим достичь какой-то другой из перечисленных в make-файле целей, то её имя
следует указать после команды `make`. Например, для очистки проекта нужно выполнить
команду:

```
make clean
```

####Определение зависимостей приложения

Для того чтобы определить от каких динамических библиотек зависит определенное
приложение или библиотека предназначена утилита `ldd`. В качестве параметра ей
передаётся имя файла зависимости которого мы хотим определить. Например:

```
ldd hello
ldd libHello.so
```
