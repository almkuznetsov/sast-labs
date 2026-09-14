## Часть 2. Clang static analyzer и CVE-2014-8130
[Clang static analyzer](https://clang-analyzer.llvm.org/) - это статический анализатор на базе компилятора clang, который, в свою очередь, входит в проект LLVM.
Уязвимость CVE-2014-8130 была обнаружена в библиотеке для работы с TIFF-файлами [libtiff](https://gitlab.com/libtiff/libtiff).
Давайте попробуем обнаружить и исправить эту уязвимость, опираясь на статический анализ с помощью clang static analyzer.

Результат сдаётся по [общим правилам](../SUBMISSION.md). В PR должны
находиться:
- `submission/artifacts/scan-before/` с результатами scan-build для
  `base_commit`;
- `submission/artifacts/scan-after/` с результатами scan-build для
  `fix_commit`;
- `submission/artifacts/poc-before.txt` и `poc-after.txt`, содержащие команду,
  stdout, stderr и код либо сигнал завершения PoC;
- `submission/artifacts/fix.patch`, созданный с помощью `git format-patch` из
  `fix_commit`;
- `submission/artifacts/screenshots/finding-before.png`, `poc-before.png`,
  `analysis-after.png` и `poc-after.png`, сделанные на указанных ниже этапах;
- `submission/submission.json`.

### Настройка окружения
Поскольку clang static analyzer построен на базе компилятора clang, он позволяет проводить более глубокий анализ программы путем участия в ее сборке. Следовательно, для такого анализа нам потребуется собрать библиотеку libtiff, а значит нам потребуется установить ее зависимости:
```shell
apt-get install make libSM-devel libXi-devel libXmu-devel libfreeglut-devel libjpeg-devel liblzma-devel libwebp-devel libzstd-devel zlib-devel libdeflate-devel
```
 Также установим LLVM и сам clang static analyzer
```shell
apt-get install clang19.1-analyzer gcc-c++ llvm19.1
```
Склонируем себе репозиторий с проектом libtiff и перейдем в его директорию
```shell
git clone (your assignment repo link)
cd (your assignemnt repo)
```

### Статический анализ
В новой версии libtiff рассматриваемая нами уязвимость исправлена. Чтобы проанализировать код, где она еще не была исправлена, откатимся к версии 4.0.3
```shell
git checkout v4.0.3-branch
git checkout -b v4.0.3-branch-fix
git rev-parse v4.0.3-branch
```
Полный SHA из последней команды укажите как `base_commit`.
Чтобы провести статический анализ с использованием clang static analyzer, можно использовать утилиту scan-build, которая проведет сборку нашего проекта с необходимыми для статического анализа настройками компилятора. Поскольку наш проект собирается с помощью скрипта `configure` и утилиты `make`, то для сборки со статическим анализом используем следующие команды
```shell
mkdir -p submission/artifacts
scan-build ./configure
scan-build -o submission/artifacts/scan-before make -j4
```
Здесь ключ `-j4` запускает сборку в несколько потоков, что позволяет ускорить этот процесс. Ключ `-o` сохраняет отчет в каталоге сдачи.

### Разбор срабатываний
После завершения работы утилиты scan-build будет выведено сообщение вида
```shell
scan-build: Run 'scan-view submission/artifacts/scan-before/[..]' to examine bug reports.
```
Запустив указанную команду мы сможем ознакомиться с результатами статического анализа в любом браузере по указанному в выводе команды адресу.

Нас будет интересовать первое срабатывание детектора `Division by zero` в файле `libtiff/tif_write.c:119`.
Как видим, `td->td_stripsperimage` может быть равным нулю, в результате чего может возникнуть деление на ноль.

В интерфейсе scan-view откройте трассу целевого срабатывания и сделайте
`submission/artifacts/screenshots/finding-before.png`. В кадре должны быть
видны тип ошибки, `libtiff/tif_write.c`, строка с делением.

Для этой уязвимости у нас есть proof of conecpt (PoC) файл - это файл, позволяющий продемонстрировать существование уязвимости. Давайте остановим работу утилиты scan-view (Ctrl+C) и попробуем воспроизвести эксплуатацию уязвимости, для этого подадим на вход собранной нами утилите tiffdither специально сформированный tiff файл, который приведет к рассматриваемому нами делению на ноль. Предварительно любым удобным способом склонируйте PoC файл из репозитория с заданиями.
```shell
.grading/run-and-log.sh submission/artifacts/poc-before.txt -- \
  ./tools/tiffdither <path/to/poc/15_tiffdither.tiff> out.tiff
```
Мы должны увидеть сообщение вида `floating point exception`, свидетельствующего о том, что выполнение программы было прервано указанным исключением.

Покажите сохранённый `poc-before.txt` в терминале и сделайте
`submission/artifacts/screenshots/poc-before.png`. В кадре должны быть видны
команда запуска и аварийный сигнал.

### Исправление ошибки
Для исправления ошибки можно добавить проверку, равен ли `td->td_stripsperimage` нулю непосредственно перед проведением деления. Проверка может выглядеть следующим образом
```c
if (td->td_stripsperimage == 0) {
	return (-1);
}
```

Самостоятельно проведите указанное или подобное ему исправление, внесите его в систему контроля версий с помощью команд `git add` и `git commit`, а затем повторно запустите статический анализ (для этого потребуется сначала удалить результаты предыдущей сборки с помощью `make clean`). Сохраняйте результаты двух запусков scan-build в указанные выше разные каталоги. Убедитесь, что срабатывание `Division by zero` в `libtiff/tif_write.c` присутствует до исправления и отсутствует после него. Отключение детектора или исключение файла из анализа не считается исправлением.

Полный SHA коммита с исправлением укажите как `fix_commit`. Повторную проверку
сохраните командами:
```shell
make clean
scan-build -o submission/artifacts/scan-after make -j4
.grading/run-and-log.sh submission/artifacts/poc-after.txt -- \
  ./tools/tiffdither <path/to/poc/15_tiffdither.tiff> out.tiff
```

После исправления сделайте ещё два снимка:

- `submission/artifacts/screenshots/analysis-after.png` — результат повторного
  scan-build без целевого `Division by zero`, origin и полный SHA `fix_commit`;
- `submission/artifacts/screenshots/poc-after.png` — содержимое
  `poc-after.txt` с той же командой PoC и без аварийного сигнала.

### Подготовка патча
Аналогично первой части необходимо подготовить патч с помощью `git format-patch -1 --stdout HEAD > submission/artifacts/fix.patch`.

### Сдача задания
Скопируйте пример, заполните `submission.json`, добавьте материалы сдачи
отдельным коммитом и отправьте ветку:
```shell
cp .grading/examples/submission.json submission/submission.json
# заполняем submission.json
git add submission
git commit -m "Add SAST-2 evidence"
git push -u origin v4.0.3-branch-fix
```
Создайте PR и добавьте `@almkuznetsov` в reviewers.

### Итог 2 части
В ходе 2 части мы познакомились с clang static analyzer, который позволяет проводить более глубокий анализ за счет участия в сборке программы. Проанализировали уязвимую к CVE-2014-8130 версию библиотеки libtiff, обнаружили данную уязвимость посредством статического анализа, проверили ее наличие с помощью PoC-файла, исправили ее и убедились в отсутствии повторного срабатывания статического анализатора.

Дополнительно:
- Сравните ваше исправление с коммитом, который действительно использовался для исправления этой уязвимости (информацию об уязвимости удобно смотреть по ее идентификатору на https://CVE.org).
