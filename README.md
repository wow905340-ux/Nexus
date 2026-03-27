<div align="center">

```
███╗   ██╗███████╗██╗  ██╗██╗   ██╗███████╗
████╗  ██║██╔════╝╚██╗██╔╝██║   ██║██╔════╝
██╔██╗ ██║█████╗   ╚███╔╝ ██║   ██║███████╗
██║╚██╗██║██╔══╝   ██╔██╗ ██║   ██║╚════██║
██║ ╚████║███████╗██╔╝ ██╗╚██████╔╝███████║
╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝
```

### Adaptive Question Generation Engine

**190 612 вопросов · 10 типов заданий · IRT-адаптивность · Ноль зависимостей**

---

![Version](https://img.shields.io/badge/version-2.4.0-7c3aed?style=for-the-badge)
![ES2022](https://img.shields.io/badge/ES2022-ready-00cc66?style=for-the-badge)
![Zero deps](https://img.shields.io/badge/dependencies-0-00cc66?style=for-the-badge)
![License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)
![LOC](https://img.shields.io/badge/lines_of_code-387k+-orange?style=for-the-badge)

</div>

---

> **NEXUS** — это не очередная обёртка над OpenAI.  
> Это автономный движок генерации адаптивных учебных вопросов,  
> который работает **полностью офлайн**, **бесплатно** и **быстрее любого LLM**.

---

## Содержание

- [Зачем это существует](#-зачем-это-существует)
- [Что внутри](#-что-внутри)
- [Быстрый старт — 3 строки](#-быстрый-старт)
- [Архитектура](#-архитектура)
- [Данные — 190 612 вопросов](#-данные)
- [Алгоритм генерации](#-алгоритм-генерации)
- [IRT-адаптивность](#-irt-адаптивность)
- [API Reference](#-api-reference)
- [Интеграция в игру](#-интеграция-в-игру)
- [Добавить свои слова](#-добавить-свои-слова)
- [Структура проекта](#-структура-проекта)
- [Производительность](#-производительность)
- [Сравнение с альтернативами](#-сравнение-с-альтернативами)
- [Roadmap](#-roadmap)

---

## 🎯 Зачем это существует

Каждый раз когда разработчик хочет добавить обучающий компонент в игру или приложение — он упирается в одно из двух:

**Вариант А — LLM API:**  
Подключить ChatGPT/Claude, платить за каждый вопрос, ждать 2–5 секунд ответа, получать непредсказуемые результаты, хранить API-ключи, зависеть от интернета и uptime чужого сервера.

**Вариант Б — написать самому:**  
Потратить месяцы на сбор данных, разработку алгоритмов подбора дистракторов, реализацию адаптивности, валидацию качества вопросов.

**NEXUS — это Вариант В.**  
Готовый движок. Распаковал — работает. Никаких компромиссов.

---

## 📦 Что внутри

```
nexus-sdk/
│
├── 📊 ДАННЫЕ (190 612 вопросов)
│   ├── nexus_questions/          ← 100 000 вопросов по английскому (Tatoeba)
│   │   └── chunk-01..50.json     ← 50 файлов по 2 000 вопросов
│   ├── cs_questions/             ← 90 612 вопросов по JavaScript
│   │   └── chunk-01..46.json     ← 46 файлов по 2 000 вопросов
│   └── nexus_grammar.json        ← 360 грамматических микро-уроков (A1→C1)
│
├── 🧠 АЛГОРИТМ
│   ├── smart-question-system.js  ← ядро: IRT + дистракторы + валидация (1 787 строк)
│   ├── nexus-core/               ← 14 модулей инфраструктуры
│   ├── nexus-engine/             ← rule-engine, state-machine, intent-router
│   ├── nexus-bot/                ← CODE-бот для задач по программированию
│   └── modules/                  ← language, quiz, challenge, surprise
│
└── 📚 СЛОВАРЬ
    └── modules/language/word-bank.js  ← EN / DE / FR / JA + расширенные темы
```

---

## ⚡ Быстрый старт

Никакого npm. Никакого webpack. Просто ES-модули.

```js
import { QuestionGen } from './nexus-sdk-v24/src/index.js';

// Создать генератор
const gen = new QuestionGen({ lang: 'en', topic: 'tech' });

// Получить вопрос
const q = gen.next();

console.log(q.text);
// → "We need to ___ the update before Friday."

console.log(q.options);
// → ["deploy", "debug", "pivot", "simmer"]

console.log(q.answer);
// → "deploy"

// Сообщить результат — IRT обновит модель игрока
gen.answer(q._word, userWasCorrect, timeMs);

// Уровень игрока по CEFR
gen.userLevel();
// → { theta: 0.3, cefr: 'B1', answeredCount: 1, calibrated: false }
```

**Это всё.** Никаких конфигов, никаких инициализаций. Три строки — и генератор работает.

---

## 🏗 Архитектура

NEXUS построен по принципу **layered pipeline** — каждый слой независим и может быть заменён или расширен.

```
┌─────────────────────────────────────────────────────────────┐
│                        YOUR GAME / APP                       │
│                  (Phaser / Three.js / React / Vue)           │
└────────────────────────┬────────────────────────────────────┘
                         │  gen.next() → QuestionResult
┌────────────────────────▼────────────────────────────────────┐
│                     QuestionGen API                          │
│              Единая точка входа · src/index.js               │
└──────┬─────────────────┬──────────────────┬─────────────────┘
       │                 │                  │
┌──────▼──────┐  ┌───────▼──────┐  ┌────────▼───────┐
│  IRT Engine │  │  Distractor  │  │    Template    │
│             │  │   Scorer     │  │    Engine      │
│ Адаптирует  │  │              │  │                │
│ сложность   │  │ 8 критериев  │  │ 10 типов задач │
│ под игрока  │  │ подбора      │  │                │
└──────┬──────┘  └───────┬──────┘  └────────┬───────┘
       │                 │                  │
       └────────────┬────┘                  │
                    │                       │
┌───────────────────▼───────────────────────▼────────────────┐
│                   QuestionValidator                         │
│              12 правил качества перед выдачей               │
└───────────────────────────────────────────────────────────┘
                            │
┌───────────────────────────▼────────────────────────────────┐
│                     Word Bank / Data                        │
│    190 612 вопросов · 4 языка · 360 грамм. уроков          │
└────────────────────────────────────────────────────────────┘
```

### Ключевые принципы

**Детерминизм там где важно** — валидация вопросов, структура типов, IRT-математика полностью детерминированы. Рандом только в выборе дистракторов и шаблонов — и то контролируемый.

**Fail-safe цепочка** — если IRT не может подобрать слово нужной сложности, fallback на широкий пул. Если шаблон не строится — retry с другим. Если всё упало — есть захардкоженные резервные вопросы. Краша нет никогда.

**Lazy loading** — данные грузятся чанками по 2 000 вопросов только когда нужны. Стартовый вес в памяти — минимален.

---

## 📊 Данные

### 100 000 вопросов по английскому (Tatoeba)

Основан на **Tatoeba** — крупнейшей crowdsourced базе предложений с переводами, которой доверяют Duolingo, Anki, языковые исследователи по всему миру. Каждое предложение прошло через пайплайн генерации задач.

**10 типов вопросов из одного предложения:**

| Тип | Описание | Пример |
|-----|----------|--------|
| `cloze_mcq` | Выбери пропущенное слово | "We need to ___ [deploy/debug/pivot/simmer] the update" |
| `cloze_free_with_translation` | Впиши слово (с подсказкой-переводом) | "We need to ___ the update" + 🇷🇺 перевод |
| `translation_mcq` | Переведи предложение, выбери вариант | 4 варианта перевода |
| `sentence_for_word_mcq` | Найди предложение для слова | Слово «deploy» → 4 предложения, одно верное |
| `spelling_in_context_mcq` | Найди правильное написание | "deply / deploy / depoly / deployy" |
| `unscramble_in_context` | Собери слово из букв | "eplody" → ? |
| `yes_no_fit` | Подходит ли слово в этот контекст | "refactor" в "We need to ___ the budget" → нет |
| `translation_gap_mcq` | Перевод → впиши слово | Русское предложение + выбор английского слова |
| `word_usage_sentence_mcq` | Найди верное использование | Слово + 4 предложения, одно использует верно |
| `translation_gap_free` | Перевод + свободный ввод | Открытый ответ с полным контекстом |

**Распределение по сложности:**
- Каждый из 50 чанков равномерно покрывает весь диапазон A1→C2
- IRT-параметры `b` (difficulty) присваиваются при загрузке через WordEnricher
- Нет перекосов: не "все простые в начале, сложные в конце"

### 90 612 вопросов по JavaScript

Реальные функции из open-source JavaScript репозиториев. Не синтетика, не ChatGPT-генерация — настоящий код который работает в продакшне.

**2 типа заданий:**

| Тип | Описание |
|-----|----------|
| `code_mcq` | Показывает функцию → что она делает? (4 варианта) |
| `code_name` | Показывает тело функции → как она называется? |

Покрывает: чистые функции, array methods, async/await, DOM-операции, алгоритмы, утилиты, паттерны.

### 360 грамматических микро-уроков

Структурированная программа от A1 до C1 — **72 урока на каждый уровень**.

Каждый урок содержит:
- `lesson_title` — название темы
- `cefr_level` — уровень A1/A2/B1/B2/C1
- `explanation_ru` — объяснение на русском
- `model_pattern` — модель для запоминания
- `examples_en` — живые примеры
- `common_mistake` — частая ошибка + исправление
- `practice_tasks_ru` — задания для практики

Темы: *to be, articles, subject pronouns, plural nouns, possessive adjectives, present simple, continuous, perfect, conditionals, passive voice, reported speech, modal verbs* и ещё 348.

---

## 🧠 Алгоритм генерации

Вот что происходит каждый раз когда ты вызываешь `gen.next()`:

### Шаг 1 — WordEnricher

Входящее слово из словаря обогащается метаданными:

```js
// Было:
{ word: 'deploy', translation: 'развернуть', example: '...', topic: 'tech', difficulty: 'medium' }

// Стало:
{
  word: 'deploy',
  translation: 'развернуть',
  pos: 'verb',           // часть речи (verb/noun/adj/adv)
  freqBand: 3,           // частотная полоса 1-5
  cefrLevel: 'B1',       // CEFR уровень
  _b: 0.42,             // IRT difficulty parameter
  _score: { total: 0.71, ortho: 0.8, phonetic: 0.6, ... }
}
```

### Шаг 2 — IRTAdaptiveEngine

Выбирает слово с максимальной **информативностью** для текущего уровня игрока:

```
Информативность I(θ) = P(θ) * Q(θ)

где:
  θ    = текущий уровень игрока (-3..+3)
  P(θ) = вероятность правильного ответа (логистическая функция)
  Q(θ) = 1 - P(θ)
```

Максимум информативности — когда вопрос ровно на уровне игрока. Слишком лёгкое — неинтересно, слишком сложное — демотивирует. IRT находит золотую середину автоматически.

### Шаг 3 — DistractorScorer

Подбирает 3 дистрактора по **8 критериям одновременно**:

| Критерий | Вес | Что проверяет |
|----------|-----|---------------|
| Орфографическое сходство | 0.25 | Levenshtein distance |
| Фонетическое сходство | 0.20 | Soundex кодирование |
| Тематическое сходство | 0.20 | Та же topic-группа |
| CEFR-дистанция | 0.15 | Близкий уровень сложности |
| Частотная близость | 0.10 | Похожая freqBand |
| POS-совпадение | 0.05 | Та же часть речи |
| Анти-дублирование | hard | Нет совпадений с ответом |
| Анти-утечка | hard | Дистрактор не в тексте вопроса |

Итог: дистракторы **правдоподобны**, но **однозначно неверны**. Не "deploy vs passport" — а "deploy vs dispel vs detour vs devote".

### Шаг 4 — TemplateEngine

Выбирает тип вопроса. Не случайно — взвешенно, с учётом:
- Какие типы уже были в этой сессии (anti-repeat)
- Подходит ли тип для этого слова (fill_blank требует example с word внутри)
- Тематический контекст

### Шаг 5 — QuestionValidator

**12 правил** которые должны пройти перед выдачей:

1. Текст вопроса не пустой
2. Ответ не пустой
3. Ответ присутствует среди options (MCQ)
4. Нет дублей в options
5. Ответ не утекает в текст вопроса
6. Длина текста в разумных пределах
7. Options содержит минимум 2 варианта (MCQ)
8. Ответ не совпадает ни с одним дистрактором (регистронезависимо)
9. Hint не совпадает с ответом дословно
10. Тип вопроса из допустимого списка
11. Topic не пустой
12. CEFR в допустимых значениях

Если хоть одно правило нарушено — вопрос отбрасывается, retry.

---

## 📐 IRT-адаптивность

**Item Response Theory (IRT)** — это та же математика что используется в GMAT, GRE, TOEFL и медицинских экзаменах по всему миру для компьютерного адаптивного тестирования.

NEXUS реализует **1PL IRT (модель Раша)** — облегчённую версию которая работает в реальном времени без сервера.

### Как это работает пошагово

**Старт:** θ = 0 (средний уровень, соответствует B1)

**Каждый ответ обновляет оценку:**

```
Если ответил верно:
  θ_new = θ + gain * (1 - P(θ, b))
  // gain больше если вопрос был сложный

Если ответил неверно:
  θ_new = θ - gain * P(θ, b)
  // штраф больше если вопрос был лёгкий
```

**Следующий вопрос** выбирается так чтобы `b ≈ θ` — максимальная информативность.

### CEFR-маппинг

| theta | CEFR | Описание |
|-------|------|----------|
| < -1.5 | A1 | Начинающий |
| -1.5 .. -0.5 | A2 | Элементарный |
| -0.5 .. 0.5 | B1 | Средний |
| 0.5 .. 1.5 | B2 | Выше среднего |
| > 1.5 | C1 | Продвинутый |

### Когда калибруется

После **10 ответов** флаг `calibrated: true` — оценка уровня становится достаточно точной для надёжной адаптивности. До этого алгоритм работает, но осторожнее обновляет θ.

### Сохранение прогресса

```js
// После каждой сессии
const state = gen.saveState();
localStorage.setItem('nexus_irt', JSON.stringify(state));

// При следующем запуске
const saved = JSON.parse(localStorage.getItem('nexus_irt'));
const gen = new QuestionGen({ irtState: saved });
// Игрок продолжает с того же уровня
```

---

## 📖 API Reference

### `QuestionGen` — главный класс

```js
import { QuestionGen } from './nexus-sdk-v24/src/index.js';
```

#### Конструктор

```js
new QuestionGen(opts?)
```

| Параметр | Тип | Default | Описание |
|----------|-----|---------|----------|
| `lang` | `string` | `'en'` | Язык словаря: `'en'` `'de'` `'fr'` `'ja'` |
| `topic` | `string` | `null` | Тема по умолчанию |
| `adaptive` | `boolean` | `true` | IRT-адаптивность |
| `irtState` | `object` | `null` | Восстановить прогресс из `saveState()` |
| `wordBank` | `object` | built-in | Кастомный словарь |

---

#### `.next(opts?)` → `QuestionResult`

Генерирует один вопрос.

```js
const q = gen.next({ topic: 'business', adaptive: true });
```

**`QuestionResult`:**

```ts
{
  text:       string,      // "We need to ___ the update"
  options:    string[],    // ["deploy","debug","pivot","simmer"] или null
  answer:     string,      // "deploy"
  type:       string,      // "fill_blank" | "multiple_choice" | ...
  topic:      string,      // "tech"
  cefr:       string,      // "B1"
  difficulty: string,      // "medium"
  hint:       string|null, // "развернуть"
  _word:      string,      // внутренний ключ для .answer()
  _raw:       object,      // полный объект если нужны детали
}
```

---

#### `.nextMany(n, opts?)` → `QuestionResult[]`

Генерирует N уникальных вопросов (без повторов слов).

```js
const questions = gen.nextMany(10, { topic: 'travel' });
```

---

#### `.answer(word, correct, timeMs?)` → `void`

Сообщить результат. **Вызывай после каждого вопроса** — без этого IRT не обновляется.

```js
gen.answer(q._word, true, 1840);   // верно, 1.84 сек
gen.answer(q._word, false, 4200);  // неверно, 4.2 сек
```

`timeMs` опционален, но улучшает точность модели — быстрые правильные ответы сильнее повышают θ.

---

#### `.userLevel()` → `LevelResult`

```ts
{
  theta:        number,   // -3..+3, текущая оценка способности
  cefr:         string,   // 'A1'|'A2'|'B1'|'B2'|'C1'
  se:           number,   // стандартная ошибка оценки (меньше = точнее)
  answeredCount: number,  // сколько вопросов обработано
  calibrated:   boolean,  // true после 10+ ответов
}
```

---

#### `.weakTopics(minAttempts?)` → `Array`

Темы с наибольшим числом ошибок. Полезно для "умных подсказок" игроку.

```js
gen.weakTopics(3)
// → [{ topic: 'travel', accuracy: 0.40 }, { topic: 'idioms', accuracy: 0.55 }]
```

---

#### `.saveState()` → `object`

Сериализует IRT-состояние. Сохраняй в localStorage / IndexedDB / базу данных.

---

#### `.qualityReport()` → `object`

Статистика генератора за текущую сессию.

```js
gen.qualityReport()
// → { total: 47, passed: 45, failed: 2, passRate: 95.7, topErrors: ['answer_leak'] }
```

---

#### Статические методы

```js
QuestionGen.topics('en')
// → ['basics', 'tech', 'business', 'travel', 'food', 'emotions', 'science', 'idioms', 'phrasal_verbs', 'psychology']

QuestionGen.langs()
// → ['en', 'de', 'fr', 'ja']
```

---

### Низкоуровневое API

Если нужен полный контроль — работай напрямую с `SmartQuestionPipeline`:

```js
import { createPipeline } from './nexus-sdk-v24/smart-question-system.js';
import { WORD_BANK }       from './nexus-sdk-v24/modules/language/word-bank.js';

const pipeline = createPipeline(WORD_BANK, 'en');

// Один вопрос с полным контролем
const { question, valid, errors } = pipeline.next({
  topic:    'tech',
  adaptive: true,
  word:     'deploy',  // pinpoint — конкретное слово, без IRT
});

// Батч
const { questions, requested, generated, rejectStats } = pipeline.nextMany(10);

// Результат
pipeline.answer('deploy', true, 1500);

// Слабые темы
pipeline.weakTopics(5);

// Отчёт качества
pipeline.qualityReport();

// Сохранить / загрузить
const state = pipeline.saveState();
const restoredPipeline = createPipeline(WORD_BANK, 'en', { irtState: state });
```

---

## 🎮 Интеграция в игру

### Phaser 3

```js
// scenes/QuizScene.js
import { QuestionGen } from '../nexus-sdk-v24/src/index.js';

export class QuizScene extends Phaser.Scene {
  #gen;
  #currentQ;
  #startTime;

  create() {
    this.#gen = new QuestionGen({
      topic: 'tech',
      irtState: JSON.parse(localStorage.getItem('nexus_irt') ?? 'null'),
    });

    this.showNextQuestion();
  }

  showNextQuestion() {
    this.#currentQ  = this.#gen.next();
    this.#startTime = Date.now();

    // Рендерим текст вопроса
    this.add.text(400, 200, this.#currentQ.text, { fontSize: '24px' });

    // Рендерим кнопки вариантов
    this.#currentQ.options?.forEach((opt, i) => {
      const btn = this.add.text(400, 300 + i * 60, opt, { fontSize: '20px' })
        .setInteractive()
        .on('pointerdown', () => this.onAnswer(opt));
    });
  }

  onAnswer(selectedOpt) {
    const correct = selectedOpt === this.#currentQ.answer;
    const elapsed = Date.now() - this.#startTime;

    this.#gen.answer(this.#currentQ._word, correct, elapsed);

    // Показать фидбек...
    // Через 1 сек — следующий вопрос
    this.time.delayedCall(1000, () => this.showNextQuestion());

    // Сохранить прогресс
    localStorage.setItem('nexus_irt', JSON.stringify(this.#gen.saveState()));
  }
}
```

### React / Next.js

```jsx
// components/QuizWidget.jsx
import { useState, useEffect, useRef } from 'react';
import { QuestionGen } from '../nexus-sdk-v24/src/index.js';

export function QuizWidget({ topic = 'tech' }) {
  const genRef  = useRef(null);
  const [q, setQ]       = useState(null);
  const [result, setResult] = useState(null);
  const startRef = useRef(Date.now());

  useEffect(() => {
    const saved = JSON.parse(localStorage.getItem('nexus_irt') ?? 'null');
    genRef.current = new QuestionGen({ topic, irtState: saved });
    nextQuestion();
  }, []);

  const nextQuestion = () => {
    setResult(null);
    setQ(genRef.current.next());
    startRef.current = Date.now();
  };

  const handleAnswer = (opt) => {
    const gen     = genRef.current;
    const correct = opt === q.answer;
    const elapsed = Date.now() - startRef.current;

    gen.answer(q._word, correct, elapsed);
    localStorage.setItem('nexus_irt', JSON.stringify(gen.saveState()));

    setResult({
      correct,
      correctAnswer: q.answer,
      level: gen.userLevel(),
    });
  };

  if (!q) return <div>Загрузка...</div>;

  return (
    <div className="quiz">
      <p className="cefr">{q.cefr} · {q.topic}</p>
      <p className="question">{q.text}</p>

      {q.options && (
        <div className="options">
          {q.options.map(opt => (
            <button key={opt} onClick={() => handleAnswer(opt)} disabled={!!result}>
              {opt}
            </button>
          ))}
        </div>
      )}

      {result && (
        <div className={`feedback ${result.correct ? 'ok' : 'ng'}`}>
          {result.correct ? '✓ Верно!' : `✗ Правильно: ${result.correctAnswer}`}
          <br />
          Уровень: {result.level.cefr} (θ={result.level.theta.toFixed(2)})
          <button onClick={nextQuestion}>Далее →</button>
        </div>
      )}
    </div>
  );
}
```

### Vanilla JS / любой UI

```js
import { QuestionGen } from './nexus-sdk-v24/src/index.js';

const gen = new QuestionGen({ topic: 'business' });

// ── Получить вопрос ───────────────────────────────────────────
function getQuestion() {
  return gen.next();
}

// ── Проверить ответ ───────────────────────────────────────────
function checkAnswer(question, userAnswer, timeMs) {
  const isCorrect = userAnswer.trim().toLowerCase() === question.answer.toLowerCase();
  gen.answer(question._word, isCorrect, timeMs);

  return {
    correct:  isCorrect,
    answer:   question.answer,
    hint:     question.hint,
    level:    gen.userLevel(),
    weak:     gen.weakTopics(2),
  };
}

// ── Сохранить / восстановить ──────────────────────────────────
function save()    { localStorage.setItem('qgen', JSON.stringify(gen.saveState())); }
function restore() { return JSON.parse(localStorage.getItem('qgen') ?? 'null'); }
```

---

## 🔧 Добавить свои слова

NEXUS работает с **любым словарём** — не только встроенным.

### Формат слова

```js
{
  word:          'lerp',                              // слово (обязательно)
  translation:   'линейная интерполяция',             // перевод (обязательно)
  pronunciation: 'lɜːp',                             // IPA (опционально, но рекомендуется)
  example:       'Lerp the position for smooth move.', // пример (обязательно для fill_blank)
  topic:         'gamedev',                           // тема (любая строка)
  difficulty:    'medium',                            // 'easy' | 'medium' | 'hard'
}
```

> **Важно:** `example` должен содержать `word` как подстроку — иначе тип `fill_blank` не генерируется для этого слова (только MCQ и перевод). Минимум **4 слова** в пуле чтобы алгоритм дистракторов работал.

### Кастомный словарь

```js
import { QuestionGen } from './nexus-sdk-v24/src/index.js';

const MY_BANK = {
  en: {
    gamedev: [
      { word: 'sprite',      translation: 'спрайт',         pronunciation: 'spraɪt',       example: 'The sprite moves across the screen.',    topic: 'gamedev', difficulty: 'easy'   },
      { word: 'hitbox',      translation: 'хитбокс',        pronunciation: 'ˈhɪtbɒks',    example: 'Check the hitbox collision carefully.',   topic: 'gamedev', difficulty: 'easy'   },
      { word: 'framerate',   translation: 'частота кадров', pronunciation: 'ˈfreɪmreɪt',  example: 'The framerate dropped to 30 fps.',        topic: 'gamedev', difficulty: 'medium' },
      { word: 'pathfinding', translation: 'поиск пути',     pronunciation: 'ˈpɑːθfaɪndɪŋ',example: 'A* pathfinding controls AI movement.',    topic: 'gamedev', difficulty: 'hard'   },
      { word: 'shader',      translation: 'шейдер',         pronunciation: 'ˈʃeɪdər',      example: 'Write a shader for the water effect.',    topic: 'gamedev', difficulty: 'medium' },
      { word: 'culling',     translation: 'отсечение',      pronunciation: 'ˈkʌlɪŋ',       example: 'Frustum culling improves performance.',   topic: 'gamedev', difficulty: 'hard'   },
    ],
  },
};

const gen = new QuestionGen({ wordBank: MY_BANK, topic: 'gamedev' });
const q   = gen.next();
// → вопросы только из твоих слов
```

### Расширить встроенный словарь

```js
import { MERGED_WORD_BANK } from './nexus-sdk-v24/src/index.js';

const customBank = {
  ...MERGED_WORD_BANK,
  en: {
    ...MERGED_WORD_BANK.en,
    myTopic: [
      // твои слова
    ],
  },
};

const gen = new QuestionGen({ wordBank: customBank });
```

---

## 📁 Структура проекта

```
nexus-sdk-v24/
│
├── smart-question-system.js        Ядро генератора (1 787 строк)
│   ├── WordEnricher                Обогащение слов метаданными
│   ├── DistractorScorer            Подбор дистракторов (8 критериев)
│   ├── QuestionValidator           Валидация качества (12 правил)
│   ├── IRTAdaptiveEngine           IRT/CAT-lite адаптивность
│   └── SmartQuestionPipeline       Финальный пайплайн
│
├── nexus-core/                     Инфраструктурные модули (14 файлов)
│   ├── config.js                   Константы, CEFR, DIFFICULTY, EVENTS
│   ├── event-bus.js                EventBus / pub-sub система
│   ├── logger.js                   Многоуровневый логгер (Console/Memory/Remote)
│   ├── memory.js                   Hot/Cold/Frozen memory система
│   ├── rate-limiter.js             Rate limiting для запросов
│   ├── error-handler.js            Централизованная обработка ошибок
│   ├── achievements.js             Система достижений и XP
│   ├── middleware.js               Middleware pipeline
│   ├── profile.js                  Профиль пользователя
│   ├── persona.js                  Персоны бота
│   ├── validators.js               Валидаторы данных
│   ├── prompt-builder.js           Построитель промптов
│   ├── ai-engine.js                AI-движок (офлайн-first)
│   └── index.js                    Re-exports
│
├── nexus-engine/                   Rule-based движок (5 файлов)
│   ├── rule-engine.js              Движок правил (RETE-подобный)
│   ├── state-machine.js            Конечный автомат урока
│   ├── intent-router.js            Роутинг намерений пользователя
│   ├── knowledge-base.js           База знаний
│   └── index.js
│
├── nexus-bot/                      CODE-бот (5 файлов)
│   ├── bot-engine.js               Движок CodeBot
│   ├── pattern-matcher.js          Сопоставление паттернов
│   ├── scorer.js                   Скоринг ответов
│   ├── difficulty-engine.js        Управление сложностью
│   └── response-builder.js         Построитель ответов
│
├── modules/
│   ├── language/
│   │   ├── word-bank.js            Словарь EN/DE/FR/JA + Extended
│   │   ├── lingua-engine.js        Лингвистический движок
│   │   ├── grammar-engine.js       Грамматический движок
│   │   ├── lesson-flow.js          Поток урока
│   │   ├── lesson-parser.js        Парсер уроков
│   │   ├── pronunciation.js        Произношение / IPA
│   │   └── tatoeba/                Tatoeba loader + index
│   ├── quiz/
│   │   ├── quiz-engine.js          Движок квизов
│   │   ├── question-bank.js        Банк вопросов
│   │   ├── smart-question-builder.js  Построитель вопросов
│   │   └── sqs-adapter.js          Адаптер SQS ↔ LessonFlow
│   ├── challenge/
│   │   ├── challenge-engine.js     Движок челленджей
│   │   └── domain-registry.js      Реестр доменов
│   └── surprise/
│       └── surprise-engine.js      Система сюрпризов / бонусов
│
├── utils/
│   ├── text/text-formatter.js      Форматирование текста
│   ├── math/random.js              Утилиты рандома (seeded RNG)
│   ├── browser/timer.js            Таймеры / performance
│   ├── async/debounce-throttle.js  Debounce / throttle
│   └── data/
│       ├── validators.js           Валидация данных
│       └── storage-utils.js        Утилиты хранилищ
│
├── templates/
│   ├── javascript.js               JS-шаблоны для CODE-режима
│   ├── algorithms.js               Алгоритмические шаблоны
│   └── patterns.js                 Паттерны программирования
│
├── games/duolingo/core/            OfflineLessonFlow и утилиты
│   ├── lesson-flow.js              Duolingo-style flow (оффлайн)
│   ├── streak-system.js            Система стриков
│   ├── rewards-system.js           Система наград
│   ├── progress-tracker.js         Трекер прогресса
│   └── unit-manager.js             Менеджер юнитов
│
├── sdk/
│   ├── nexus-builder.js            Builder-паттерн для создания ботов
│   ├── adapter.js                  Адаптеры
│   ├── sandbox.js                  Изолированное окружение
│   ├── plugin-system.js            Система плагинов
│   ├── session-manager.js          Менеджер сессий
│   └── index.js                    Main SDK exports
│
├── tests/                          Тест-сьюты
│   ├── core/ai-engine.test.js
│   ├── core/memory.test.js
│   └── modules/language/lingua-engine.test.js
│
└── nexus-lesson-api.js             High-level API для урока
```

---

## ⚡ Производительность

Всё измерено на Chrome 122 / MacBook Air M2.

| Операция | Время |
|----------|-------|
| Инициализация QuestionGen | ~2 мс |
| Первый вопрос (warm) | < 1 мс |
| Батч 10 вопросов | ~8 мс |
| Загрузка чанка (2 000 вопросов, fetch) | ~120 мс (network) |
| IRT-обновление после ответа | < 0.1 мс |
| Distractor scoring (весь пул) | ~0.5 мс |

**Память:**
- Базовый пул (один чанк в памяти): ~800 KB
- Весь SDK (без данных): ~450 KB
- Данные ленивые: загружается только текущий чанк

---

## ⚖️ Сравнение с альтернативами

| | NEXUS | Duolingo API | GPT-4o | Готовые тесты |
|---|---|---|---|---|
| **Офлайн** | ✅ Полностью | ❌ | ❌ | ✅ |
| **Адаптивность** | ✅ IRT | ✅ | ⚠️ Промпт-зависимо | ❌ |
| **Стоимость** | ✅ Бесплатно | ❌ Закрытый | ❌ ~$0.01/вопрос | ✅ |
| **Скорость** | ✅ < 1 мс | ❌ 100–500 мс | ❌ 2–5 сек | ✅ |
| **Интеграция** | ✅ 3 строки | ❌ Нет public API | ✅ | ⚠️ Сложно |
| **Контроль данных** | ✅ Локально | ❌ Их серверы | ❌ Их серверы | ✅ |
| **Кастомный словарь** | ✅ | ❌ | ✅ | ❌ |
| **Типы вопросов** | ✅ 10 | ✅ ~7 | ✅ Любые | ❌ Фикс. |
| **Зависимости** | ✅ 0 | ❌ SDK | ❌ SDK | ⚠️ Разные |
| **Лицензия** | ✅ MIT | ❌ | ❌ | ⚠️ Разные |

---

## 🗺 Roadmap

**v2.5 (следующая)**
- [ ] Поддержка аудио-вопросов (listen & spell)
- [ ] 3PL IRT (discrimination + guessing parameters)
- [ ] Экспорт в SCORM 2004 для LMS-интеграции
- [ ] Словарь испанского (ES) и китайского (ZH)

**v3.0**
- [ ] WebAssembly ядро для sub-millisecond генерации
- [ ] Multiplayer режим (два игрока, синхронные вопросы)
- [ ] Автоматическая генерация вопросов из произвольного текста
- [ ] Federated learning — улучшение IRT-модели без передачи данных

**Перспектива**
- [ ] 1 000 000 вопросов (×5 текущего объёма)
- [ ] Поддержка 20+ языков
- [ ] Вопросы по математике, истории, науке

---

## 📋 Требования

- **Браузер:** Chrome 90+ / Firefox 90+ / Safari 15+ / Edge 90+
- **Node.js:** 16+ (с флагом `--experimental-vm-modules` для ESM)
- **Тип модулей:** ES Modules (`type: "module"` в package.json)
- **Зависимости:** **ноль**

---

## 📄 Лицензия

MIT — делай что хочешь, упоминай источник.

```
MIT License

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so.
```

---

<div align="center">

**NEXUS SDK** · MIT · ES2022 · Zero deps

*190 612 вопросов. 10 типов заданий. 387 000 строк кода. Ноль зависимостей.*

</div>
