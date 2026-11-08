# От Express и Nest REST к этой визитке

Статус на 2026-11-07: путь запроса идёт по файлам `it_smi-server`. Ниже имена функций, которые реально вызываются.

## Что остаётся прежним

Nest поднимает один процесс Node и сам слушает порт. Штатный HTTP-адаптер — Express: `NestFactory.create` в `src/main.ts` адаптер не подменяет. GraphQL не заменяет Express и не переводит процесс на Fastify. Он занимает маршрут `/graphql` этого HTTP-сервера. Регистрация маршрута, схемы и Sandbox — `src/app.module.ts`.

Знакомое из Nest REST остаётся в силе:

- модуль собирает провайдеры (`AppModule`, `ProfileModule`, `PrismaModule`);
- сервис — место сценария (`ProfileService.readOwner`);
- зависимости приходят в конструктор;
- к базе ходят асинхронно через один `PrismaService`;
- окружение читается при старте, кривой env роняет процесс в `validateEnv`;
- `OnModuleInit` / `OnModuleDestroy` открывают и закрывают клиент базы, shutdown hooks включены в `main.ts`.

Знакомое из Express остаётся под этим слоем: входящий HTTP, разбор тела POST, ответ JSON, `listen`.

Пайпы, гарды и `ValidationPipe`, которыми в REST проверяют body, на пути чтения визитки не стоят. У запроса `profile` нет аргументов и нет тела с полями пользователя. Эти инструменты появятся только вместе с входными данными.

## Кто собирает форму ответа

В Express ты объявляешь path и сам решаешь, какой JSON уйдёт в `res`. В Nest REST то же делает метод контроллера: один `@Get` — одна форма ответа.

В GraphQL сервер публикует граф типов, а форму конкретного ответа выбирает клиент текстом запроса. Один маршрут `/graphql` отдаёт и имя, и имя вместе с навыками. Сервер исполняет только выбранные поля.

Поэтому вместо набора контроллеров здесь два класса резолвера в `src/profile/profile.resolver.ts`. `ProfileResolver` принимает корень и связи профиля. `ExperienceResolver` принимает период и достижения опыта. В REST нельзя «не прислать кусок DTO», не написав второй endpoint или флаг `?include=`. Здесь невыбранное поле просто не вызывает свой метод.

## Словарь

| Слово | Продолжение того, что уже знаешь | Где в визитке |
| --- | --- | --- |
| Схема | Полный перечень типов и полей. По роли близко к собранному OpenAPI. | Собирается в памяти из `profile.objects.ts` и `@ResolveField`, `autoSchemaFile: true` |
| `query { ... }` | Чтение. По роли близко к GET, состав JSON задан текстом. | POST `/graphql` |
| Тип | Форма объекта. По роли близко к response DTO. | Класс `@ObjectType` |
| Поле | Свойство этой формы. | `@Field` или метод `@ResolveField` |
| `@Query` | Точка входа. По роли близко к `@Get`. | `ProfileResolver.profile` |
| `@ResolveField` | Функция одного вложенного поля. Вызывается, только если клиент это поле выбрал. | `links`, `skills`, `experience`, `projects`, `period`, `achievements` |
| Resolver | Класс этих функций. Занимает место контроллера на чтении визитки. | `profile.resolver.ts` |
| Apollo Sandbox | Страница, с которой отправляют запрос и смотрят схему. По роли близко к Swagger UI. | Плагин в `src/app.module.ts`, GET `/graphql` |
| Интроспекция | Схема, отданная самой себе. | `introspection: true` |
| `String!` | Поле обязательно и не будет `null`. | Скаляры вроде `name`, `on` |
| Nullable `profile` | Корня может не быть. В REST это чаще 404. | `@Query(..., { nullable: true })`, ответ `null` |
| `[Skill!]!` | Список приходит массивом, элементы заполнены. Пустая связь — `[]`. | Коллекции без `nullable: true`; loader отдаёт массив |
| Mutation | Запись. В REST это POST/PATCH. | В резолвере мутаций нет, пишет `prisma/seed.ts` |
| Batch loader | Сбор одинаковых чтений связи в один запрос к базе на текущий HTTP-запрос. Пакет DataLoader не подключён: ту же роль выполняет `createBatchLoader`. | `profile.batch-loader.ts`, сборка в `profile.loader.ts` |

## Путь запроса из задания

