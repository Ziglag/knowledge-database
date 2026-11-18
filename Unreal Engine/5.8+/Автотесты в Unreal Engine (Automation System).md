# Автотесты в Unreal Engine (Automation System)

Automation System — встроенная в Unreal Engine система, которая находит, запускает и показывает результаты автоматических тестов. Тесты — это обычный C++ код (или Blueprint/карты для функциональных тестов), зарегистрированный через макросы движка. Заметка объясняет, какие виды тестов бывают, как написать первый тест, как его запустить и как применять это к логике ИИ (evaluator'ы StateTree) и GAS.

**Основная мысль:** тест полезен, когда проверяет **маленький кусок логики с понятным входом и выходом**. Чем меньше код тянет за собой мир, акторов и таймеры, тем проще и надёжнее тест. Поэтому главный приём — отделить «принятие решения» (чистая функция) от «сбора данных из мира» и тестировать первое.

> Факты о макросах и флагах ниже сверены с исходниками движка: `Engine\Source\Runtime\Core\Public\Misc\AutomationTest.h`.

## Термины

- **Automation Test** — зарегистрированный класс-тест. Имеет «красивое имя» (`PrettyName`) в виде пути через точки, например `System.StateTree.PropertyPath.Offset`. По этому пути тест можно искать и фильтровать.
- **Флаги теста (`EAutomationTestFlags`)** — где и когда тест может выполняться. Обязательно нужны: *контекст приложения* (`EditorContext`, `ClientContext`, `ServerContext`, …) и *фильтр* (`SmokeFilter`, `EngineFilter`, `ProductFilter`, `PerfFilter`, `StressFilter`, `NegativeFilter`). Без них код не соберётся: в макросах стоит `static_assert`.
- **Session Frontend → Automation** — окно редактора со списком тестов и кнопкой запуска. Открывается из **Tools → Session Frontend**, вкладка **Automation**.
- **Spec (BDD-стиль)** — способ писать тесты блоками `Describe` / `It` / `BeforeEach` / `AfterEach`.
- **Latent-команда** — шаг теста, который выполняется несколько кадров (например, «подождать 2 секунды»). Нужен там, где результат зависит от времени.

## Какие бывают тесты

| Вид | Как объявить | Когда подходит |
|---|---|---|
| **Simple** | `IMPLEMENT_SIMPLE_AUTOMATION_TEST` + метод `RunTest` | Один сценарий, короткая проверка. |
| **Complex** | `IMPLEMENT_COMPLEX_AUTOMATION_TEST` + `GetTests` и `RunTest(Parameters)` | Один и тот же код на наборе входов (например, список таблиц или карт). |
| **Spec** | `BEGIN_DEFINE_SPEC` / `END_DEFINE_SPEC` или `DEFINE_SPEC`, внутри `Define()` | Много сценариев одного класса, общая подготовка в `BeforeEach`. Читается как описание поведения. |
| **Latent / Async** | `ADD_LATENT_AUTOMATION_COMMAND` или `LatentIt` в Spec | Нужно ждать кадры или время. |
| **Functional Test** | Актор `AFunctionalTest` на специальной карте | Нужен настоящий уровень: акторы, физика, ИИ. Самый дорогой вариант. |

Для логики вроде выбора цели лучший старт — **Spec** или **Simple**.

## Минимальный пример (Simple)

```cpp
#include "Misc/AutomationTest.h"

#if WITH_DEV_AUTOMATION_TESTS

IMPLEMENT_SIMPLE_AUTOMATION_TEST(
    FMyAddTest,
    "DevilGate.Sample.Add",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

bool FMyAddTest::RunTest(const FString& Parameters)
{
    TestEqual(TEXT("2 + 2"), 2 + 2, 4);
    return true;
}

#endif
```

Ключевые моменты:
- Макрос создаёт класс `FMyAddTest` и регистрирует его глобальным экземпляром, поэтому дополнительно ничего вызывать не нужно.
- `TestEqual`, `TestTrue`, `TestNull` и другие записывают ошибку в тест, но **выполнение продолжается**. Итог определяют накопленные ошибки; `return false` из `RunTest` тоже считается провалом, поэтому в конце обычно пишут `return true`.
- Оборачивать код в `#if WITH_DEV_AUTOMATION_TESTS` рекомендуется в проекте, чтобы тесты не попали в сборки, где автотесты отключены (`Test`/`Shipping`). Движок в своих тестах это не всегда делает, так что это осознанная практика проекта, а не требование.
- Для сравнения `float` используй `TestEqual` с допуском (`Tolerance`), а не `==`.
- Флаги обязательны: нужен хотя бы один флаг **контекста** (`EditorContext`, `ClientContext`, ...) и **ровно один** флаг **фильтра**. Иначе `static_assert` не даст собрать код.

| Фильтр | Смысл |
|---|---|
| `SmokeFilter` | Быстрые проверки, запускаются чаще всего |
| `EngineFilter` | Тесты самого движка (так сделан `StateTreeTestSuite`) |
| `ProductFilter` | Тесты игрового продукта — выбор для проекта |
| `PerfFilter`, `StressFilter`, `NegativeFilter` | Производительность, нагрузка, ожидаемые ошибки |

## Пример: чистая функция выбора цели и тест

Идея: логику выбора выносим из Evaluator'а в функцию без `UWorld`.

```cpp
// PriorityTargetSelection.h
struct FTargetCandidate
{
    float Distance = 0.f;
    int32 DebuffCount = 0;
};

/** Возвращает индекс цели: сначала ближайшая среди DebuffCount > 0, при равенстве в пределах допуска - с большим числом дебафов. */
int32 SelectWarlockTarget(const TArray<FTargetCandidate>& Candidates, float DistanceTolerance);
```

```cpp
// Private/Tests/PriorityTargetSelectionTest.cpp
#if WITH_DEV_AUTOMATION_TESTS
IMPLEMENT_SIMPLE_AUTOMATION_TEST(FSelectWarlockTargetTest, "DevilGate.AI.SelectWarlockTarget",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)

bool FSelectWarlockTargetTest::RunTest(const FString& Parameters)
{
    const TArray<FTargetCandidate> Candidates = {{500.f, 1}, {300.f, 0}, {320.f, 3}};
    // 300 без дебафов игнорируется; 320 ближе 500
    TestEqual(TEXT("Ближайшая с дебафами"), SelectWarlockTarget(Candidates, 10.f), 2);
    TestEqual(TEXT("Пустой список"), SelectWarlockTarget({}, 10.f), INDEX_NONE);
    return true;
}
#endif
```

Evaluator после рефакторинга лишь собирает `TArray<FTargetCandidate>` и вызывает функцию.

## Пример Spec

```cpp
BEGIN_DEFINE_SPEC(FPriorityTargetSpec, "DevilGate.AI.PriorityTarget",
    EAutomationTestFlags::EditorContext | EAutomationTestFlags::ProductFilter)
    // Общие данные для всех It — поля класса.
    TArray<int32> Values;
END_DEFINE_SPEC(FPriorityTargetSpec)

void FPriorityTargetSpec::Define()
{
    BeforeEach([this]() { Values.Reset(); });

    Describe("Когда кандидатов нет", [this]()
    {
        It("возвращает пустой результат", [this]()
        {
            TestEqual(TEXT("Количество"), Values.Num(), 0);
        });
    });
}
```

`BeforeEach` запускается перед каждым `It`, поэтому тесты не влияют друг на друга. Это важнейшее свойство хорошего теста: он **независим** и не зависит от порядка запуска.

## Как запускать

1. **В редакторе:** *Tools → Session Frontend → Automation*, отметить тесты по фильтру (например, `DevilGate`), нажать **Start Tests**. Нужно, чтобы флаги теста разрешали контекст редактора (`EditorContext`).
2. **Из командной строки** (удобно для автоматизации и CI):

```
UnrealEditor-Cmd.exe Project.uproject -ExecCmds="Automation RunTests DevilGate; Quit" -unattended -nopause -nullrhi -log
```

`RunTests` принимает имя или префикс пути теста. Параметры `-unattended` и `-nullrhi` нужны для запуска без интерактивных окон и без рендера. Точный набор параметров зависит от версии и окружения, проверяй его на своей сборке.

## Где хранить тесты и как собирать

- Самый простой вариант — папка `Private/Tests/` внутри основного модуля под `#if WITH_DEV_AUTOMATION_TESTS`. Минус: тестовый код живёт в игровом модуле.
- Чистый вариант — **отдельный модуль** (например, `DevilGateTests`) с зависимостью на основной модуль и типом `Editor` или `Developer`. Его не включают в shipping-сборки. Так сделан `StateTreeTestSuite` (модуль типа `UncookedOnly`). Тесты GAS, наоборот, лежат в `Private/Tests` внутри основного модуля `GameplayAbilities`.
- Новый модуль нужно описать в `.uproject` (`Modules`, тип `Editor`) и добавить в `ExtraModuleNames` у Target-файла редактора. Минимальный `DevilGateTests.Build.cs`:

```csharp
public class DevilGateTests : ModuleRules
{
    public DevilGateTests(ReadOnlyTargetRules Target) : base(Target)
    {
        PublicDependencyModuleNames.AddRange(new[] { "Core", "CoreUObject", "Engine", "DevilGate" });
    }
}
```

Если тестируемые классы не экспортируются (`DEVILGATE_API`), линковка не пройдёт — это ещё один довод в пользу чистых функций с экспортом или тестов внутри основного модуля.
- Результаты и логи: в Session Frontend — по каждому тесту; из командной строки добавь `-ReportExportPath="<папка>"` (JSON-отчёт) и смотри лог `Saved/Logs`.
- Конфигурации сборки влияют на тесты: автотесты в `Shipping` обычно отсутствуют (см. [[Конфигурации сборки Unreal Engine]]).

## Как тестировать логику ИИ и GAS

Evaluator'ы StateTree, контроллеры и способности тесно связаны с миром: перцепция, ASC, таймеры, делегаты. Реалистичные подходы, от простого к дорогому:

1. **Выделить чистую функцию.** Evaluator собирает данные в простую структуру (дистанция, число дебафов, сопротивление), а отдельная функция по списку таких структур выбирает цель. Тест вызывает только эту функцию. Это быстро, стабильно и не требует мира.
2. **Мини-мир в тесте.** Создать `UWorld` и акторы, добавить ASC, применить эффекты, вызвать метод. Всё сложнее настраивать, а ошибки труднее диагностировать.
3. **Functional Test на карте.** Проверка поведения «глазами игрока». Нужен при регрессиях вида «моб не атакует». Дорого поддерживать.

Что обычно мешает:
- `private`-методы: тест их не видит. Решения: вынести логику в отдельный класс или функцию, использовать `friend`, либо тестировать через публичный интерфейс.
- Таймеры (`SetTimer`): либо вынести период и вызывать метод напрямую, либо использовать latent-команды.
- Случайность: подменяемый источник случайных чисел или фиксированное зерно.
- Данные GAS (теги, атрибуты): перед тестом нужно, чтобы теги были зарегистрированы (в редакторе они берутся из ini и нативных определений).

Где посмотреть образцы в движке (исходники читать можно без запроса доступа):
- `Engine\Plugins\Runtime\StateTree\Source\StateTreeTestSuite\Private\` — тесты StateTree и макрос `IMPLEMENT_STATE_TREE_INSTANT_TEST`.
- `Engine\Plugins\Runtime\GameplayAbilities\Source\GameplayAbilities\Private\Tests\` — тесты тегов и предсказаний GAS.
- `Engine\Source\Runtime\Core\Public\Misc\AutomationTest.h` — все макросы, флаги и `TestXxx`.

## Частые ошибки

- **Тест не виден в Session Frontend.** Проверь флаги контекста, фильтр и `#if WITH_DEV_AUTOMATION_TESTS`, а также что модуль загружен.
- **Тест «проходит», хотя проверка упала.** Не полагайся на `return true`: ошибки накапливаются внутри теста через `TestXxx`, а итог берётся из них.
- **Тесты зависят друг от друга.** Общее состояние (глобальные данные, синглтоны) нужно сбрасывать в `BeforeEach`/`AfterEach`.
- **Сравнение `float` через `==`.** Используй допуск.
- **Тест слишком большой.** Если для одного кейса нужно поднимать полмира, стоит выделить логику.
- **Тест проверяет реализацию, а не поведение.** Такой тест ломается при каждом рефакторинге.

## Вывод

Начинай с **маленьких Spec-тестов на чистую логику**, запускай их из Session Frontend, а затем добавляй командную строку для автоматизации. Тесты с миром и функциональные тесты оставляй для критичных сценариев. Хорошая цель: у каждого решения ИИ («кого атаковать», «когда отступать») есть тест, который падает, если логика меняется.

## Связанные заметки

- [[Конфигурации сборки Unreal Engine]] — почему тесты зависят от конфигурации.
- [[AI и StateTree]] — что именно мы тестируем в evaluator'ах.
- [[Game Ability System (GAS)]] — теги, атрибуты и эффекты, которые нужно подготовить в тесте.
- [[GetWorld, жизненный цикл и контекст делегатов]] — как получать мир и когда он доступен (нужно для тестов с акторами).
