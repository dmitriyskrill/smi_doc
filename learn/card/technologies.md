# Технологии

Статусы:

- **в конспекте** — выбор и механизм записаны, файла с этим поведением ещё нет;
- **на коде** — в заметке есть путь к файлу этого репозитория, и путь сверен с текущей реализацией;
- **могу вслух** — ставишь сам, когда объяснил механизм на коде без подсказки.

GraphQL и приёмы вокруг него разобраны как отличие от Express и Nest REST. TypeScript, Node и Git ты уже применял: здесь только то, чем этот проект отличается от привычного.

| Технология | Сейчас | Что уметь объяснить к концу | Где лежит разбор |
| --- | --- | --- | --- |
| GraphQL | на коде | Текст запроса выбирает поля; `@Query` nullable и `@ResolveField`; пустой список; день `YYYY-MM-DD` строкой | [bridge.md](bridge.md), решения 4, 5 и 9 |
| Apollo Sandbox | на коде | Страница текущего Apollo на `/graphql`: явный плагин, `graphiql` и `playground` выключены, интроспекция включена | `src/app.module.ts`, решение 4 |
| Batch loader | на коде | Свой `createBatchLoader`, не пакет DataLoader; один `findMany` на связь за микрозадачу; карта по id родителя; кэш умирает вместе с запросом | `src/profile/profile.batch-loader.ts`, `src/profile/profile.loader.ts`, решение 6 |
| NestJS | на коде | Модуль и DI как в REST; два резолвера вместо контроллера; три слоя; env проверяется при старте; shutdown hooks закрывают Prisma | `src/app.module.ts`, `src/main.ts`, `src/config/env.ts`, `src/profile/profile.module.ts`, решения 1–3 |
| Express (адаптер) | на коде | Процесс по-прежнему Express под Nest; GraphQL — маршрут `/graphql`; Fastify в визитке не выбран | `src/main.ts`, решение 1, [bridge.md](bridge.md), [Fastify_vs_Express_for_GraphQL.md](Fastify_vs_Express_for_GraphQL.md) |
| Prisma | на коде | Клиент 7.10.1 через `@prisma/adapter-pg`; один `PrismaService` на процесс Nest; репозиторий переводит строку в доменный тип; seed создаёт второй клиент в своём процессе | `src/prisma/prisma.service.ts`, `src/profile/profile.repository.ts`, `src/profile/profile.mapping.ts`, `prisma/seed.ts`, `prisma.config.ts`, решение 7 |
| PostgreSQL | на коде | Единственная база compose; чтение по `profileId` и `experienceId`; пустой volume наполняется до `listen` | `compose.yaml`, решения 7 и 10 |
| Docker | на коде | Postgres healthy, затем `migrate deploy`, seed, затем `exec node`; ненулевой шаг не открывает порт | `Dockerfile`, `compose.yaml`, `docker-entrypoint.sh`, решение 10 |
| TypeScript | на коде | Версия 6.0.3, сборка `nest build`. На границе слоя свои типы: домен, `@ObjectType`, строки `select` | `package.json`, `profile.types.ts`, `profile.objects.ts`, `profile.mapping.ts`, решения 3 и 7 |
| Node.js | на коде | `engines` — 26; образ `node:26-bookworm-slim`; процесс слушает порт после migrate и seed | `package.json`, `Dockerfile`, `docker-entrypoint.sh`, `src/main.ts` |
| Git | в этой сверке не проверялся | В истории ожидаются схема, миграции и конспект; секреты и том базы в коммиты не входят | git-команды не запускались, строку не переводить в «на коде» по памяти |

Номера решений — блоки [decisions.md](decisions.md).

## Что уже можно учить на коде

По строкам «на коде» путь в таблице существует в репозитории сервера. Фрагменты в [decisions.md](decisions.md) и [bridge.md](bridge.md) взяты из этих файлов.

Чем видно ленивое чтение без живого SQL:

- `ProfileRepository.findByOwnerKey` выбирает три скаляра и не передаёт `include`;
- `findExperienceByProfileIds` не выбирает достижения;
- методы связей вызываются только из loader, а loader — только из `@ResolveField`;
- `src/profile/profile.repository.spec.ts` закрепляет корень без `include` и пустой список id без `findMany`;
- `src/profile/profile.batch-loader.spec.ts` закрепляет одну выборку на микрозадачу, дедупликацию и отдельный кэш у каждого экземпляра.

Живой лог SQL с запущенного процесса в конспект не вставлялся. После прогона «только имя» и запроса из задания пункт можно учить вслух и отмечать «могу вслух». Чекбокс за себя конспект не ставит.

## Поле, которое попросят добавить вживую

Скаляр на уже существующей сущности проходит через те файлы, которые уже держат такое поле:

1. колонка в `prisma/schema.prisma` и новая миграция;
2. поле доменного типа в `src/profile/profile.types.ts`;
3. `@Field` в `src/profile/profile.objects.ts`;
4. `select` и функция маппинга в `src/profile/profile.repository.ts` и `src/profile/profile.mapping.ts`, если значение читается из строки;
5. значение в `prisma/seed.ts`.

Корневой `ProfileService.readOwner` при этом остаётся сценарием «найти владельца». Резолвер корня, Docker и entrypoint не меняются, если поле скалярное и не меняет правило «кто такой владелец».

Новая связь, которую клиент выбирает отдельно, добавляет ещё запись в seed, метод репозитория `find…By…Ids`, вызов `createBatchLoader` в `ProfileLoaderFactory.create` и метод `@ResolveField`. Второй уровень уже есть у опыта: `period` и `achievements`, родитель — опыт, класс — `ExperienceResolver`.

Мутации, подписки, авторизация, S3 и второй агрегат в этот список не входят. Их в сценарии визитки нет. Если их попросят на собеседовании, сначала появится решение в `decisions.md`, потом код.

## Что в этой сверке не открывалось

Чтобы заметка не подтверждала чужой файл по памяти:

- `prisma/schema.prisma`, текст миграций, `tsconfig.json` и `.nvmrc` не перечитывались;
- история git не смотрелась;
- сервер в Docker не поднимался, лог SQL не снимался.

Каскад удаления точек периода и достижений в блок 7 попал из комментария `prisma/seed.ts`. Сам seed всё равно удаляет их раньше опыта.
