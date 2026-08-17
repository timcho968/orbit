# Project Brief: Add to Home screen + Webapp-режим (Orbit)

## Суть

Orbit — форк одноимённого открытого браузера (WebView + движок Brave adblock-rust
с uBO-списками), в который мы уже внесли серию UI-правок: нижняя адресная строка в стиле
Jelly, корректные инсеты под вырез камеры, фиксы скролла при выходе из полноэкранного
видео, фикс утечки URL при закрытии вкладки, кнопка-стоп при зависшей загрузке.

В этом проходе добавляем фичу «Add to Home screen»: пункт меню создаёт ярлык сайта
на рабочем столе (иконка — favicon), а запуск по ярлыку открывает сайт в WebappActivity —
полноэкранном режиме без какого-либо UI браузера (адресная строка, панели, статусбар),
с работающим адблоком и навигацией «назад» по истории страницы.

Успех: открытый yummyani → меню → «Add to Home screen» → ярлык с favicon на рабочем
столе → тап → сайт открывается полноэкранно без UI, адблок активен, «назад» ходит
по истории, повторный тап по ярлыку возвращает в уже запущенный экземпляр.

## Пользователь и сценарий

Один пользователь (владелец устройства), запускает вручную на телефоне Redmi 9
(crDroid 15, Android 15, arm64). Сценарий — «сайт как приложение»: аниме-каталог,
часто открываемые сервисы, без возни с вкладками и адресной строкой.

## Скоуп

**В скоупе:**
- Пункт меню «Add to Home screen» (строка `add_to_home` уже в ресурсах)
- Сохранение favicon при получении (`onReceivedIcon` → `files/favicons/<host>.png`)
- Динамический ярлык через `ShortcutManagerCompat`; идентичность ярлыка — хост
  (один ярлык на сайт, повторное добавление обновляет существующий)
- WebappActivity: WebView на весь экран, статусбар скрыт (immersive, свайп показывает),
  адблок через общий `AdblockService` (движок синглтон — без двойной памяти),
  отдельный task (`taskAffinity`, `singleTask`, `onNewIntent` для повторного тапа)
- Скачивания в Webapp-режиме работают (логика DownloadManager выносится в общий helper)

**Вне скоупа (non-goals):**
- Чтение PWA-manifest.json сайтов (иконку/имя/standalone берём из браузера)
- Автообновление иконок ярлыков после создания
- Пункт «Remove shortcut» в меню браузера (удаление — системным лонг-тапом)
- Несколько ярлыков на один сайт (ключ — хост, а не полный URL)

## Стек и окружение

- **Цель:** Android, arm64-v8a, minSdk 21, targetSdk/compileSdk 36, WebView из прошивки
- **Язык:** Kotlin 2.2.20; сборка Gradle 9.1.0 + AGP 8.13, JDK 21
- **Движок адблока:** Rust-crate `adblock` 0.10 (Brave) через JNI (`.so` закоммичены в `jniLibs/`)
- **Среда сборки:** NixOS 26.05 — свои грабли: aapt2/zipalign требуют patchelf + LD_LIBRARY_PATH
  (libcxx/zlib/gcc-lib/glibc), apksigner запускается через `bash`, Android SDK собран
  sdkmanager-ом в `/tmp/opencode/android-sdk` (platform-36, build-tools 36.0.0)
- **Установка:** `adb install -r` (keystore `/tmp/opencode/orbit-keystore.jks`), устройство `b0d3d4a50505`
- Тестов нет, CI нет

## Внешние зависимости

- Фильтр-списки (uAssets, EasyList, EasyPrivacy, Peter Lowe, AdGuard TR) и scriptlets —
  загружаются рантаймом с официальных HTTPS-источников
- Лаунчер crDroid (Quickstep) — должен показывать динамические ярлыки ShortcutManager
- Файловая система приложения (`files/`) — хранение favicon; без внешних прав

## Ограничения и риски

1. **Динамические ярлыки на Quickstep/crDroid** — стандартный API, но видимость ярлыка
   на конкретном лаунчере проверяем фактически на устройстве (стадия 5). Риск низкий.
2. **Favicon как иконка ярлыка** — `IconCompat.createWithBitmap` требует квадратную
   картинку; favicon может быть 16×16 или 32×32, придётся масштабировать до 96dp.
   Риск низкий, закрывается кодом.
3. **WebappActivity и адблок** — `AdblockService` поёт из глобального состояния
   (`AdblockService.get(context)`); жизненный цикл движка между двумя activity не проверен.
   Если движок не «проснётся» в webapp — потребуется явная инициализация. Закрывается
   вертикальным слайсом (стадия 5).
4. **singleTask + onNewIntent** — поведение повторного тапа по ярлыку (переоткрыть URL
   в том же task) проверяем вручную на устройстве.
5. **Полноэкранный immersive в webapp** — insets-логика из MainActivity не переносится;
   webapp живёт поверх системных панелей, контент под вырезом камеры — принято как есть
   (стандарт для standalone-режима).
6. **Незакоммиченные правки прошлой сессии** (6 файлов в ветке `ui-fixes`) — фича
   пишется поверх них; коммиты — по явной просьбе пользователя.

## Как запускать

Проверено вживую 2026-08-17 (уточняется на стадии 5):

```bash
export ANDROID_HOME=/tmp/opencode/android-sdk ANDROID_SDK_ROOT=/tmp/opencode/android-sdk
export LD_LIBRARY_PATH=$(nix eval --raw nixpkgs#libcxx.outPath)/lib:$(nix eval --raw nixpkgs#zlib.outPath)/lib:$(nix eval --raw nixpkgs#gcc.cc.lib.outPath)/lib:$(nix eval --raw nixpkgs#glibc.outPath)/lib
./gradlew --no-daemon assembleRelease

BT=/tmp/opencode/android-sdk/build-tools/36.0.0
$BT/zipalign -f 4 app/build/outputs/apk/release/app-arm64-v8a-release.apk /tmp/opencode/orbit.apk
bash $BT/apksigner sign --ks /tmp/opencode/orbit-keystore.jks --ks-pass pass:orbit123 --key-pass pass:orbit123 --out /tmp/opencode/orbit-signed.apk /tmp/opencode/orbit.apk
adb install -r /tmp/opencode/orbit-signed.apk && adb shell am force-stop com.orbit.browser
```

Тесты: отсутствуют. Ручная проверка через `adb shell uiautomator dump` + `dumpsys activity top`.

## Проверка webapp-режима (слайс, отработано 2026-08-17)

```bash
adb shell am start -a android.intent.action.VIEW -n com.orbit.browser/.ui.WebappActivity --es url "https://example.com/"
adb shell uiautomator dump /sdcard/w.xml && adb pull /sdcard/w.xml /tmp/opencode/w.xml
# UI браузера отсутствует: grep -c "id/urlBar" /tmp/opencode/w.xml → 0
```

Известное: тяжёлые страницы в webapp рендерятся долго на холодном старте (30–75 с
на Redmi 9); favicon сохраняется только если сайт объявляет `<link rel=icon>`
(иначе — дефолтная иконка Orbit).
