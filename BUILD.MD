# Сборка корпоративного VSIX расширения ESLint

Инструкция по сборке расширения vscode-eslint в пакет `.vsix` на чистой виртуальной машине macOS.

Итог сборки: `vscode-eslint-<version>.vsix` (~1–2 МБ). Все зависимости бандлятся внутрь пакета (webpack), `node_modules` в VSIX не попадает.

## 1. Предпосылки

### Xcode Command Line Tools (нужен git)

```bash
xcode-select --install
```

### Node.js 20 LTS

CI проекта использует Node 20.14.0 — рекомендуется та же мажорная версия. Установка одним из способов:

**Вариант А — официальный установщик:**

Скачать macOS Installer (.pkg) для Node 20.x LTS с <https://nodejs.org> и установить.

**Вариант Б — через nvm (без sudo):**

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# перезапустить терминал или source профиля
nvm install 20
nvm use 20
```

Проверка:

```bash
node --version   # ожидается v20.x
npm --version
```

### Доступ к npm registry

Если ВМ за корпоративным прокси или используется внутреннее зеркало:

```bash
npm config set registry https://<внутреннее-зеркало-npm>   # при необходимости
npm config set proxy http://<proxy>:<port>                 # при необходимости
npm config set https-proxy http://<proxy>:<port>           # при необходимости
```

**Важно:** параметр `ignore-scripts` должен быть выключен (`npm config get ignore-scripts` → `false`), иначе не отработает `postinstall`, который устанавливает зависимости в `client/` и `server/`.

## 2. Получение исходников

### Вариант А — git clone (предпочтительно)

```bash
git clone <url-внутреннего-зеркала> vscode-eslint
cd vscode-eslint
```

### Вариант Б — копирование архива

В репозитории папка `$shared` связана симлинками с `client/src/shared` и `server/src/shared`. Копировать нужно с сохранением симлинков — используйте `tar`, `rsync -a` или `ditto`. Не используйте инструменты, которые разворачивают симлинки в копии файлов (сборка сломается или задублирует код).

```bash
tar -xf vscode-eslint.tar -C <целевая-папка>
```

Проверка после получения исходников (симлинки на месте):

```bash
ls -la client/src/shared server/src/shared
# ожидается: client/src/shared -> ../../$shared/
```

Восстановление при необходимости:

```bash
ln -sfn '../../$shared' client/src/shared
ln -sfn '../../$shared' server/src/shared
```

## 3. Установка зависимостей

Выполняется один раз из корня репозитория:

```bash
npm ci
```

`postinstall` автоматически выполнит установку зависимостей в `client/` и `server/` (скрипт `build/bin/all.js`). Требуется сеть.

## 4. Сборка VSIX

```bash
npm install -g @vscode/vsce   # или: npx @vscode/vsce package — тогда установка не нужна
```

Упаковка:

```bash
vsce package
# или без глобальной установки:
npx @vscode/vsce package
```

`vsce package` автоматически запускает хук `vscode:prepublish` → `npm run webpack`, который:
1. Очищает `client/out` и `server/out`
2. Собирает бандлы: `client/out/extension.js` и `server/out/eslintServer.js` (production-режим)

Сборку можно выполнить и отдельно для отладки проблем:

```bash
npm run webpack
```

Результат в корне: `vscode-eslint-<version>.vsix` (текущая версия — из поля `version` в `package.json`).

## 5. Проверка

### Содержимое пакета

```bash
unzip -l vscode-eslint-*.vsix
```

В пакете должны быть два бандла и не должно быть `node_modules`:

```
extension/package.json
extension/client/out/extension.js
extension/server/out/eslintServer.js
extension/README.md
extension/CHANGELOG.md
extension/License.txt
...
```

### Линт и тесты (опционально, до упаковки)

```bash
npm run lint
npm run compile && npm test
```

Примечание: `npm run webpack` (запускается при упаковке) очищает `out/`-каталоги, поэтому тесты выполняйте до `vsce package`.

### Установка

```bash
code --install-extension vscode-eslint-*.vsix
```

или в VS Code: Extensions → `…` → **Install from VSIX...**

## 6. Типичные проблемы

| Проблема | Причина / решение |
|---|---|
| Ошибка сборки: не найдены модули из `$shared` | Симлинки потеряны при копировании — см. раздел 2, восстановить `ln -sfn` |
| `postinstall` не установил зависимости в `client/`/`server/` | Включён `ignore-scripts`; выполнить `npm config set ignore-scripts false` и повторить `npm ci` |
| Ошибки совместимости инструментов сборки | Проверить версию Node — рекомендуется 20.x LTS |
| `npm ci` падает с сетевой ошибкой | Настроить прокси/зеркало registry — см. раздел 1 |
| Расширение перезаписывается маркетплейсной версией | VSIX имеет тот же ID (`dbaeumer.vscode-eslint`). Для изолированной корпоративной установки смените `publisher` в `package.json` (например, на внутренний ID) и увеличьте `version` — тогда расширение будет существовать отдельно от оригинала |
