# mood_diaries_prepilot_public
# Adaptive Diary UI — демонстрация frontend-архитектуры

> Безопасный публичный фрагмент приложения. Тексты приложения, production-endpoints, внутренние идентификаторы, backend-логика, приватные ресурсы и бизнес-правила намеренно исключены или обобщены.

## Обзор

Проект демонстрирует модульную frontend-архитектуру для нескольких mobile-first сценариев дневника:

- полный дневник состояния за день;
- короткий утренний дневник;
- быстрый check-in состояния;
- страницы профиля и настроек;
- управление видимостью вопросов данными с сервера;
- адаптивные формулировки в зависимости от пользовательских настроек;
- переиспользуемый рендеринг полей через фабрику;
- локальное сохранение черновиков;
- клиентская валидация и нормализация payload.

В публичном репозитории размещены только упрощённые примеры и материалы интерфейса.

## Превью интерфейсов

### Дневник за день
https://github.com/user-attachments/assets/575bdaae-051f-41e9-9172-8ac4540fb161

### Утренний дневник
https://github.com/user-attachments/assets/ef986af6-37d0-41ec-9c5b-a930438d8b71

### Профиль
<img width="374" height="666" alt="Снимок экрана — 2026-06-24 в 20 56 47" src="https://github.com/user-attachments/assets/9a734433-8133-4a00-bf1a-fca2580389ad" />

### Основной интерфейс приложения
https://github.com/user-attachments/assets/285d7950-820a-422d-af46-9acaea74edcd

### Авторизация
<img width="374" height="667" alt="Снимок экрана — 2026-06-24 в 20 55 12" src="https://github.com/user-attachments/assets/c750dea5-45bf-49e1-aeea-e0b7911535d9" />

## Архитектура

### Декларативная конфигурация полей

Вопросы описываются как данные, а не прописываются вручную в HTML.

```js
const DAILY_DIARY_CONFIG = {
  containerId: 'page',

  fields: [
    {
      name: 'overall_state',
      type: 'range',
      required: true,
      mount: 'summarySlot',
      title: 'Как вы оцениваете прошедший день?',
      props: {
        min: 0,
        max: 5,
        step: 1,
        showTicks: true,
        showValue: false
      }
    },
    {
      name: 'activity_level',
      type: 'radio',
      required: true,
      mount: 'activitySlot',
      title: 'Каким был уровень активности?',
      options: [
        { label: 'Низкий', value: 'low' },
        { label: 'Средний', value: 'medium' },
        { label: 'Высокий', value: 'high' }
      ]
    }
  ]
};
```

Добавление нового вопроса в большинстве случаев требует только новой записи в конфиге.

### Переиспользуемая фабрика полей формы

```js
class FormModuleFactory {
  constructor(container) {
    this.container = container;
    this.modules = new Map();
    this.changeListeners = [];
  }

  createFromField(field, context) {
    const common = {
      name: field.name,
      title: resolveText(field.title, context),
      required: Boolean(field.required),
      mount: field.mount
    };

    switch (field.type) {
      case 'range':
        return this.createRange({ ...common, ...field.props });

      case 'radio':
        return this.createRadio({
          ...common,
          options: resolveOptions(field.options, context)
        });

      case 'textarea':
        return this.createTextarea({ ...common, ...field.props });

      default:
        throw new Error(`Неподдерживаемый тип поля: ${field.type}`);
    }
  }

  getAllValues({ onlyVisible = true } = {}) {
    const result = {};

    for (const [name, module] of this.modules) {
      if (onlyVisible && !module.visible) continue;
      result[name] = module.getValue();
    }

    return result;
  }

  validate({ onlyVisible = true } = {}) {
    return [...this.modules.values()].every(module => {
      if (onlyVisible && !module.visible) return true;
      return module.isValid();
    });
  }
}
```

Единый API используется для получения и установки значений, валидации, управления видимостью, сброса и восстановления черновиков.

### Видимость вопросов, управляемая backend-данными

```js
const mockSettings = {
  visibleQuestions: {
    nutrition: ['meal_count', 'breakfast'],
    wellbeing: ['headache', 'social_energy'],
    moduleA: ['module_a_question_1']
  }
};
```

