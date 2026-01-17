# Алгоритм генерации кроссворда

Этот документ описывает алгоритм генерации кроссворда, реализованный в проекте.

## Содержание

1. [Общий обзор](#общий-обзор)
2. [Структуры данных](#структуры-данных)
3. [Предварительная обработка слов](#предварительная-обработка-слов)
4. [Основной алгоритм генерации](#основной-алгоритм-генерации)
5. [Алгоритм размещения слова](#алгоритм-размещения-слова)
6. [Проверка совместимости](#проверка-совместимости)
7. [Расчёт доступных позиций](#расчёт-доступных-позиций)
8. [Параллельная генерация](#параллельная-генерация)
9. [Оценка качества кроссворда](#оценка-качества-кроссворда)
10. [Аннотирование и визуализация](#аннотирование-и-визуализация)

---

## Общий обзор

Алгоритм генерации кроссворда основан на **жадном алгоритме с элементами backtracking**. Основная идея:

1. Слова сортируются по частоте встречаемости их символов
2. Первое слово размещается в случайной позиции
3. Каждое следующее слово пытается найти пересечение с уже размещёнными словами
4. Если слово не может быть размещено, оно откладывается и пробуется позже
5. Процесс повторяется многократно (1000+ раз) для нахождения оптимального варианта
6. Выбирается кроссворд с максимальной плотностью заполнения

**Расположение исходного кода:**
- `src/main/scala/com/papauschek/puzzle/Puzzle.scala` — основной класс кроссворда
- `src/main/scala/com/papauschek/puzzle/PuzzleWords.scala` — сортировка слов
- `src/main/scala/com/papauschek/puzzle/PuzzleConfig.scala` — конфигурация
- `src/main/scala/com/papauschek/puzzle/Point.scala` — вспомогательные структуры

---

## Структуры данных

### Класс Puzzle

```scala
case class Puzzle(
  chars: Array[Char],      // Одномерный массив символов (сетка кроссворда)
  config: PuzzleConfig,    // Конфигурация размеров
  words: Set[String]       // Множество размещённых слов
)
```

**Файл:** `Puzzle.scala:18-20`

#### Хранение сетки

Сетка кроссворда хранится как одномерный массив `Array[Char]` размером `width × height`:
- Пробел `' '` означает пустую ячейку
- Буква означает заполненную ячейку

Преобразование координат:
```scala
// (x, y) → индекс в массиве
index = y * width + x

// индекс → (x, y)
x = index % width
y = index / width
```

**Файл:** `Puzzle.scala:70-75`

### PuzzleConfig

```scala
case class PuzzleConfig(
  width: Int = 18,         // Ширина сетки
  height: Int = 18,        // Высота сетки
  wrapping: Boolean = false // Режим "обёртывания" (для цилиндрической топологии)
)
```

**Файл:** `PuzzleConfig.scala:3-4`

### Point (оптимизированные координаты)

```scala
class Point(val underlying: Int) extends AnyVal {
  def x: Int = underlying & 0xFFFF
  def y: Int = underlying >> 16
}
```

Компактное хранение координат в одном 32-битном числе вместо двух.

**Файл:** `Point.scala:6-12`

### CharPoint (точка пересечения)

```scala
case class CharPoint(char: Char, x: Int, y: Int, vertical: Boolean)
```

Описывает позицию символа в кроссворде с указанием доступного направления для нового слова.

**Файл:** `Point.scala:21`

---

## Предварительная обработка слов

Перед генерацией слова сортируются по "полезности" — насколько хорошо они будут пересекаться с другими словами.

### Алгоритм сортировки

```scala
def sortByBest(words: Seq[String]): Seq[String] = {
  val allChars = words.flatten.toVector
  val allCharCount = allChars.length.toDouble

  // Частота каждого символа во всём наборе слов
  val frequency = allChars.groupBy(c => c).map {
    case (c, list) => (c, list.length / allCharCount)
  }

  // Оценка слова = сумма частот его символов
  def rateWord(word: String): Double = word.map(frequency).sum

  words.sortBy(rateWord).reverse
}
```

**Файл:** `PuzzleWords.scala:7-17`

**Логика:** Слова с более частыми символами размещаются первыми, так как они с большей вероятностью найдут пересечения с последующими словами.

**Пример:**
- Слово "КОШКА" содержит букву "К" (частая) и "А" (очень частая) → высокий рейтинг
- Слово "ЖЮРИ" содержит редкие буквы "Ж", "Ю" → низкий рейтинг

---

## Основной алгоритм генерации

### Точка входа

```scala
def generate(initialWord: String, list: List[String], config: PuzzleConfig): Seq[Puzzle] = {
  val emptyPuzzle = empty(config)
  val puzzles = Seq(
    initial(emptyPuzzle, initialWord, horizontal = false, centered = false),
    initial(emptyPuzzle, initialWord, horizontal = true, centered = false)
  )
  puzzles.map(p => generateAndFinalize(p, list))
}
```

**Файл:** `Puzzle.scala:252-255`

Создаются два начальных варианта — с горизонтальным и вертикальным размещением первого слова.

### Итеративное заполнение

```scala
@tailrec
private def generate(puzzle: Puzzle, words: List[String], tried: List[String]): Puzzle = {
  if (words.isEmpty) {
    puzzle  // Все слова обработаны
  } else {
    if (puzzle.words.contains(words.head)) {
      // Слово уже размещено — пропускаем
      generate(puzzle, words.tail, tried)
    } else {
      val options = puzzle.addWord(words.head)
      if (options.isEmpty) {
        // Слово не подходит — откладываем
        generate(puzzle, words.tail, words.head +: tried)
      } else {
        // Выбираем случайный вариант размещения
        val nextPuzzle = options(Random.nextInt(options.length))
        // Возвращаем отложенные слова в очередь
        generate(nextPuzzle, tried ++ words.tail, Nil)
      }
    }
  }
}
```

**Файл:** `Puzzle.scala:296-311`

### Стратегия backtracking

Если слово не может быть размещено:
1. Оно добавляется в список `tried`
2. Алгоритм переходит к следующему слову
3. После успешного размещения слова, все отложенные слова возвращаются в очередь
4. Это даёт второй шанс словам, которые не подошли ранее

---

## Алгоритм размещения слова

Метод `addWord` находит все возможные варианты размещения нового слова.

```scala
def addWord(word: String): Array[Puzzle] = {
  val puzzles = for {
    // Группируем символы слова по их значению
    (char, indicesList) <- word.zipWithIndex.groupBy(_._1).toList

    // Находим позиции с совпадающим символом в кроссворде
    point <- positions.getOrElse(char, Array.empty[CharPoint])

    // Для каждого совпадающего индекса в слове
    (_, index) <- indicesList

    // Вычисляем координаты начала слова
    (x, y, right, bottom) = if (point.vertical) {
      (point.x, point.y - index, point.x, point.y - index + word.length - 1)
    } else {
      (point.x - index, point.y, point.x - index + word.length - 1, point.y)
    }

    // Проверяем границы
    if x >= 0 && y >= 0 && bottom < config.height && right < config.width

    // Проверяем совместимость
    if fits(word, point.vertical, x, y)

  } yield {
    copyWithWord(x, y, point.vertical, word)
  }

  puzzles.toArray
}
```

**Файл:** `Puzzle.scala:86-100`

### Пошаговое объяснение

1. **Группировка по символам:** Для слова "КОШКА" получаем: `{'К' -> [0,4], 'О' -> [1], 'Ш' -> [2], 'А' -> [3]}`

2. **Поиск пересечений:** Для каждого символа ищем, где он уже есть в кроссворде

3. **Расчёт координат:** Если буква 'О' слова находится на индексе 1, а в кроссворде 'О' на позиции (5, 3), то начало слова будет (4, 3)

4. **Проверка границ:** Слово должно полностью помещаться в сетку

5. **Проверка совместимости:** Слово не должно создавать конфликтов

---

## Проверка совместимости

Метод `fits` проверяет, может ли слово быть размещено в заданной позиции.

```scala
private def fits(word: String, vertical: Boolean, x: Int, y: Int): Boolean = {
  var connect = false

  val fitsAll = (0 until word.length).forall { index =>
    val (locX, locY) = if (vertical) (x, y + index) else (x + index, y)
    val existingChar = getChar(locX, locY)

    val same = existingChar == word(index)   // Символы совпадают
    val isEmpty = existingChar == ' '         // Ячейка пуста

    if (same) connect = true

    (same || isEmpty) && {
      if (isEmpty) {
        // Пустая ячейка не должна иметь соседних букв
        // (кроме направления самого слова)
        if (vertical) {
          !hasChar(locX - 1, locY) && !hasChar(locX + 1, locY)
        } else {
          !hasChar(locX, locY - 1) && !hasChar(locX, locY + 1)
        }
      } else {
        true
      }
    }
  }

  fitsAll && connect && {
    // Перед и после слова не должно быть букв
    if (vertical) {
      !hasChar(x, y - 1) && !hasChar(x, y + word.length)
    } else {
      !hasChar(x - 1, y) && !hasChar(x + word.length, y)
    }
  }
}
```

**Файл:** `Puzzle.scala:104-125`

### Правила размещения

| Правило | Описание |
|---------|----------|
| Совпадение символов | Каждая ячейка либо пуста, либо содержит тот же символ |
| Пересечение | Хотя бы одна буква должна совпадать с уже размещённым словом |
| Изоляция | Новые буквы не должны соседствовать с чужими буквами (по перпендикулярному направлению) |
| Концы слова | Перед началом и после конца слова не должно быть букв |

### Визуализация правил

```
Корректно:                    Некорректно:
    К                             К
    О                             ОМ    ← соседняя буква
    Т                             Т
```

```
Корректно:                    Некорректно:
  СЛОВО                         СЛОВО
    О                               О
    Т                               Т
                                  ПОДАРОК ← слова сливаются
```

---

## Расчёт доступных позиций

Метод `positions` вычисляет, где в кроссворде можно разместить новые слова.

```scala
lazy val positions: Map[Char, Array[CharPoint]] = {
  val list = ListBuffer.empty[CharPoint]

  (0 until chars.length).foreach { index =>
    val char = chars(index)
    if (char != ' ') {
      val x = index % config.width
      val y = index / config.width

      // Проверка горизонтального размещения
      val canHorizontal = isEmpty(x + 1, y) && isEmpty(x - 1, y) &&
        ((isEmpty(x + 1, y + 1) && isEmpty(x + 1, y - 1) && x < config.width - 1) ||
         (isEmpty(x - 1, y + 1) && isEmpty(x - 1, y - 1) && x > 0))

      // Проверка вертикального размещения
      val canVertical = isEmpty(x, y + 1) && isEmpty(x, y - 1) &&
        ((isEmpty(x + 1, y + 1) && isEmpty(x - 1, y + 1) && y < config.height - 1) ||
         (isEmpty(x + 1, y - 1) && isEmpty(x - 1, y - 1) && y > 0))

      if (canHorizontal) list += CharPoint(char, x, y, vertical = false)
      else if (canVertical) list += CharPoint(char, x, y, vertical = true)
    }
  }

  list.toArray.groupBy(_.char)
}
```

**Файл:** `Puzzle.scala:34-53`

### Условия для точки пересечения

Буква может служить точкой пересечения, если:
1. С обеих сторон по перпендикулярному направлению — пустота
2. Есть пространство для размещения нового слова хотя бы в одном направлении

---

## Параллельная генерация

Для ускорения генерации используются Web Workers.

### Архитектура

```scala
object PuzzleGenerator {
  private val WORKER_COUNT = 4  // 4 параллельных worker'а
  private val workers = (0 until WORKER_COUNT).map(_ => new Worker("./js/worker.js"))
}
```

**Файл:** `PuzzleGenerator.scala:8-47`

### Процесс генерации

```
┌─────────────────┐
│   Главный поток │
└────────┬────────┘
         │ Отправка задачи
         ▼
┌────────┴────────┬────────┬────────┐
│   Worker 1      │ Worker 2│ Worker 3│ Worker 4│
│   1000 попыток  │ 1000    │ 1000    │ 1000    │
└────────┬────────┴────┬───┴────┬───┴────┬────┘
         │             │        │        │
         ▼             ▼        ▼        ▼
      Лучший 1      Лучший 2  Лучший 3  Лучший 4
         │             │        │        │
         └──────┬──────┴────────┴────────┘
                ▼
         Выбор лучшего из 4
```

### Код выбора лучшего варианта

```scala
private def createPuzzle(config: PuzzleConfig, mainWords: Seq[String]): Puzzle = {
  val puzzles = (0 until 1000).flatMap { _ =>
    val puzzles = Puzzle.generate(mainWords.head, mainWords.tail.toList, config)
    puzzles
  }.sortBy(-_.density)

  puzzles.head  // Возвращаем с максимальной плотностью
}
```

**Файл:** `WorkerMain.scala:29-48`

---

## Оценка качества кроссворда

### Плотность (Density)

Основная метрика качества — **плотность заполнения**.

```scala
def density: Double = {
  words.toSeq.map(_.replaceAll(" ", "").length).sum.toDouble /
    (config.width * config.height)
}
```

**Файл:** `Puzzle.scala:25-30`

**Формула:**
```
density = сумма_длин_слов / (ширина × высота)
```

**Пример:**
- Сетка 18×18 = 324 ячейки
- Размещено слов на 150 букв
- Плотность = 150 / 324 ≈ 0.46 (46%)

### Почему плотность важна

| Плотность | Качество | Описание |
|-----------|----------|----------|
| < 0.3     | Низкое   | Много пустого пространства |
| 0.3 - 0.5 | Среднее  | Типичный результат |
| > 0.5     | Высокое  | Компактный кроссворд |

---

## Аннотирование и визуализация

### Получение аннотаций (нумерация слов)

```scala
def getAnnotation: Map[Point, Seq[AnnotatedPoint]] = {
  var index = 0

  val points = for {
    x <- 0 until config.width
    y <- 0 until config.height
    nonEmpty = !isEmpty(x, y)
    vertical = isEmpty(x, y - 1)      // Начало вертикального слова?
    horizontal = isEmpty(x - 1, y)    // Начало горизонтального слова?
    if nonEmpty && (vertical || horizontal)
  } yield {
    val annotations = ListBuffer.empty[AnnotatedPoint]

    // Если это начало вертикального слова длиной > 1
    if (vertical && getWord(Point(x, y), true).length > 1) {
      index += 1
      annotations += AnnotatedPoint(index, vertical = true, getWord(...))
    }

    // Если это начало горизонтального слова длиной > 1
    if (horizontal && getWord(Point(x, y), false).length > 1) {
      index += 1
      annotations += AnnotatedPoint(index, vertical = false, getWord(...))
    }

    (Point(x, y), annotations.toSeq)
  }

  points.toMap
}
```

**Файл:** `Puzzle.scala:180-203`

### Правила нумерации

1. Обход сетки слева направо, сверху вниз
2. Номер присваивается в точке начала слова
3. Если в одной точке начинаются и горизонтальное, и вертикальное слово — они получают **один номер**
4. Слова длиной 1 буква не нумеруются

### Визуализация (SVG)

```scala
def renderPuzzle(puzzle: Puzzle, ...): String = {
  // Генерация SVG с ячейками, буквами и номерами
}
```

**Файл:** `HtmlRenderer.scala:12-60`

---

## Диаграмма полного процесса

```
┌─────────────────────────────────────────────────────────────┐
│                    ВВОД ПОЛЬЗОВАТЕЛЯ                        │
│                   (список слов + подсказки)                 │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. ПРЕДОБРАБОТКА                                           │
│     • Удаление спецсимволов                                 │
│     • Приведение к верхнему регистру                        │
│     • Сортировка по частоте символов                        │
│                                             (PuzzleWords.scala)│
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. ПАРАЛЛЕЛЬНАЯ ГЕНЕРАЦИЯ (4 Workers)                      │
│     Каждый worker выполняет 1000 попыток:                   │
│                                                             │
│     ┌─────────────────────────────────────────────────┐     │
│     │  2.1 Размещение первого слова                   │     │
│     │      (горизонтально или вертикально)            │     │
│     └───────────────────────┬─────────────────────────┘     │
│                             ▼                               │
│     ┌─────────────────────────────────────────────────┐     │
│     │  2.2 Для каждого оставшегося слова:             │     │
│     │      • Найти пересечения с размещёнными         │     │
│     │      • Проверить совместимость (fits)           │     │
│     │      • Выбрать случайный вариант                │     │
│     │      • Если не подходит — отложить              │     │
│     └───────────────────────┬─────────────────────────┘     │
│                             ▼                               │
│     ┌─────────────────────────────────────────────────┐     │
│     │  2.3 Вычисление плотности                       │     │
│     └───────────────────────┬─────────────────────────┘     │
│                                         (Puzzle.scala)      │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. ВЫБОР ЛУЧШЕГО ВАРИАНТА                                  │
│     • Сравнение по плотности                                │
│     • Возврат кроссворда с max density                      │
│                                        (WorkerMain.scala)   │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. АННОТИРОВАНИЕ                                           │
│     • Нумерация слов                                        │
│     • Сопоставление с подсказками                           │
│                                             (Puzzle.scala)  │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. ВИЗУАЛИЗАЦИЯ                                            │
│     • Генерация SVG сетки                                   │
│     • Отображение подсказок                                 │
│                                        (HtmlRenderer.scala) │
└─────────────────────────────────────────────────────────────┘
```

---

## Сложность алгоритма

| Операция | Сложность | Описание |
|----------|-----------|----------|
| Сортировка слов | O(n × m) | n слов, m — средняя длина |
| Одна попытка генерации | O(n × k) | n слов, k — количество позиций |
| Проверка fits | O(m) | m — длина слова |
| Расчёт positions | O(W × H) | размер сетки |
| Полная генерация | O(1000 × 4 × n × k) | с учётом параллелизма |

---

## Ограничения и особенности

1. **Случайность:** Алгоритм использует случайный выбор, поэтому результаты могут отличаться при каждом запуске

2. **Неполное размещение:** Не все слова гарантированно попадут в кроссворд — некоторые могут быть "не использованы"

3. **Оптимальность:** Алгоритм не гарантирует глобально оптимальное решение, но находит хорошее за разумное время

4. **Режим обёртывания:** При `wrapping = true` кроссворд "замыкается" по горизонтали (цилиндрическая топология)

---

## Файлы исходного кода

| Файл | Описание |
|------|----------|
| `src/main/scala/com/papauschek/puzzle/Puzzle.scala` | Основной класс и алгоритмы |
| `src/main/scala/com/papauschek/puzzle/PuzzleWords.scala` | Сортировка слов |
| `src/main/scala/com/papauschek/puzzle/PuzzleConfig.scala` | Конфигурация |
| `src/main/scala/com/papauschek/puzzle/Point.scala` | Вспомогательные структуры |
| `src/main/scala/com/papauschek/ui/PuzzleGenerator.scala` | Параллельная генерация |
| `src/main/scala/com/papauschek/worker/WorkerMain.scala` | Логика Worker |
| `src/main/scala/com/papauschek/ui/HtmlRenderer.scala` | Визуализация |
