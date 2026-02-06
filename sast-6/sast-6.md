## Часть 6. Go, SARIF и GitHub Code Scanning
Большинство анализаторов поддерживают выгрузку результатов в формате SARIF. В ходе этой части мы познакомимся с анализом проектов на языке Go, форматом SARIF и веб-интерфейсом GitHub для разметки срабатываний.

В ходе выполнения заданий 6 части у нас должны получиться следующие артефакты работы:
- скриншоты выполнения пунктов задания
- файлы gosec_results.sarif и staticcheck_results.sarif, полученные в результате анализа двух проектов
- публичные форки репозиториев с загруженными для анализа сработками

### Настройка окружения
Часть анализаторов не собрана в ветке p11, поэтому для выполнения этой части рекоменудется использовать контейнер на базе alt:sisyphus

Устанавливаем необходимые зависимости и клонируем репозиторий geoipupdate
```shell
apt-get install git gosec
git clone https://github.com/maxmind/geoipupdate
```

### Анализ с помощью GoSec, получение SARIF
У большинства анализаторов нет собственного веб-интерфейса для просмотра сработок. Обычно, все анализаторы выдают результат работы движка в формате SARIF (Static Analysis Results Interchange Format). Проанализируем код на языке Go с помощью анализатора gosec, который как раз выдает результаты в этом формате.

Выполним анализ:
```shell
gosec -fmt sarif -out ./gosec-results.sarif ./...
```

Изучим полученный `gosec-results.sarif` в любом текстовом редакторе.

В таком формате хранится вся информация о результатах анализа. Однако, чтобы провести разметку, требуется какой-либо веб-интерфейс. Попробуем загрузить полученный sarif-файл для разметки в веб-интерфейс GitHub.

### Загрузка SARIF в GitHub
GitHub бесплатно позволяет загружать результаты анализа в формате SARIF для публичных репозиториев. Для загрузки нам необходимо сделать собственный публичный форк проекта geoipuodate.

Также нам потребуется fine-grained token. Получим его в Settings -> Developer Settings -> Personal Access Tokens -> Fine-grained tokens -> Generate new token
- Имя токена: любое, например sarif-import
- Repository access: выбираем all repositories
- Add permissions: включаем code scanning alerts с read-write access

Сохраняем полученный после генерации токен.

GitHub принимает для загружки sarif-файл, закодированный в base64, сделаем это (получится довольно длинная строка)
```shell
gzip -c gosec-results.sarif | base64 -w0
```

Затем отправляем запрос с загрузкой этого sarif-файла, заполняя все поля, отмеченные `<YOUR_...>`:
```shell
curl -L \
  -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer <YOUR_TOKEN>" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/<YOUR_USERNAME>/<YOUR_REPO>/code-scanning/sarifs \
  -d '{"commit_sha":"<YOUR_LAST_COMMIT_SHA>","ref":"refs/heads/main","sarif":"<YOUR_BASE64_SARIF>"}'
```

Теперь можно посмотреть результаты анализа в репозитории во вкладке Security -> Code scanning

При проблемах с созданием токена и загрузкой можно обратиться к официальной документации GitHub:
- https://docs.github.com/en/code-security/how-tos/scan-code-for-vulnerabilities/integrate-with-existing-tools/uploading-a-sarif-file-to-github
- https://docs.github.com/en/rest/code-scanning/code-scanning?apiVersion=2022-11-28#upload-an-analysis-as-sarif-data
- https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app

### Разметка срабатываний
Выберите и разметьте с комментарием в веб-интерфейсе GitHub одно из срабатываний детектора:

- Вариант 1. Potential file inclusion via variable
- Вариант 2. Use of weak cryptographic primitive

Будьте готовы объяснить вердикт и предложения по исправлению, если оно требуется.

### Staticcheck и анализ проекта krew
Поскольку мы загружаем sarif-файлы, то можем загрузить и результат работы нескольких анализаторов. Проанализируем проект https://github.com/kubernetes-sigs/krew с использованием GoSec и Staticcheck (`apt-get install staticcheck`).

Самостоятельно определите необходимые настройки staticcheck для проведения базового анализа с выводом результатов в формате sarif.

Загрузите полученные sarif-файлы от работы GoSec и Staticcheck на проекте krew в аналогичный собственный публичный форк проекта.

Выберите и разметьте с комментарием в веб-интерфейсе GitHub одно из срабатываний детектора:
- Вариант 1. Using a deprecated function, variable, constant or field
- Вариант 2. Potential file inclusion via variable

### Итог 6 части
В ходе 6 части мы познакомились с анализом проектов на языке Go, форматом SARIF и веб-интерфейсом GitHub для разметки срабатываний.
