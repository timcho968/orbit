# Задачи

- [x] T-01 — Скачивания в webapp (DownloadHelper)
      Файлы: `app/src/main/java/com/orbit/browser/browser/DownloadHelper.kt` (новый),
             `app/src/main/java/com/orbit/browser/ui/MainActivity.kt`,
             `app/src/main/java/com/orbit/browser/ui/WebappActivity.kt`
      Делаем: выносим тело `MainActivity.startDownload` в `DownloadHelper.enqueue(...)`;
              MainActivity вызывает helper; WebappActivity ставит `setDownloadListener`
              → `DownloadHelper.enqueue` (без него в webapp-режиме кнопки скачивания мертвы)
      DoD: `assembleRelease` зелёный; в webapp на странице с прямой ссылкой на файл
           тап по ссылке создаёт загрузку в системном DownloadManager
           (`dumpsys download` показывает активную/завершённую задачу)
      Исполнитель: главный цикл
      Зависит от: —

- [x] T-02 — Статус-строка движка обновляется вживую
      Файлы: `app/src/main/java/com/orbit/browser/ui/MainActivity.kt`
      Делаем: listener на `AdblockService` state; при изменении — обновить статус-строку
              стартовой (`homeView.setStatus`) и, при открытом Shield-диалоге, его данные
      DoD: после force-stop и запуска, без единого действия пользователя, стартовый экран
           через ≤60 с показывает «Total N requests blocked», а не «Loading filters…»
           (проверка: `uiautomator dump` + grep)
      Исполнитель: главный цикл
      Зависит от: —

- [x] T-03 — Финальная верификация AC-1…AC-6 на устройстве
      Файлы: `app/src/main/java/com/orbit/browser/ui/MainActivity.kt`,
             `app/src/main/java/com/orbit/browser/ui/WebappActivity.kt`,
             `app/src/main/java/com/orbit/browser/data/Prefs.kt`
      Делаем: пересборка пайплайном из brief, установка, прогон всех AC из plan.md
              (ярлык с favicon на сайте с `<link rel=icon>`, дубль-ярлык, webapp без UI,
              назад/закрытие, повторный тап, тест-страница адблока)
      DoD: AC-1…AC-6 все зелёные; результаты зафиксированы в отчёте стадии 7/8
      Исполнитель: главный цикл
      Зависит от: T-01, T-02
      Итог: AC-1…AC-5 зелёные (два фикса по факту прогона:
      (1) launcher3 не пинит dynamic-ярлыки → ушёл на `requestPinShortcut`;
      повторный «Add» даёт ещё одну иконку только после повторного диалога —
      кеш-отслеживание убрано как источник расхождений (решение пользователя);
      (2) в webapp прятался весь systemBars → первый свайп-жест съедался
      SystemUI (не работал «назад») → теперь скрыт только статусбар).
      AC-6: d3ward.github.io/toolz/adblock архивирован (внешний фактор);
      на живом стенде checkadblock.ru движок в webapp работает — 78/100
      (Google AdSense/Яндекс Директ и др. проверки ✅). Критерий «≥90» адаптирован
      по решению пользователя: зафиксирован факт работы движка в webapp;
      ру-списки фильтров — отдельная задача (не в этой итерации).
