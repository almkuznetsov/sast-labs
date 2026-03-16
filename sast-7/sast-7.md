## Часть 7. Сканирование в CI, SourceCraft, CodeQL

Мы не хотим каждый раз запускать сканирование руками, давайте подготовим CI-workflow для анализа, который потом можно будет подключать к каждому pull request. В качестве платформы будем использовать SourceCraft (https://sourcecraft.dev)

В ходе выполнения заданий 7 части у нас должны получиться следующие артефакты работы:
- скриншоты выполнения пунктов задания
- публичные форки репозиториев в SourceCraft с загруженными для анализа сработками

### SourceCraft, CI и анализ damn-vulnerable-golang
Начнем с анализа заведомо уязвимого приложения damn-vulnerable-golang (https://github.com/TheHackerDev/damn-vulnerable-golang). Создадим в SourceCraft пустой репозиторий damn-vulnerable-golang. Добавим его url в git remote для нашего damn-vulnerable-golang (для пуша потребуется добавить в SourceCraft свой SSH-ключ):
```shell
git remote add sc ssh://ssh.sourcecraft.dev/<USERNAME>/damn-vulnerable-golang.git
```
Основываясь на [примере](https://sourcecraft.dev/examples/appsec-analysis) cоздадим файл `./sourcecraft/ci.yaml` с описанием workflow, в котором выполняются уже сделанные нами команды, а также дополнительный шаг валидации и загрузки полученного SARIF в SourceCraft:
```shell
workflows:
  security-pipeline:
    tasks:
      - name: gosec-security-scan
        cubes:
          # Step 1: Run gosec scanner
          - name: gosec-scan
            image:
              name: alt:p11
            script:
              - <your gosec launch code here>

          # Step 2: Optional - Debug/validate results
          - name: validate-scan-results
            script:
              - cat $SOURCECRAFT_WORKSPACE/result.sarif

          # Step 3: Upload results to SourceCraft
          - name: upload-sarif-to-sourcecraft
            image:
              name: sourcecraft/scan-result-uploader:0.6.0
            script:
              - export APPSEC_CUSTOM_ENGINE_NAME="gosec"
              - /app/bin/scan-result-uploader
```

Запушим этот файл в SourceCraft, чтобы применились настройки
```shell
git add .sourcecraft
git push sc
```
Теперь во вкладке CI/CD можно запустить новый пайплайн. Запустим наш пайплайн не меняя другие настройки и посмотрим на результаты. Здесь будет удобно обсуждать сработки и размечать их.

Ориентируясь на [пример](https://sourcecraft.dev/examples/appsec-analysis) добавьте в `ci.yaml` условия, чтобы наш новый workflow анализа запускался для каждого Pull Request в ветку main. Сделайте собственный Pull Request, содержащий любую новую ошибку (проверить, найдет ли её gosec, всегда можно локально) и дождитесь прохождения проверки - вы должны получить комментарий от SourceCraft Security Bot о новой сработке.

### CodeQL и анализ geoipupdate
Проведем анализ проекта geoipupdate (https://github.com/maxmind/geoipupdate) версии 7.0.0 (`git checkout v7.0.0`) с помощью CodeQL и посмотрим на результаты.

Локально проверить работу CodeQL можно так:
```shell
apt-get update -qq && apt-get install -y wget golang glibc
export PATH="$PATH:$(go env GOPATH)/bin"
wget https://github.com/github/codeql-action/releases/latest/download/codeql-bundle-linux64.tar.gz
tar -xzf codeql-bundle-linux64.tar.gz -C /opt
ln -sf /opt/codeql/codeql /usr/local/bin/codeql

codeql database create /tmp/codeql-db --language=go
codeql database analyze /tmp/codeql-db <your commands here>
```

Аналогичным образом создаем пустой репозиторий SourceCraft, клонируем geoipupdate и пушим с добавлением `.sourcecraft/ci.yaml` с workflow запуска CodeQL.

Запустите анализ на версии 7.0.0 и разметьте две **разные** сработки, полученные в результате анализа.

### Итог 7 части
В ходе 7 части мы познакомились с новым анализатором CodeQL, платформой SourceCraft и запуском сканирования в CI.
