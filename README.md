# Android-конвейер ShapitoGram

Собирает подписанный release-APK **ShapitoGram** (`com.shapitogram.messenger`) на
GitHub Actions. Результат — тот же клиент, что и XTree Gram APK, но под нашим
именем, с нашим сервером и с выключенными платежами.

## Как собрать

Три команды. Токен — classic PAT со scope `repo` и `workflow`, положенный в
`.gh_token` рядом с этим файлом (или в `$GH_TOKEN`).

```bash
PY=C:/Users/rgali/.workbuddy-ai/binaries/python/envs/default/Scripts/python.exe

# 1. Проверить всё локально — 5 секунд вместо 60 минут прогона
$PY tools/check_pipeline.py

# 2. Положить ключ подписи и пароль в секреты репозитория
$PY tools/run_build.py secrets --keystore ../artifacts/release.keystore

# 3. Залить конвейер, создать репозиторий и запустить сборку
$PY tools/run_build.py dispatch --create --public
$PY tools/run_build.py watch
$PY tools/run_build.py download
```

Сборка идёт 40–70 минут: ~10 минут на образ с Android SDK и NDK, остальное —
нативная часть `tmessages.49` под четыре ABI.

Если прогон упал:

```bash
$PY tools/run_build.py status      # на каком шаге
$PY tools/run_build.py logs        # хвост лога (доступен после завершения)
```

## Почему overlay, а не патч

iOS-конвейер состоит из патча и скриптов: там upstream — это десятки тысяч
файлов, и держать их копию бессмысленно. С Android так не получается.

Кастомизация XTree Gram в `E:\xtreegram-build\android` — это **205 изменённых
файлов** поверх `origin/main`, из которых **145 — бинарные иконки** (`webp`,
`png`). Патч на такое дерево хрупок: контексты, fuzz, бинарные хунки. Overlay —
это просто «положить файлы поверх свежего клона», и результат детерминирован.

Поэтому в репозитории лежат не патчи, а сами файлы — `files/` (9.65 МБ), а
upstream клонируется на раннере из закреплённого коммита:

```
UPSTREAM_REPO: https://github.com/owpengram/owpengram-android-client.git
UPSTREAM_REF:  4adc49b29dcfe8117cfd804131e403021df7c09a
```

Смена `UPSTREAM_REF` означает, что overlay надо пересобрать и перепроверить:
файлы написаны ровно под этот коммит.

### Про 866 «изменённых» файлов

`git status` в дереве XTree Gram показывает 866 изменённых файлов, но 634 из
них — автогенерённые `.kt` в `TMessagesProj_AppTests`, у которых нет ни одного
реального изменения: в репозитории включён `core.autocrlf=true`, и они
«изменены» только по переводам строк. `git diff` их не видит, поэтому
`tools/make_overlay.py` берёт именно `git diff --name-only` — 205.

### Одно отступление от «байт в байт»

`gradle.properties` попадает в overlay **с вычищенными** `RELEASE_KEY_ALIAS`,
`RELEASE_STORE_PASSWORD` и `RELEASE_KEY_PASSWORD`: в дереве XTree Gram там лежит
рабочий пароль ключа подписи, а overlay уезжает в публичный репозиторий. На
сборку это не влияет — значения всё равно перезаписываются на шаге ребрендинга.
Остальные 204 файла копируются байт в байт.

## Что правит ребрендинг

Один шаг (`scripts/rebrand_android.py`) делает всё, и в конце сам ищет остатки
XTree Gram в дереве — падает, если что-то нашлось.

