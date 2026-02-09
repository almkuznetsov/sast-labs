## Часть 5. compile_commands.json, CodeChecker и оркестрация анализаторов
Современный статический анализ добавляет еще несколько инструментов и возможностей поверх рассмотренных ранее. В ходе этой части мы посмотрим на то, как реализовать оркестрацию нескольких статических анализаторов и на то, как выглядят современные open-source инструменты для разметки полученных срабатываний.

В ходе выполнения заданий 5 части у нас должны получиться следующие артефакты работы:
- скриншоты выполнения пунктов задания
- файлы compile_commands.json полученные в результате анализа двух проектов
- директории cc_out с результатами работы CodeChecker для двух проектов

### Настройка окружения
В отдельном контейнере запустим веб-сервер CodeChecker (он потребуется нам позже)
```shell
podman run -d -p 8001:8001 codechecker/codechecker-web:latest
```

Настроим окружение для работы в контейнере на базе alt:p11 (обязательно потребуется --network host)

Устанавливаем необходимые зависимости и CodeChecker
```shell
apt-get install make llvm20.1 clang20.1-analyzer clang20.1-tools cppcheck git cmake gcc-c++ pip

export PATH="/usr/lib/llvm-20.1/bin:$PATH"

pip install codechecker
```

Проверяем, что в CodeChecker доступны анализаторы clangsa, clang-tidy, cppcheck с указанием версий:
```shell
CodeChecker analyzers --details
```

Склонируем репозиторий pugixml
```shell
git clone https://github.com/zeux/pugixml
```

### Сбор compile_commands.json
Существует большое количество анализаторов, имеющих возможность перехвата сборки. Часто нам требуется провести анализ не одним, а несколькими анализаторами. В таком случае обычно требуется запустить сборку несколько раз. При разработке на C/C++ эту задачу можно решить, один раз записав во время сборки команды компиляции, которые затем будут передаваться анализаторам без необходимости каждый раз заново перезапускать сборку.

Выполним cmake:
```shell
cmake .
```

Чтобы записать команды компиляции воспользуемся механизмами CodeChecker:
```shell
CodeChecker log -o compile_commands.json -b "make -j8"
```

Посмотрим, какие команды нам удалось перехватить:
```shell
cat compile_commands.json
```

### Оркестрация статических анализаторов, веб-сервер CodeChecker
Теперь, имея файл compile_commands.json, мы можем с помощью оркестратора CodeChecker запустить анализ сразу несколькими анализаторами (в нашем случае clangsa и clang-tidy):
```shell
CodeChecker analyze compile_commands.json --analyzers clangsa clang-tidy -j8 -o cc_out
```

Посмотрим на результат:
```shell
CodeChecker parse ./cc_out
```

Загрузим результаты на запущенный ранее веб-сервер CodeChecker
```shell
CodeChecker store --name pugixml --url localhost:8001/Default ./cc_out
```

### Разметка срабатываний
Выберите и разметьте с комментарием в веб-интерфейсе CodeChecker одно из срабатываний детектора:

- Вариант 1. bugprone-inc-dec-in-conditions
- Вариант 2. bugprone-signed-char-misuse

Будьте готовы объяснить вердикт и предложения по исправлению, если оно требуется.

### Анализ проекта sqlsmith
Выполните аналогичный анализ проекта https://github.com/anse1/sqlsmith, используя анализаторы clangsa, clang-tidy, cppcheck

Загрузите результат на веб-сервер, выберите и разметьте с комментарием в web-интерфейсе CodeChecker одно из срабатываний детектора:
- Вариант 1. cppcheck-nullPointerRedundantCheck
- Вариант 2. cppcheck-assertWithSideEffect

### Итог 5 части
В ходе 5 части мы познакомились с CodeChecker, compile_commands.json и возможностями оркестрации нескольких статических анализаторов, а также провели разметку нескольких их сработок.

Дополнительно:
- Попробуете подключить к анализу gcc-analyzer.
