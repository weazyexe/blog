+++
title = "Адаптируем Android приложение для незрячих людей. Часть 2: Jetpack Compose"
description = "Вторая часть ряда текстов про Accessibility в Android. В этом тексте объясняю как работать с Accessibility в Jetpack Compose"
date = "2022-10-21T10:20:00+04:00"

[extra]
cover_url = "assets/cover.jpg"
+++

{{ image_fill_width(path = "assets/cover.jpg") }}

Accessibility, или доступность, — важная штука в разработке программного обеспечения, особенно под мобильные платформы. В июле 2022 года я написал [статью про Accessibility в Android](@/accessibility-views/index.md). Тогда я рассказал про имплементацию Accessibility в системе View, но не сказал ни слова про поддержку модного и молодёжного Jetpack Compose.

А поговорить есть о чём: сделать графический интерфейс приложения на Compose доступным для людей с ограниченными возможностями стало гораздо легче, чем на View-интерфейсе.

> Рекомендую сначала ознакомиться с [первой частью](@/accessibility-views/index.md). Вы узнаете про Accessibility в Android, службу специальных возможностей TalkBack и познакомитесь с хорошими практиками адаптации приложения для пользователей с ограниченными возможностями. Без понимания этих моментов будет сложнее понять, что происходит в этой статье.

## Чек-лист доступности приложения

Для начала давайте вспомним, что нужно, чтобы приложение стало доступным для людей с ограниченными возможностями.

