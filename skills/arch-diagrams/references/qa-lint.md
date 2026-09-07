# QA-lint — программная детекция и избегание пересечений

Линтер: `scripts/arch_lint.js` — тело `use_holst`-вызова (прочитай файл,
подставь в `code`). Работает на любом борде, возвращает JSON-отчёт.

## Что детектирует (программно, без визуальной модели)

| Класс | Метод | Точность |
|---|---|---|
| Объект ↔ объект | `absoluteBounds` AABB попарно; пары предок-потомок исключены (контейнеры — не коллизии) | точная |
| Стрелка ↔ объект | reconstruction полилинии: `attachPoint` (side-центр / AUTO-проекция / point) + `arrowType`/`controls` → сегменты → Liang-Barsky vs AABB; свои endpoint-ноды исключены | exact для явных side/point/controls; **approx для AUTO-маршрутов** (нативный роутер не отдаёт waypoints) |
| Лейбл ↔ объект | `getLabelBounds()` vs все AABB (кроме endpoint-нод) | точная |
| Стрелка ↔ стрелка | segment-segment, пары с общим endpoint исключены | approx там же, где полилиния approx |

Отчёт: `{counts, overlaps, arrowHits, approxHits, labelHits, crossings}`.
`arrowHits` — подтверждённые (exact), `approxHits` — advisory. Frame-границы
исключены из hit-детекции: стрелка, входящая в зону к своей ноде, — не дефект.

## Избегание (приоритет мер)

Линтер v3 сам генерирует **`suggestions`** — готовые `apply`-объекты
(валидированы: кандидат-маршрут проверен на чистоту против всех нод):

- `kind:"reroute"` — `arrow.update(apply)`: явные side + controls-коридор
  (±48 от задетой ноды; для AUTO-стрелок — explicit sides, что делает
  маршрут точным: `confidence:"exact-after-apply"`);
- `kind:"shift"` — `node.update(apply)` для объекта-перекрытия (сдвиг
  меньшей ноды на overlap+24);
- `kind:"label"` — `arrow.update(apply)` с position 0.15 (fallback 0.85).

Ручные меры (если suggestions нет):
1. Точечный reroute: `controls:[{orientation, x|y}]` через свободный коридор.
2. Смена endpoint-`side`.
3. Сдвиг ноды `node.update({x,y})`.
4. `position` лейбла (0..1) + `side: path-left/path-right`.
5. Пересечения **линия × линия** — приемлемый компромисс; чинить только
   когда линия проходит сквозь ноду/лейбл.

Ритуал: lint → применить suggestions → lint повторно → delta-вердикт
→ финальный rect-скриншот. Closed-loop проверено: искусственный дефект
(стрелка сквозь карточку) → suggestion → apply → 0 hits; recordings-борд:
overlaps 1→0, exact-hits 2→0, labelHits 7→0 за 2 вызова.

## Ограничения и визуальный fallback

- AUTO-маршруты (нет side/controls) реконструируются эвристикой — их флаги
  advisory; при сомнении смотри rect-скриншот.
- Curved-стрелки аппроксимируются хордой.
- **Визуальная модель (крайний случай)**: `holst_get_screenshot` → картинка в
  qwen-mm (`read_image` / `draw_bbox`) — когда нужна оценка «визуальной
  зашумлённости» (плотность, наезды теней), не геометрии. Геометрия — всегда
  программно.
