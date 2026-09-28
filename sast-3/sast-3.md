## Часть 3. Svace и CVE-2021-41459
[Svace](https://www.ispras.ru/technologies/svace/) - это статический анализатор, разработанный ИСП РАН. Для выполнения команд анализа (начиная с команды `svace analyze`) требуется приобретенная лицензия на использование Svace, поэтому мы будем использовать прегенерированный снепшот.
Уязвимость CVE-2021-41459 была обнаружена в [GPAC](https://github.com/gpac/gpac) - фреймворке для работы с медиаконтентом.
Обнаружить данную уязвимость с использованием рассмотренных ранее cppcheck и clang static analyzer не получится, давайте попробуем обнаружить ее с помощью Svace.

Результат сдаётся по [общим правилам](../SUBMISSION.md). В pull request должны
находиться:
- `submission/artifacts/build-info.txt` — копия `.svace-dir/bitcode/build-info.txt`;
- `submission/artifacts/poc-before.txt` и `poc-after.txt`, содержащие команду,
  stdout, stderr и код либо сигнал завершения PoC;
- `submission/artifacts/fix.patch`, созданный с помощью `git format-patch` из
  `fix_commit`;
- `submission/artifacts/screenshots/finding-before.png`, `poc-before.png` и
  `poc-after.png`, сделанные на указанных ниже этапах;
- `submission/submission.json`.

### Настройка окружения
#### Установка и настройка Svace
Скачаем и распакуем архив со статическим анализатором Svace
```shell
curl -LJ -o svace.tar.bz2 'https://nextcloud.ispras.ru/public.php/dav/files/pB8sn5a2bp6BjLP/Svace/svace-4.0.250829-x64-linux.tar.bz2'
tar -xjf svace.tar.bz2
```
Создадим симлинк в PATH, чтобы появилась возможность запускать svace без указания пути
```shell
ln -s ~/svace-4.0.250829-x64-linux/bin/svace /usr/local/bin/svace
```

#### Установка и настройка Svacer
Установим сервер для просмотра результатов разметки Svacer из предоставляемого RPM-пакета
```shell
curl -LJ -o svacer-11.2-0.x86_64.rpm 'https://nextcloud.ispras.ru/public.php/dav/files/pB8sn5a2bp6BjLP/Svace/Svacer/svacer-11.2-0.x86_64.rpm'
sudo apt-get install ./svacer-11.2-0.x86_64.rpm
```
Устанавливаем postgresql с необходимыми расширениями:
```shell
sudo apt-get install postgresql16-server postgresql16-contrib
```
Инициируем базу данных postgresql и запускаем сервис (от имени root)
```shell
/etc/init.d/postgresql initdb
systemctl start postgresql
```
Запускаем утилитку psql под именем пользователя postgres:
```shell
psql -U postgres
```
Внутри psql создаем БД для работы svacer:
```shell
 postgres=# create database svace;
 postgres=# create user svace with encrypted password 'svace';
 postgres=# grant all privileges on database svace to svace;
 postgres=# alter user svace superuser;
 postgres=# CREATE EXTENSION IF NOT EXISTS btree_gin;
```
Для выхода используем Ctrl+D

Запускаем Svacer, например, на порту 8081:
```
svacer-server run --port 8081
```
После этого оставляем запущенным процесс сервера, дальнейшую работу проводим в другом окне

#### Установка GPAC
Клонируем репозиторий GPAC
```shell
git clone <your assignment repo>
```
Перейдите в корень репозитория. Самостоятельно с помощью ресурсов с данными об уязвимости CVE-2021-41459 определите уязвимую версию и переключитесь на неё. Создайте ветку для исправления и сохраните полный SHA исходного коммита как `base_commit`:
```shell
git checkout -b sast-3-fix
git rev-parse HEAD
mkdir -p submission/artifacts/screenshots
```

### Статический анализ
```
./configure --static-mp4box --use-zlib=no
```

Инициируем директорию .svace-dir, которая будет использоваться для хранения артефактов статического анализа:
```shell
svace init
```

Запускаем сборку (`make`) под контролем Svace:
```shell
svace build make
```

Сразу после сборки сохраните сведения о её запуске из своей `.svace-dir`:
```shell
cp .svace-dir/bitcode/build-info.txt submission/artifacts/build-info.txt
```

На этом шаге у нас есть наполненная .svace-dir, но по ней не проведен анализ.
<details>
  <summary>Следующим шагом должен был быть запуск анализа Svace, но на этом шаге требуется наличие лицензии, а ее у нас нет, поэтому можем просто посмотреть на команду, а дальше будем использовать прегенерированный снепшот:</summary>

> Опция `--with-cache` замедляет анализ, но позволяет продолжить "с того же места" в случае прерывания. Можно убрать для ускорения.
> ```shell
> svace analyze --with-cache
> ```
</details>



### Разбор срабатываний


<details>
  <summary>Следующую команду мы тоже пропускаем из-за отсутсвия лицензии:</summary>

> Сначала используем внутренний сервер просмотра разметок svace
> ```shell
> svace history import
> svace server single-start
> ```
>После чего по адресу http://127.0.0.1:8060 можно будет посмотреть результаты работы анализатора
>
> Для более удобной работы используем утилиту Svacer, для выполнения команды upload требуется ранее запущенный сервер Svacer
> ```shell
> svacer import
> svacer upload --port 8081
> ```
</details>

Теперь по адресу  http://127.0.0.1:8081 можно будет подключиться к Svacer с логином и паролем admin:admin

Загрузим заранее подготовленный снепшот - gpac.snap. Загрузка снепшотов через Проекты -> Новый проект -> Импорт -> снимок (*.snap).

Самостоятельно с помощью ресурсов с данными об уязвимости CVE-2021-41459 найдите детектор, сработавший на рассматриваемую ошибку.

Назовите проект в Svacer `sast-3-<GitHub login>`, откройте целевое
срабатывание и сделайте
`submission/artifacts/screenshots/finding-before.png`. В кадре должны быть
видны имя проекта, детектор, файл, функция и ключевые шаги трассы.

Для этой уязвимости также есть PoC-файл. Проверьте существование уязвимости и сохраните результат:
```shell
.grading/run-and-log.sh submission/artifacts/poc-before.txt -- \
  ./bin/gcc/MP4Box -add poc.nhml -new new.mp4
```

Покажите сохранённый `poc-before.txt` в терминале и сделайте
`submission/artifacts/screenshots/poc-before.png`. В кадре должны быть видны
команда запуска, аварийный сигнал, origin репозитория и полный SHA
`base_commit`.

### Исправление ошибки
Самостоятельно с помощью ресурсов с данными об уязвимости CVE-2021-41459 исправьте ошибку и создайте коммит (можно использовать `git cherry-pick` с исправляющим коммитом из апстрима). Полный SHA этого коммита укажите как `fix_commit`. Пересоберите проект и повторите запуск того же PoC:
```shell
make clean
make
.grading/run-and-log.sh submission/artifacts/poc-after.txt -- \
  ./bin/gcc/MP4Box -add poc.nhml -new new.mp4
```
Убедитесь, что PoC больше не вызывает прежнее аварийное завершение.

Покажите сохранённый `poc-after.txt` в терминале и сделайте
`submission/artifacts/screenshots/poc-after.png`. В кадре должны быть видны та
же команда PoC, отсутствие аварийного сигнала, origin репозитория и полный SHA
`fix_commit`.

Без лицензии snapshot позволяет подтвердить только срабатывание на исходной версии, поэтому поле `analysis.after` в `submission.json` опускается.

### Подготовка патча
Сохраните патч из коммита с исправлением до добавления материалов сдачи:
```shell
git format-patch -1 --stdout HEAD > submission/artifacts/fix.patch
```

### Сдача задания
Заполните `submission/submission.json` по общим правилам: `assignment` — `sast-3`, `target_finding` — данные срабатывания из Svacer. В `analysis.before.artifact` укажите `submission/artifacts/screenshots/finding-before.png`; вместо команды анализа в `analysis.before.command` укажите `Импорт gpac.snap в Svacer`.

Добавьте материалы сдачи отдельным коммитом и отправьте ветку:
```shell
git add submission
git commit -m "Add SAST-3 evidence"
git push -u origin sast-3-fix
```

### Итог 3 части
В ходе 3 части мы познакомились с Svace, разобрали срабатывание на CVE-2021-41459 в GPAC по готовому снепшоту, подтвердили уязвимость с помощью PoC-файла, исправили её и проверили, что PoC больше не вызывает прежнее аварийное завершение.

Дополнительно:
- Попробуете обнаружить эту уязвимость с помощью других средств статического анализа. Предположите, почему получается/не получается.
