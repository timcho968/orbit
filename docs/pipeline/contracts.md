# Контракты

Стадия 4, brownfield: формы вычитаны из кода (`ui-fixes`), новые формы — что добавим.

## Существующие сигнатуры (используем как есть)

```kotlin
// WebViewFactory.kt
fun create(context: Context, prefs: Prefs, bridge: CosmeticBridge, incognito: Boolean): OrbitWebView
fun applyTheme(web: WebView, prefs: Prefs)
fun setDesktopMode(web: WebView, desktop: Boolean)

// AdblockService.kt (синглтон процесса)
fun get(context: Context): AdblockService   // companion
var engine: AdblockEngine                    // потокобезопасно, swap() при пересборке
var state: State                             // IDLE / LOADING / READY / FAILED
fun start()                                  // идемпотентная инициализация списков/движка
fun onBlocked()                              // счётчик блокировок

// AdblockEngine.kt
fun isValid(): Boolean
fun blocks(url: String, sourceUrl: String, requestType: String): Boolean
fun cosmetic(url: String): CosmeticResult?   // css, proceduralJson, genericHide
fun classIdCss(classes: Array<String>, ids: Array<String>, exceptions: Array<String>): Array<String>

// CosmeticBridge.kt
class CosmeticBridge(adblock: AdblockService, isEnabled: () -> Boolean)
fun attach(web: WebView); fun detach()

// OrbitWebViewClient.kt
class OrbitWebViewClient(tab: Tab, adblock: AdblockService, prefs: Prefs, callbacks: Callbacks)
// Callbacks: onPageUrlChanged / onPageStarted / onPageFinished / onBlockedCountChanged / onExternalIntent

// OrbitChromeClient.kt
class OrbitChromeClient(tab: Tab, host: Host)
// Host: onProgress / onTitle / onIcon(tab, icon: Bitmap?) / onNewWindow / onCreateNewTab /
//        onShowCustomView / onHideCustomView / onFileChooser

// Prefs.kt (SharedPreferences-обёртка)
var adBlockEnabled: Boolean; var cosmeticFiltering: Boolean; var blockThirdPartyCookies: Boolean
var doNotTrack: Boolean; var safeBrowsing: Boolean; var forceDarkWeb: Boolean
var homePage: String                              // дефолт "orbit://home"
fun isAllowlisted(host: String): Boolean

// UrlUtils.kt
fun host(url: String?): String                    // host без www, lowercase
fun isHttp(url: String?): Boolean
```

## Новые формы

```kotlin
// app/.../browser/FaviconStore.kt
object FaviconStore {
    const val ICON_SIZE_PX = 192               // 96dp @ xxhdpi
    fun save(context: Context, host: String, icon: Bitmap)   // центр-кроп → квадрат → PNG
    fun load(context: Context, host: String): Bitmap?
    // Хранилище: files/favicons/<host>.png; host нормализован UrlUtils.host
}

// app/.../browser/DownloadHelper.kt — вынос тела MainActivity.startDownload
object DownloadHelper {
    fun enqueue(context: Context, url: String, userAgent: String?,
                contentDisposition: String?, mimeType: String?, size: Long)
}

// app/.../ui/WebappActivity.kt
class WebappActivity : Activity() {
    companion object {
        const val EXTRA_URL = "url"            // Intent.getStringExtra(EXTRA_URL) — обязателен
        // нет URL → finish()
    }
}

// Shortcut-контракт (ShortcutManagerCompat, androidx.core)
//   id       = UrlUtils.host(url)              // один ярлык на хост
//   label    = tab.title (fallback: host)
//   icon     = FaviconStore.load(host) → IconCompat.createWithBitmap
//              fallback: adaptive-иконка приложения
//   intent   = Intent(context, WebappActivity::class.java)
//                .setAction(ACTION_VIEW).putExtra(EXTRA_URL, pageUrl)
//   повторный pushDynamicShortcut с тем же id — обновление, дублей не создаёт
```

## Манифест (новое)

```xml
<activity
    android:name=".ui.WebappActivity"
    android:exported="true"
    android:launchMode="singleTask"
    android:taskAffinity="com.orbit.browser.webapp"
    android:configChanges="orientation|screenSize|screenLayout|keyboardHidden|smallestScreenSize|density|uiMode"
    android:theme="@style/Theme.Orbit" />
```

> Изменение 2026-08-17 (находка слайса): изначально планировался `exported="false"`,
> но ярлык запускается лаунчером (чужой uid) — с `false` лаунчер получает
> SecurityException. Исправлено на `true` и проверено вживую.

## Хранилище

| Что | Где | Ключ идентичности |
|---|---|---|
| favicon | `files/favicons/<host>.png` (внутреннее) | хост (UrlUtils.host) |
| ярлыки | ShortcutManager (система) | тот же хост |
| движок адблока | синглтон `AdblockService` + `files/engine.bin` | — (один на процесс) |

## Конфигурация

| Параметр | Тип | Дефолт | Откуда | Смысл |
|---|---|---|---|---|
| `FaviconStore.ICON_SIZE_PX` | Int | 192 | константа | размер иконки ярлыка (96dp) |
| `EXTRA_URL` | String-extra | — | Intent | URL для WebappActivity |

Новых пользовательских настроек нет.

## Секреты и доступы

Секретов нет. Подпись APK — локальный keystore `/tmp/opencode/orbit-keystore.jks`
(значения паролей в контракт не вносятся; команды — в brief.md «Как запускать»).
Keystore в репозиторий не попадает (лежит вне дерева проекта).

## Ошибки

| Ситуация | Поведение |
|---|---|
| favicon не получен (`onIcon(null)`) | ярлык с дефолтной иконкой Orbit, не ошибка |
| `pushDynamicShortcut` упал (лаунчер не поддержал) | toast + лог, приложение не падает |
| WebappActivity запущена без `EXTRA_URL` | `finish()` |
| `AdblockService.state != READY` в webapp | движок пустой: блокировок нет, страница работает; `start()` идемпотентно поднимает движок |
| страница в webapp уходит с хоста ярлыка | ничего (webapp — обычный браузер без UI, навигация свободная) |