| Что | Откуда берётся | Почему это важно |
|---|---|---|
| `APP_PACKAGE` | `brand.json` | `defaultConfig.applicationId = APP_PACKAGE`; разный пакет — ShapitoGram и XTree Gram стоят рядом, а не обновляют друг друга |
| `AppName` / `AppNameBeta` | `brand.json` | имя под иконкой, все 10 локалей |
| текст в `values*/strings.xml` | правила из `brand.json` | **все** локали, включая те, которых нет в overlay |
| следы XTree Gram в Java | правила из `brand.json` | 66 вхождений в 24 файлах: комментарии и текстовые литералы |
| `DEFAULT_HOST` / `DEFAULT_PORT` | `CUSTOM_HOST` / `CUSTOM_PORT` | в overlay стоит `2.27.200.203` — адрес сервера XTree Gram |
| `OWPENGRAM_RSA_KEY` | `SERVER_INFO_URL` | ключ сервера XTree Gram не подойдёт нашему |
| `RELEASE_KEY_*` | секреты GitHub | ключ подписи ShapitoGram |

### Две группы правил замены

Правил две, и это не дублирование.

Overlay — это файлы **XTree Gram**, то есть в тексте уже стоит `Xtreegram`
вместо `Telegram`: 1080 вхождений в 10 локалях `strings.xml` и 66 в Java. Правило
`Telegram -> ShapitoGram` такие строки не ловит — в слове `Xtreegram` нет
подстроки `Telegram` (там `treegram`). Поэтому сначала снимаются следы XTree
Gram, потом добирается то, что XTree Gram сам не тронул.

В Java применяется **только** группа XTree Gram. Правило `Telegram -> ShapitoGram`
там опасно: слово встречается в именах (`TelegramRsaKey`), в путях
(`org/telegram/...`) и в ссылках (`t.me/Telegram`). Правила XTree Gram такого
риска не несут — проверено по overlay: `Xtreegram` в Java ни разу не стоит в
идентификаторе. Единственные `Xtreegram`-в-идентификаторе — это `APP_PACKAGE` и
`RELEASE_KEY_*` в `gradle.properties`, и они правятся отдельными регекспами.

## Платежи выключены

Требование: в APK не должно быть оплаты. В overlay это
`TMessagesProj/src/main/java/org/telegram/ui/PaymentFormActivity.java`:

```java
// ShapitoGram: платежи отключены. Не final — иначе javac свернёт условие
// в константу и объявит код после return недостижимым (ошибка компиляции).
private static boolean XTREE_PAYMENTS_DISABLED = true;

@Override
public boolean onFragmentCreate() {
    if (XTREE_PAYMENTS_DISABLED) {
        return false;   // фрагмент не попадёт в стек, экран оплаты не откроется
    }
    ...
```

Проверяется дважды:

1. **Шаг workflow** «Verify payments are disabled» — по исходнику, который уйдёт
   в сборку. Это единственная надёжная проверка: в готовом APK имя
   `XTREE_PAYMENTS_DISABLED` может быть переименовано R8.
2. **`tools/make_overlay.py`** — отказывается собирать overlay, если правки нет.

## Секреты

Ключ подписи и пароль в репозитории не лежат: репозиторий публичный, а ключ
подписи — это право выпускать обновления от имени приложения. Они живут в GitHub
Secrets, шифруются на клиенте открытым ключом репозитория и попадают в лог уже
замаскированными.

| Секрет | Откуда | Обязателен |
|---|---|---|
| `ANDROID_KEYSTORE_BASE64` | `../artifacts/release.keystore` | да |
| `ANDROID_KEYSTORE_PASSWORD` | `../artifacts/KEYSTORE_INFO.txt` | да |
| `ANDROID_KEY_ALIAS` | `shapitogramkey` | да |
| `TELEGRAM_API_ID` / `TELEGRAM_API_HASH` | `my.telegram.org` | нет |

`tools/run_build.py secrets` ставит первые три сам, читая пароль из
`KEYSTORE_INFO.txt`. Без `TELEGRAM_API_*` шаг `Set api_id / api_hash` оставляет
учётные данные upstream (`3893395`, они рабочие и зарегистрированы) и говорит об
этом в логе. Своё приложение заводится на <https://my.telegram.org> — тогда
`--api-id` и `--api-hash` добавятся к той же команде.

## Проверки на выходе

Ничего не отдаётся «просто потому что собралось». Три уровня:

**1. Исходник** — `rebrand_android.py --check` ищет следы XTree Gram в дереве.