```js
isFieldVisible(field) {
  if (!field.module) return true;

  const visibleNames = this.visibleQuestions[field.module];

  return Array.isArray(visibleNames)
    && visibleNames.includes(field.name);
}

renderFields() {
  for (const field of this.config.fields) {
    this.factory.createFromField(field, this.context);
    this.factory.setVisible(
      field.name,
      this.isFieldVisible(field)
    );
  }
}
```

За счёт этого одна frontend-сборка может отображать разные структуры дневника для разных пользователей.

### Подключаемые тематические модули

```js
window.DIARY_MODULES = window.DIARY_MODULES || {};

window.DIARY_MODULES.exampleModule = {
  title: 'Дополнительные вопросы',

  fields: [
    {
      name: 'example_module_intensity',
      module: 'exampleModule',
      type: 'range',
      required: true,
      title: 'Оцените интенсивность',
      props: {
        min: 0,
        max: 5,
        step: 1,
        showValue: false
      }
    }
  ]
};
```

```js
extraModules: [
  {
    id: 'exampleModule',
    mount: 'extraModulesSlot'
  }
]
```

```js
renderExtraModules() {
  for (const reference of this.config.extraModules ?? []) {
    const moduleConfig = window.DIARY_MODULES?.[reference.id];
    if (!moduleConfig) continue;

    const fields = moduleConfig.fields.map(field => ({
      ...field,
      module: field.module ?? reference.id,
      mount: field.mount ?? reference.mount
    }));

    const hasVisibleFields = fields.some(field =>
      this.isFieldVisible(field)
    );

    if (!hasVisibleFields) continue;

    for (const field of fields) {
      this.factory.createFromField(field, this.context);
      this.factory.setVisible(
        field.name,
        this.isFieldVisible(field)
      );
    }
  }
}
```

Один и тот же модуль может использоваться в основном, утреннем или отдельном тематическом дневнике.

### Адаптивные формулировки

```js
const sharedCopy = {
  addressPacks: {
    formal: {
      rate: 'Оцените',
      your: 'ваш'
    },
    informal: {
      rate: 'Оцени',
      your: 'твой'
    }
  }
};
```

```js
function applyTemplate(text, mode, config) {
  const dictionary = config.addressPacks?.[mode] ?? {};

  return String(text).replace(/\{(\w+)\}/g, (_, key) => {
    return dictionary[key] ?? `{${key}}`;
  });
}
```

```js
{
  name: 'energy',
  type: 'range',
  title: '{rate} {your} текущий уровень энергии',
  props: {
    min: 0,
    max: 5,
    step: 1
  }
}
```

Тот же механизм применяется к заголовкам полей, подписям вариантов ответа и статическим текстам страницы.

### Общая конфигурация

```js
window.AppConfig = window.AppConfig || {};

window.AppConfig.DIARY_SHARED = {
  moodGroups: [
    { id: 'emotional', title: 'Эмоциональное состояние' },
    { id: 'physical', title: 'Физическое состояние' }
  ],

  moods: [
    {
      key: 'calm',
      label: 'Спокойно',
      group: 'emotional',
      className: 'chip-calm'
    }
  ],

  addressPacks: {
    formal: {},
    informal: {}
  }
};
```

Это позволяет не дублировать словари состояний и наборы формулировок между разными страницами.

### Общий handler для коротких дневников

```js
class ShortDiaryHandler {
  async init() {
    await this.loadProfile();

    this.renderMoodSelector();
    this.renderFields();
    this.restoreDraft();

    this.factory.onChange(() => this.saveDraft());
  }

  buildPayload() {
    return {
      sessionData: getSessionData(),
      moods: this.getMoodValues(),
      ...this.factory.getAllValues({
        onlyVisible: true
      })
    };
  }
}
```

Разные короткие дневники создаются преимущественно за счёт изменения конфигурации.

### Локальное сохранение черновика

```js
saveDraft() {
  const draft = {
    savedAt: Date.now(),
    values: this.factory.getAllValues({
      onlyVisible: false
    }),
    selectedMoods: this.selectedMoods
  };

  localStorage.setItem(
    this.config.draftKey,
    JSON.stringify(draft)
  );
}
```

