# Как добавить новый класс

Класс = **способности + дерево талантов + (необязательно) условие получения, хуки и HUD**. Рыцарь Смерти (`deathknight/`) — рабочий образец.
Класс можно добавить в этом моде (`ClassRegistry.init()`) или из другого мода через событие.

## 1. Способности
Наследуйте `core.Ability` (или копируйте шаблон `DkAbility`):
```java
public final class PyroAbilities {
    public static final Ability FIREBOLT = new Ability("firebolt", 1, 30) {          // id, уровень, кулдаун (тики)
        @Override public boolean canPay(ServerPlayer p, PlayerRpgData d) { return d.extra.getInt("heat") < 100; }
        @Override protected boolean cast(ServerPlayer p, PlayerRpgData d) {
            LivingEntity t = Targeting.find(p, 16, true);       // луч + конус-автоприцел (с Epic Fight работает всегда)
            if (t == null) return false;                         // false = «нет цели», кулдаун не запускается
            t.hurt(p.damageSources().indirectMagic(p, p), 6 + d.level * 0.2f);
            d.extra.putInt("heat", d.extra.getInt("heat") + 10); d.dirty = true;   // d.extra — своё хранилище ресурсов класса
            return true;
        }
    }.namespace("mymod").requires("fire_mastery").animated(8);    // необязательно: требует талант; анимированная иконка (8 кадров 32x32 столбиком)
}
```
`namespace("mymod")` задаёт пространство имён ключей lang и иконки (по умолчанию `rpgclasses`).
Лимит — 12 способностей на класс (`RpgClass.MAX_ABILITY_SLOTS`). Игрок экипирует не больше `maxEquippedSkills` (по умолчанию 6) — это управляется автоматически (`PlayerRpgData.refreshLoadout`: доступные навыки попадают в набор, пока есть место; недоступные убираются), колесо навыков и HUD работают для любого класса без доработок.

## 2. Класс, таланты, условие получения, хуки
```java
RpgClass c = new RpgClass(new ResourceLocation("mymod", "pyromancer"), List.of(PyroAbilities.FIREBOLT))
    .blocksArcane(false)                                  // true = запрет книг/свитков Iron's Spells
    .unlockedBy(new RitualUnlock(...))                    // или свой ClassUnlock; без него класс доступен всем
    .hooks(new RpgClass.Hooks() {                         // onTick / onMeleeHit / onActivated — все необязательны
        @Override public void onTick(ServerPlayer p, PlayerRpgData d, long now) { /* остывание */ }
    })
    .addBranch("flame").addBranch("embers")               // порядок = индекс ветки в талантах
    // id, ветка, строка, колонка, макс. ранг, родители (достаточно ЛЮБОГО), модификации (макс. ранг → выбор одной)
    .add(new Talent("fire_mastery", 0, 0, 1, 5, "", "wildfire,inferno"));
```
Сетка талантов произвольная: экран сам подбирает размеры под число веток/колонок/строк; родитель всегда в строке выше потомка; две вершины не делят клетку.
Требование строки: строки 0/1/2/3 открываются на уровнях 1/5/10/20 (`Talent.ROW_LEVEL`).

## 3. Регистрация
- В этом моде: `ClassRegistry.init()` → `register(Pyromancer.create())`.
- Из другого мода (мод-шина): `modBus.addListener((RegisterRpgClassesEvent e) -> e.register(Pyromancer.create()));`

## 4. Получение класса
- Без `unlockedBy` — класс доступен сразу в экране классов (клавиша **K**, пока нет класса).
- `RitualUnlock(центр, кольцо, мин. дист., макс. дист., нужны_ли_зажжённые, предмет, количество, ключ_описания)` — готовый ритуал: предмет бросают в центральный блок, вокруг которого с 4 сторон стоят блоки-кольца. Обрабатывается `events/RitualEvents` для **любого** класса с таким условием.
- Свой тип: реализуйте `ClassUnlock` (нужен только `description()` для карточки) и вызовите `PlayerRpgData#unlock(RpgClass)` на сервере, когда условие выполнено (достижение, квест, предмет…).
- Прогресс каждого класса хранится отдельно: смена класса (вне боя) не теряет уровень, таланты и модификации.

## 5. Ресурсы и HUD
Свои ресурсы храните в `d.extra` (NBT, сохраняется, синхронизируется, откладывается при смене класса) или добавляйте поля в `PlayerRpgData`. Отрисовка — на клиенте:
```java
ClientClassHuds.register(new ResourceLocation("mymod", "pyromancer"), (g, d, c, w, h, now, time, mc) -> { /* полоска жара */ });
```
Общая часть (уровень, опыт, панель способностей с кулдаунами и клавишами) рисуется автоматически.

## 6. Ассеты и локализация (имена = id)
- `textures/gui/class/<class>.png`, `textures/gui/ability/<id>.png`, `textures/gui/talent/<id>.png` — 32×32; анимированная иконка — полоса `32 × (32·кадры)`.
- Ключи lang: `class.<ns>.<class>` и `.desc`; `ability.<ns>.<id>` и `.desc`; `talent.<ns>.<id>` и `.desc`; `mod.<ns>.<id>` и `.desc`; `branch.<ns>.<branch>`; `ritual.<ns>.<class>` (если ритуал).
- `tools/gen_assets.py` генерирует заглушки-иконки (`icon`, `anim_icon`, `glyph`) — удобно для прототипа; `tools/validate.py` проверяет ключи lang и иконки Рыцаря Смерти (расширьте под свои id).
- Эффекты на GeckoLib: см. `RuneCircleEntity` / `UnderworldFxEntity` (модель, анимации и текстуры генерируются скриптом).

## 7. Дополнительные настройки класса (см. `SpiritMaster.java`)
```java
new RpgClass(id, abilities)
    .exclusive(true)               // только один класс: нельзя взять при наличии другого и нельзя сменить
    .xpScales(0.15, 0.10)          // множитель опыта за убийства, множитель опыта на уровень
    .modPointEvery(6)              // очко модификации (морфа) каждые N уровней (0 = значение конфига)
    .branchColors(0x1B8A99, 0x7A52B8, 0x6C9CE8)   // цвета веток в дереве и карточке класса
    .unlockedBy(new DescriptionUnlock("unlock.mymod.my_class"))   // свой способ получения: сами вызовите d.unlock(cls) (предмет, бросок кубика, квест)
```
Вспомогательные классы: `core.Scheduler.later(server, ticks, runnable)` — отложенные/повторяющиеся эффекты; `core.Combat` — урон/лечение с учётом глобального баланса (множитель способностей, PvP ×0.6, лимит лечения);
`spirit.Spectral` — шаблон «вне тела» (режим наблюдателя, тело-манекен, одержимость); `d.extra` — состояние класса (шкала души, таймеры).
Источники опыта нового класса — обычные Forge-события: см. `spirit/SpiritEvents.java` (рейд, торговля, зачарование) — добавляйте свои, вызывая `d.addXp(player, n)` и `NetworkHandler.sync(player)`.

## 8. Эффекты навыков
Для своих навыков используйте готовые эффекты (`fx.Fx`: `at`, `follow`, `beam`, `bolt`) или добавьте новые модели: см. `docs/SKILL_FX.md` («Как добавить эффект»).