- Приложение просканировано с помощью [Accessibility Scanner](https://play.google.com/store/apps/details?id=com.google.android.apps.accessibility.auditor) и найдены проблемные места.

- Шрифты — только в *sp*-юнитах.

- Сложные компоненты с текстом имеют нефиксированный размер: пользователь может увеличить размер шрифта по системе.

- Заголовки помечены с помощью `accessibilityHeading = true`, чтобы пользователь мог перемещаться по ним через TalkBack.

- Соблюдены цветовые контрасты: нет мешанины цветов, контент не сливается с фоном.

- Не используется КАПС: на некоторых девайсах TalkBack может зачитать такой текст как аббревиатуру. Это некорректно.

- Минимальный размер кликабельных элементов — 48x48dp.

- `contentDescription` указан там, где нужно. Если нужно, чтобы TalkBack игнорировал элемент, задан параметр `importantForAccessibility = no`.

- Использован Live Regions, если необходимо зачитывать динамически обновляемую на UI информацию.

- Проверен порядок навигации между компонентами.

- Проверено, насколько удобно пользоваться сложными кастомными компонентами.

## Семантическое дерево

Графический интерфейс на Jetpack Compose рисуется с помощью `@Composable`-функций. Их совокупность называется **деревом композиции**: грубо говоря, это интерфейс, который мы видим на экране смартфона. 

Чтобы понимать, как работает Accessibility в Compose, следует разобраться с термином **семантическое дерево**. Оно описывает графический интерфейс альтернативным способом: содержит информацию о семантике, то есть *что это за компонент и что с ним можно сделать*. Для Accessibility Services и [Testing Framework](https://developer.android.com/jetpack/compose/testing) это более понятно.

> Accessibility Services — утилиты, которые упрощают работу со смартфоном для людей с ограниченными возможностями. К ним относятся TalkBack, Switch Access, Voice Access и так далее. Важно не путать их с AccessibilityService — одноимённым сервисом в Android. 

В документации есть хорошая схема семантического дерева. 

{{ image(path = "assets/semantic_tree.png", description = "Схема семантического дерева") }}

Сверху — видимый UI: обычный список элементов с Floating Action Button и Navigation Bar. Снизу — семантическое дерево, где каждый лепесток графа — это компонент и его семантика. TalkBack будет проходить по всем нодам дерева и зачитывать семантическую информацию.

## Как сделать UI на Compose доступным

### Кликабельные элементы

Пользователю с ограниченными возможностями **нужно** знать, можно сделать что-то с элементом или нет. Чтобы компонент в Compose стал кликабельным, не нужно никаких сложных конструкций: достаточно установить модификатор `Modifier.clickable` и передать туда лямбду. Это будет выглядеть примерно так.

```kotlin
Box(
    modifier = Modifier.clickable(onClickLabel = "Do something") {
        // Do something
    }
) {
  Text(text = "OK")
}
```

Если вдруг доступа к модификатору `clickable` нет, можно назначить слушателя на событие нажатия через `Modifier.semantics`.

```kotlin
Box(
    modifier = Modifier.semantics {
        onClick(label = "Do something") {
            // Do something
            true
        }
    }
) {
    Text(text = "OK")
}
```

### Модификатор semantics 

В `Modifier.semantics` обычно указывают семантическую информацию о компоненте: что за компонент, что с ним можно сделать и какое у него состояние. `Modifier.clearAndSetSemantics` создан, чтобы очистить семантику у дочерних `@Composable`-компонентов и перегрузить её.

В обеих функциях передаётся объект `SemanticsPropertyReceiver`, который мы можем редактировать, **но не можем считывать оттуда данные**.

### onClickLabel

TalkBack при наведении на кнопку зачитает *«Button, Double tap to __activate__»*. Если нужно дать больше семантики компоненту, можно заменить слово *«activate»* на любое другое.

В качестве примера возьмём карточку с данными о криптокошельке. Его мы разбирали в [прошлой статье](@/accessibility-views/index.md). При наведении на карточку слово *«activate»* не даст никакого контекста. Если заменить формулировку на более осмысленную, получится достаточно user-friendly: *«Card, ETH wallet, Double tap to __view wallet details__»*.

{{ image_fill_width(path = "assets/wallet_card.png") }}

### Информация о компоненте

У View в Android есть три основных семантических параметра:

- `contentDescription` — описание контента, находящегося в компоненте.

- `stateDescription` — описание состояния элемента: enabled/disabled, checked/unchecked и так далее.

- `roleDescription` — роль компонента на UI.

{{ image(path = "assets/content_description_bart.png") }}

Android развивается, способы построения UI тоже, а `contentDescription` остается: его указывают для изображений или сложных компонентов. 

У функции `Image` параметр `contentDescription` является обязательным. Можно положить туда `null`, и это будет аналогом `android:importantForAccessibility=”no”`.

```kotlin
Image(
    imageVector = ImageVector.vectorResource(R.drawable.ic_image),
    contentDescription = stringResource(R.string.image_content_description)
)
```

Если нужно задать описание не изображению, используем всё тот же `Modifier.semantics` и обновляем данные в поле `SemanticsProperties.contentDescription`:

```kotlin
val desc = stringResource(R.string.component_content_description)
Box(
    modifier = Modifier.semantics {
        contentDescription = desc
    }
) {
  // Some content
}
```

### Заголовки

Заголовки нужны для того, чтобы у пользователя была возможность удобной навигации по экрану с большим количеством информации. TalkBack имеет [несколько режимов навигации](https://support.google.com/accessibility/android/answer/6006598?hl=en) по компонентам, но мы затронем только навигацию по заголовкам. Чтобы сделать компонент заголовком, нужно назначить ему через модификатор `Modifier.semantics` семантику заголовка:

```kotlin
Text(
    modifier = Modifier.semantics {
        heading()
    },
    text = "Heading 1"
)
```

### Списки

Поддержка Accessibility в системе View хороша тем, что для списка на `RecyclerView` не нужно задавать никакую семантику: всё работает из коробки. Наводимся на элемент списка, и TalkBack зачитывает, что это за элемент списка, какой он по счёту и какого размера список.

{{ image(path = "assets/recycler_view_meme.png") }}

В Compose такой функциональности из коробки нет: придётся делать самим. Для этого есть два семантических свойства: `collectionInfo` и `collectionItemInfo`. Свойство `collectionInfo` содержит информацию о размере списка или таблицы: количество строк и столбцов. Для обычного списка количество столбцов по умолчанию равно 1. Установить `collectionInfo` нужно в модификатор `semantics` компонента списка.

```kotlin
Column(
    modifier = Modifier.semantics {
        collectionInfo = CollectionInfo(versions.size, 1)
    }
) {
    . . .
}
```

Свойство `collectionItemInfo` указывается на каждый элемент списка. Параметры `rowIndex` и `columnIndex` отвечают за позицию элемента в списке. Параметры `rowSpan` и `columnSpan` больше используются в таблицах и отвечают за количество элементов, охваченных ячейкой таблицы. Для простого списка эти параметры использовать не будем: установим значение 1.

```kotlin
versions.forEachIndexed { index, version ->
    Text(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
            .semantics {
                collectionItemInfo = CollectionItemInfo(
                    owIndex = index,
                    rowSpan = 1,
                    columnIndex = 1,
                    columnSpan = 1
                )
            },
        text = version
    )
}
```

С версии Compose 1.2.0 и выше это работает и для имплементаций `LazyList`.

```kotlin
LazyColumn(
    modifier = Modifier.semantics {
        collectionInfo = CollectionInfo(versions.size, 1)
    }
) {
    itemsIndexed(
        items = versions,
        key = { _, version -> version.hashCode() }
    ) { index: Int, version: String ->
        Text(
            modifier = Modifier
                .semantics {
                    collectionItemInfo = CollectionItemInfo(
                        rowIndex = index,
                        rowSpan = 1,
                        columnIndex = 0,
                        columnSpan = 1
                    )
                }
                .fillMaxWidth()
                .padding(16.dp),
            text = version
        )
    }
}
```

### Кастомные действия

Кастомные действия нужны для сложновыполнимых действий на UI: например, свайпов. Если у пользователя включен TalkBack, свайпнуть можно, но это достаточно неудобно. Кроме того, TalkBack бывает включён наряду вместе со Switch Access: там свайп вообще невозможен, поскольку пользователь не прикасается к экрану. 

Давайте для примера разберём, как реализовать функциональность swipe-to-dismiss для людей с ограниченными возможностями. Сверстаем обычный элемент списка из приложения почтового клиента.

```kotlin
Column(
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp)
) {
    Text(
        modifier = Modifier.padding(bottom = 4.dp),
        text = "Habr",
        fontWeight = FontWeight.Bold
    )
    Text(text = "Самое интересное по вашим хабам")
}
```

{{ image(path = "assets/swipeable.png", description = "Простой элемент списка с заголовком и описанием") }}

Поскольку UI верстаем через Compose, с реализацией swipe-to-dismiss не надо сильно париться: будем использовать стандартную из библиотеки. Оборачиваем вью в функцию `SwipeToDismiss`: теперь можем свайпать компонент 🎉

```kotlin
val state = rememberDismissState()
SwipeToDismiss(
    state = state,
    background = {}
) {
    // Здесь — Column с контентом
}
```

{{ animated_image(path = "assets/swipeable_behavior.mp4") }}

И всё бы хорошо: пользователь рад, что может свайпать, разработчик рад, что может сделать такую функциональность за пару строк. Но мы собрались здесь ради Accessibility, а эта реализация максимальна неудобна для пользователей с ограниченными возможностями: тексты не читаются как единый компонент, они независимы друг от друга.

{{ animated_image(path = "assets/swipeable_wrong_talkback.mp4") }}

Эту проблему можно решить, установив значение `true` в модификаторе `semantics`, аргумент `mergeDescendants`. Это означает, что мы замёржили все дочерние элементы этого компонента и их невозможно выделить через TalkBack. Теперь выделяться будет весь элемент списка. То же самое можно было сделать на `View`, пометив все дочерние `View` параметром `importantForAccessibility = no`.

{{ image(path = "assets/swipeable_right_talkback.png", description = "Единый компонент: TalkBack зачитывает всю информацию из него") }}

Теперь кастомизируем swipe-to-dismiss для людей с ограниченными возможностями, чтобы можно было открывать меню с дополнительными действиями и выбирать нужный элемент в списке через TalkBack.

Проинициализируем `CoroutineScope` и достанем из ресурсов локализованную строку. Напоминаю, что **все строковые ресурсы из приложения**, используемые для Accessibility, **обязаны быть локализованными**.

```kotlin
val scope = rememberCoroutineScope()
val deleteString = stringResource(R.string.delete)
SwipeToDismiss(
    . . .
)
```

Добавим в `semantics` нужную логику: при вызове кастомного действия явно свайпнуть эту ячейку. И вот что получается.

```kotlin
Column(
    modifier = Modifier
        .semantics(mergeDescendants = true) {
            customActions = listOf(
                CustomAccessibilityAction(deleteString) {
                    scope.launch {
                        state.dismiss(DismissDirection.StartToEnd)
                        }
                        true
                }
            )
        }
        .fillMaxWidth()
        .padding(16.dp)
)
```

{{ screen_recording(path = "assets/swipeable_custom_actions.mp4") }}

## Особенности Accessibility в Compose

В отличие от системы View, поддержка Accessibility в Compose немного урезана:

- В роль теперь нельзя задать любую строку — только одну из заранее предопределённых: Button, Checkbox, RadioButton, Tab, Image и Switch.

- Нельзя отслеживать Accessibility-события у компонентов. Например, теперь не получится через `AccessibilityDelegate` и перегруженную функцию `performAccessibilityAction` перехватывать события. Сам по себе `AccessibilityDelegate` никуда не пропал: об этом — ниже.

- В Compose нет аналога `ViewCompat.announceForAccessibility`.

Это основные особенности, которые мне удалось обнаружить. Скорее всего, есть ещё что-то, чем обделили Compose. Эти изменения были сделаны не просто так: неправильное использование функциональности могло ввести пользователей в заблуждение, а разработчики могли понавставлять себе палки в колёса. 

Основная рекомендация: доработки по Accessibility должны быть минимальны, не нужно строить никаких велосипедов. Оставшиеся инструменты должны покрывать весь скоуп возможных проблем.

## Куда всё-таки пропал AccessibilityDelegate

А он никуда и не пропадал! Весь Compose рисуется на `ComposeView`. Compose поддерживает Accessibility, значит, у `ComposeView` есть свой собственный делегат. И имя ему `AndroidComposeViewAccessibilityDelegateCompat.android.kt`. Название страшное, логика внутри тоже. Смысл делегата — адаптировать все указанные семантики в Compose для ОС Android.

{{ image(path = "assets/java_class_meme.png") }}

## Что в итоге

При разработке Jetpack Compose Гугл проделал огромную работу. Теперь намного легче сделать приложение доступным: не надо использовать страшные сущности типа `AccessibilityDelegateCompat` и перегружать в этом делегате странные методы. Подход стал проще: **пишешь код** как пишешь, и **оно работает**.

> Текст изначально публиковался на [habr.ru](https://habr.com/ru/companies/surfstudio/articles/694622/)