**2. Структура APK** (`scripts/verify_apk.sh`, внутри образа, через `aapt2` и
`apksigner`):

- имя пакета = `com.shapitogram.messenger` — иначе APK ставится как чужое приложение;
- `versionCode` = `APP_VERSION_CODE * 10 + 9` (`afat` даёт `abiVersionCode = 9`),
  то есть `70429`;
- подпись сходится **и** отпечаток сертификата совпадает с
  `EXPECTED_CERT_SHA256` — иначе обновление поверх установленного не встанет
  («App not installed»);
- схема подписи **v2** на месте. Её отсутствие — почти всегда признак того, что
  `zipalign` выполнили после `apksigner`; Android 11+ такой APK не поставит;
- есть хотя бы один `native-code` — иначе APK поставится и не запустится.

**3. Содержимое APK** (`tools/scan_apk.py`, на раннере). APK — это zip, и строки
в нём лежат в `resources.arsc` (UTF-8 или UTF-16LE) и `classes*.dex` (MUTF-8),
поэтому каждый файл архива ищется в обеих кодировках сразу. Проверяется, что:

- **нет** `Xtreegram`, `XTree Gram`, `XTREEGRAM`, адреса `2.27.200.203`,
  `XtreegramStore2026`, `xtreegramkey`;
- **есть** `shapitogram.xyz`, `com.shapitogram.messenger` и **тело нашего
  RSA-ключа**.

Последнее — самая ценная проверка: она ловит случай «собралось, поставилось,
выглядит правильно, а клиент уходит на сервер XTree Gram», который иначе
проявляется только вечным сплэшем на телефоне.

## Файлы

| Файл | Что делает |
|---|---|
| `.github/workflows/build-apk.yml` | сам конвейер: клон, overlay, ребрендинг, Docker, сборка, проверки, артефакт |
| `brand.json` | единый конфиг бренда (общий с iOS и сервером) |
| `files/` | overlay, 205 файлов XTree Gram |
| `push_to_github.py` | заливка конвейера одним атомарным коммитом через Git Data API |
| `tools/make_overlay.py` | собирает `files/` из дерева XTree Gram, вычищая секреты |
| `tools/run_build.py` | драйвер: `check / secrets / dispatch / status / watch / logs / download` |
| `tools/check_pipeline.py` | локальные проверки до заливки |
| `tools/set_secrets.py` | кладёт keystore и пароль в GitHub Secrets |
| `tools/scan_apk.py` | ищет следы XTree Gram в готовом APK |
| `scripts/rebrand_android.py` | весь ребрендинг одним шагом |
| `scripts/verify_apk.sh` | структурная проверка APK (`aapt2` + `apksigner`) |
| `scripts/fetch_server_key.py` | качает RSA-ключ сервера из `/owpengram/server-info` |
| `scripts/set_api_credentials.py` | вписывает `api_id`/`api_hash` в `BuildVars.java` |

`push_to_github.py`, `tools/run_build.py` и `tools/set_secrets.py` в репозиторий
сборки **не заливаются** (см. `EXCLUDE_PATHS`): в первом лежит токен, остальные
нужны только локально. Вместе с ними уезжает генерируемый `.gitattributes` с
`* -text` — чтобы git на раннере не перевёл ни один файл в CRLF.

## Что уже учтено (и почему именно так)

- **Освобождение диска — условное.** На нынешнем `ubuntu-latest` под `/` 145 ГБ
  и 86 ГБ свободны сразу, а нужно ~20 ГБ (образ + gradle с NDK). Безусловное
  `rm -rf /usr/local/lib/android` стоило ~6 минут — это 700 тысяч мелких файлов,
  дольше, чем весь остальной подготовительный этап. Поэтому шаг сначала смотрит
  на `df` и чистит только если свободно меньше 40 ГБ: раннеры бывают и теснее.
