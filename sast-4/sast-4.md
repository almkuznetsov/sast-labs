## Часть 4. compile_commands.json, CodeChecker и оркестрация анализаторов
Современный статический анализ добавляет еще несколько инструментов и возможностей поверх рассмотренных ранее. В ходе этой части мы посмотрим на то, как реализовать оркестрацию нескольких статических анализаторов и на то, как выглядят современные open-source инструменты для разметки полученных срабатываний.

Результат сдаётся по [общим правилам](../SUBMISSION.md). В PR должны находиться:
- `submission/artifacts/<project>/compile_commands.json` и `cc_out/` для каждого проекта (`pugixml` и `sqlsmith`);
- `submission/artifacts/screenshots/pugixml-finding.png` и `sqlsmith-finding.png` с разметкой срабатываний;
- `submission/submission.json`.

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
mkdir -p submission/artifacts
CodeChecker analyzers --details | tee submission/artifacts/analyzers.txt
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
Выберите и разметьте с комментарием в веб-интерфейсе CodeChecker любое из срабатываний.

Будьте готовы объяснить вердикт и предложения по исправлению, если оно требуется.

Сделайте `submission/artifacts/screenshots/pugixml-finding.png`: в кадре должны быть видны имя проекта, детектор, файл и строка, вердикт и комментарий.

### Анализ проекта sqlsmith
Выполните аналогичный анализ проекта https://github.com/anse1/sqlsmith, используя анализаторы clangsa, clang-tidy, cppcheck

Загрузите результат на веб-сервер, выберите и разметьте с комментарием в web-интерфейсе CodeChecker любое из срабатываний.

Аналогично сохраните `submission/artifacts/screenshots/sqlsmith-finding.png` с разметкой выбранного срабатывания.

### Сдача задания
Скопируйте `compile_commands.json` и весь каталог `cc_out` каждого проекта в `submission/artifacts/<project>/` в выданном репозитории. Скриншоты также поместите в его `submission/artifacts/`.

Заполните `submission/submission.json`: `schema_version` — `1`, `assignment` — `sast-5`, `environment` — по общим правилам (анализатор — CodeChecker). В массиве `projects` укажите по объекту для `pugixml` и `sqlsmith`:
- `name`, `repository` (URL исходного репозитория), `commit` (полный SHA проанализированного коммита);
- `compilation_database` — путь к сохранённому `compile_commands.json`;
- `analysis.command` и `analysis.artifact` — точную команду анализа и путь к сохранённому `cc_out`;
- `target_finding` с полями `detector`, `path`, `line`, `verdict`, `comment` — выбранное срабатывание и его разметку.

Пути к артефактам указывайте от корня выданного репозитория, команду анализа — от корня соответствующего проекта.

### Итог 4 части
В ходе 4 части мы познакомились с CodeChecker, compile_commands.json и возможностями оркестрации нескольких статических анализаторов, а также провели разметку нескольких их сработок.

Дополнительно:
- Попробуете подключить к анализу gcc-analyzer.