```js
restoreDraft() {
  const raw = localStorage.getItem(this.config.draftKey);
  if (!raw) return;

  const draft = JSON.parse(raw);
  const expired =
    Date.now() - draft.savedAt > DRAFT_TTL;

  if (expired) {
    localStorage.removeItem(this.config.draftKey);
    return;
  }

  for (const [name, value] of Object.entries(
    draft.values ?? {}
  )) {
    this.factory.setValue(name, value);
  }
}
```

### Страница настроек, собранная из тех же конфигов

Страница настроек вопросов не содержит отдельного вручную прописанного списка. Она получает доступные поля из базовой конфигурации и зарегистрированных тематических модулей.

```js
collectAvailableQuestions() {
  const baseQuestions = this.config.fields
    .filter(field => field.module)
    .map(field => ({
      source: 'base',
      module: field.module,
      name: field.name,
      title: field.title
    }));

  const extraQuestions = Object.entries(
    window.DIARY_MODULES ?? {}
  ).flatMap(([moduleId, moduleConfig]) =>
    moduleConfig.fields.map(field => ({
      source: 'extra',
      module: field.module ?? moduleId,
      name: field.name,
      title: field.title
    }))
  );

  return [
    ...baseQuestions,
    ...extraQuestions
  ];
}
```

Результат сохраняется в той же структуре, которую использует renderer дневника.

```js
{
  visibleQuestions: {
    nutrition: ['meal_count'],
    wellbeing: ['headache'],
    exampleModule: ['example_module_intensity']
  }
}
```

## Локальная имитация API

Во время frontend-разработки ответы API можно имитировать через обёртку над `window.fetch`.

```js
const originalFetch = window.fetch.bind(window);

window.fetch = async (request, options) => {
  const url = typeof request === 'string'
    ? request
    : request.url;

  if (url.includes('/api/example/profile')) {
    return createJsonResponse({
      addressMode: 'formal'
    });
  }

  if (url.includes('/api/example/question-settings')) {
    return createJsonResponse(mockSettings);
  }

  return originalFetch(request, options);
};
```

Так можно изолированно тестировать:

- разные режимы обращения к пользователю;
- разные комбинации видимых вопросов;
- пустые дополнительные модули;
- успешную и ошибочную отправку;
- восстановление состояния после перезагрузки;
- поведение интерфейса без готового backend.

## Пример структуры проекта

```text
src/
├─ pages/
│  ├─ diary.html
│  ├─ diary-morning.html
│  └─ diary-settings.html
├─ js/
│  ├─ handlers/
│  │  ├─ DiaryFormHandler.js
│  │  ├─ ShortDiaryHandler.js
│  │  └─ DiarySettingsHandler.js
│  ├─ factories/
│  │  └─ FormModuleFactory.js
│  ├─ config/
│  │  ├─ diary.shared.js
│  │  ├─ diary.config.js
│  │  └─ diary-morning.config.js
│  └─ modules/
│     ├─ example-a.module.js
│     └─ example-b.module.js
└─ css/
   ├─ theme.css
   └─ diary.css
```

## Продемонстрированные frontend-навыки

- архитектура на Vanilla JavaScript;
- конфигурационно-управляемый UI;
- фабрика переиспользуемых полей формы;
- рендеринг, управляемый backend-данными;
- подключаемые тематические модули;
- разделение конфигурации, рендеринга и бизнес-состояния;
- нормализация данных перед отправкой;
- mobile-first и адаптивная вёрстка;
- сохранение состояния в `localStorage`;
- клиентская валидация;
- доступные интерактивные элементы;
- переключение темы;
- mock API для изолированной frontend-разработки;
- отделение продуктовых текстов от логики интерфейса.

## Конфиденциальность и ограничения

Этот репозиторий предназначен только для демонстрации навыков.

Намеренно исключены или изменены:

- production API paths;
- детали авторизации;
- backend-код;
- схема базы данных;
- точные предметные формулировки;
- внутренние названия сущностей;
- приватные изображения и медиа;
- аналитика и мониторинг;
- коммерческие бизнес-правила.

Примеры кода сохраняют архитектурный подход, не раскрывая production-реализацию.