- **Базовый образ тянется с зеркала, а не из Docker Hub.** Docker Hub отдаёт
  анонимным клиентам `429 Too Many Requests` (100 pull на 6 часов на IP, а IP
  раннеров GitHub общие), и первый прогон упал именно на этом. Поэтому шаг
  `Pull the base image` идёт по списку зеркал — сначала
  `public.ecr.aws/docker/library/gradle:8.11.1-jdk17` (официальное зеркало
  образов Docker Hub от AWS, без анонимных лимитов), потом Docker Hub, — и
  **переписывает строку `FROM` в Dockerfile** на то зеркало, с которого образ
  реально скачался. Без этой правки `docker build` всё равно пошёл бы в Docker
  Hub за тем же 429, даже имея образ локально. `--pull=false` там же.
  (`mirror.gcr.io` не подошёл: этого тега у него нет, 404. `docker.m.daocloud.io`
  отдаёт 403.)
- **Сборка в Docker, а не через `setup-java` + `android-actions/setup-android`.**
  Образ из репозитория фиксирует ровно тот набор версий, которым собирался
  XTree Gram: gradle 8.11.1, SDK 36, build-tools 36.0.0, NDK 27.2.12479018,
  cmake 3.22.1. Сборка вне образа — это другой тулчейн, то есть другая сборка.
- **Слои образа не кэшируются.** Образ ~10 ГБ, в лимит кэша GitHub Actions
  (10 ГБ на репозиторий) он не влезает вместе с кэшем исходников, а установка
  SDK занимает минуты, тогда как сборка — десятки минут.
- **Кэш исходников отделён от рабочей копии** (`SOURCE_DIR` и `BUILD_DIR`).
  Кэш обязан оставаться чистым, иначе ребрендинг предыдущего прогона переживёт
  следующий. Учти: `actions/cache` **не сохраняет** кэш, если job упал, — первый
  успешный прогон кэширует дерево, до него клон идёт с нуля (1–2 минуты).
- **Клон по SHA, а не по ветке.** `git fetch --depth 1 origin <SHA>` на GitHub
  работает (проверено), поэтому тянется ровно один коммит, а не 281 МБ истории.
- **Сабмодули: сначала `--depth 1`, потом полный клон.** Зафиксированные
  коммиты сабмодулей не всегда вершины веток, и тогда сервер отказывается
  отдавать объект по SHA. Мелкий клон пробуется первым, потому что обычно
  работает и экономит десяток минут.
- **`--user root` в `docker run`.** Смонтированные каталоги принадлежат раннеру
  (uid 1001), а в образе пользователь `gradle` (uid 1000) и писать в них не может.
- **`GRADLE_USER_HOME` вынесен на раннер и кэшируется.** Раннер каждый раз
  новый, поэтому без `actions/cache` этот каталог жил бы ровно один прогон, а
  зависимости Telegram Android — это ~1 ГБ загрузок. Кэш отдельный от кэша
  исходников, с `restore-keys` без ref — чтобы частично восстановиться даже
  после смены `UPSTREAM_REF`.
- **Ссылка на лог задания требует снять `Authorization` на редиректе.** GitHub
  отдаёт 302 на подписанный S3-URL; если уйти туда с заголовком, подпись не
  сходится и приходит `403 Server failed to authenticate the request`. Это
  учтено в `tools/run_build.py logs`.
- **Пути задаются шагом через `$GITHUB_ENV`, а не в `env` job'а.** Контекст
  `runner` в `env` на уровне job GitHub не принимает — dispatch отвечает
  `422 failed to parse workflow: Unrecognized named-value: 'runner'`, и сборка
  не стартует вообще. `check_pipeline.py` теперь ловит это локально.

## Что осталось за пределами этого конвейера

- **Выдача APK пользователям.** Собранный файл лежит в артефактах запуска
  (и в `artifacts/` после `download`). Публикация на сайт — отдельный шаг.
- **Обновление `UPSTREAM_REF`.** Когда форк `owpengram-android-client` получит
  новый коммит, overlay надо перепроверить: файлы написаны под `4adc49b2`.
- **Свой `api_id`.** Пока используется upstream-значение `3893395`.
- **Firebase / push.** В форке Firebase вырезан (`google-services.json` не
  нужен), push работает через keep-alive. Это свойство overlay, не конвейера.