Клиент отправляет:

```graphql
query {
  profile {
    name
    description
    skills { name }
    experience { company position }
    projects { name }
  }
}
```

Дальше по файлам.

1. Браузер открывает GET `/graphql`. HTML рисует `ApolloServerPluginLandingPageLocalDefault` из `src/app.module.ts`. Отдельного сервиса страницы нет.
2. Запрос из страницы уходит HTTP POST на `/graphql`. Тело — JSON с текстом запроса. Express принимает соединение, Apollo исполняет текст. Контроллера `@Get('profile')` на этом пути нет.
3. Apollo сверяет текст со схемой. Её Nest собрал из `profile.objects.ts` и методов `@ResolveField`.
4. Context-функция в `src/app.module.ts` вызывает `ProfileLoaderFactory.create()`. На этот запрос появляются свои loader-ы. Репозиторий тот же, что у остальных запросов процесса.
5. Совпадает корень `profile`. `ProfileResolver.profile` вызывает `ProfileService.readOwner`. Сервис вызывает `ProfileRepository.findByOwnerKey('owner')`. В базу уходит `findUnique` с `select` из `id`, `name`, `description`, без `include`.
6. Строки владельца нет — наружу уходит `null`, шаги коллекций не начинаются.
7. Строка есть. В тексте ещё есть `skills`, `experience`, `projects`. На каждое поле Apollo вызывает свой `@ResolveField` и передаёт доменный `ProfileCard` как `@Parent()`. Полей `links`, `period` и `achievements` в этом тексте нет, их методы не вызываются.
8. Field resolver вызывает `context.loaders.skills.load(profile.id)` (и так же experience и projects). `createBatchLoader` копит ключи до микрозадачи, склеивает одинаковые и один раз зовёт `findSkillsByProfileIds`, `findExperienceByProfileIds`, `findProjectsByProfileIds`.
9. Чтение опыта не вкладывает период и достижения. `ExperienceResolver.period` и `ExperienceResolver.achievements` стартуют только если поле есть в тексте. Тогда loader зовёт `findPointsByExperienceIds(experience.id)` или `findAchievementsByExperienceIds(experience.id)`.
10. Репозиторий группирует строки по id родителя и переводит каждую в доменный тип. День точки периода (`on`) становится строкой `YYYY-MM-DD` в `mapExperiencePoint`, когда клиент запросил `period`. Ключ без строк даёт пустой массив.
11. Apollo складывает JSON только из спрошенных полей и отдаёт его телом HTTP-ответа.

Если из текста убрать `skills`, `experience` и `projects`, шаг 8 для этих связей не выполняется. Остаётся чтение строки профиля. В REST для такого поведения ты писал второй маршрут или флаг `?include=`. Здесь это свойство текста запроса.

Один профиль делает второе чтение на связь почти незаметным. Приём оставлен сразу: несколько родителей на одну связь по-прежнему сходятся в один вызов `find…ByProfileIds` за микрозадачу.

## Порядок, который легко перепутать со REST

В REST сервис часто возвращает уже собранный объект: профиль и внутри массив навыков. Если сделать так здесь, field resolver станет пустой обёрткой, а навыки прочитаются даже для запроса, где нужно только имя.

Договор в коде такой. `ProfileService.readOwner` возвращает скаляры профиля или `null`. Коллекции собирают только field resolver и loader текущего контекста. `id` профиля и `id` опыта доезжают до `@Parent()` и в схему не публикуются.

## Где смотреть пальцем

- Express-приложение Nest: `src/main.ts`, `NestFactory.create` без смены адаптера, `enableShutdownHooks`, `listen`.
- `/graphql` и Sandbox: `src/app.module.ts`, `GraphQLModule.forRootAsync`.
- Класс корня: `ProfileResolver.profile`.
- Метод навыков: `ProfileResolver.skills`. Период и достижения: `ExperienceResolver.period`, `ExperienceResolver.achievements`.
- Loader на запрос: context-функция и `ProfileLoaderFactory.create`. Кэш внутри `createBatchLoader`, не в singleton репозитория.
- Корень без таблиц связей: `ProfileRepository.findByOwnerKey`. Тест, который держит `select` и отсутствие `include`: `src/profile/profile.repository.spec.ts`.
- Живой SQL в логе сервера этой заметкой не снимался. По коду невыбранная связь не доходит до своего `findMany`.
