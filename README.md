# Неделя 5: домашнее задание

## Перед тем как начать
- Как подготовить окружение [см. тут](./docs/01-prepare-environment.md)
- **САМОЕ ВАЖНОЕ** - полное описание базы данных, схему и описание поле можно найти [тут](./docs/02-db-description.md)
- Воркшоп и примеры запросов [см. тут](./docs/02-db-description.md)

## Основные требования
- решением каждого задания является ОДИН SQL-запрос
- не допускается менять схему или сами данные, если этого явно не указано в задании
- поля в выборках должны иметь псевдоним (alias) указанный в задании
- решение необходимо привести в блоке каждой задачи ВМЕСТО комментария "ЗДЕСЬ ДОЛЖНО БЫТЬ РЕШЕНИЕ" (прямо в текущем readme.md файле)
- метки времени должны быть приведены в формат _dd.MM.yyyy HH:mm:ss_ (время в БД и выборках в UTC)

## Прочие пожелания
- всем будет удобно, если вы будете придерживаться единого стиля форматирования SQL-команд, как в [этом примере](./docs/03-sql-guidelines.md)

## Задание 1: 100 заданий с самым долгим временем выполнения
Время, затраченное на выполнение задания - это период времени, прошедший с момента перехода задания в статус "В работе" и до перехода в статус "Выполнено".
Нужно вывести 100 заданий с самым долгим временем выполнения. 
Полученный список заданий должен быть отсортирован от заданий с наибольшим временем выполнения к заданиям с наименьшим временем выполнения.

Замечания:
- Невыполненные задания (не дошедшие до статуса "Выполнено") не учитываются.
- Когда исполнитель берет задание в работу, оно переходит в статус "В работе" (InProgress) и находится там до завершения работы. После чего переходит в статус "Выполнено" (Done).
  В любой момент времени задание может быть безвозвратно отменено - в этом случае оно перейдет в статус "Отменено" (Canceled).
- Нет разницы выполняется задание или подзадание.
- Выборка должна включать задания за все время.

Выборка должна содержать следующий набор полей:
- номер задания (task_number)
- заголовок задания (task_title)
- название статуса задания (status_name)
- email автора задания (author_email)
- email текущего исполнителя (assignee_email)
- дата и время создания задания (created_at)
- дата и время первого перехода в статус В работе (in_progress_at)
- дата и время выполнения задания (completed_at)
- количество дней, часов, минут и секнуд, которые задание находилось в работе - в формате "dd HH:mm:ss" (work_duration)

### Решение
```sql
  with in_progress_first_task as (select task_id
                                       , min(at) as min_at
                                    from task_logs
                                   where status = 3 /* InProgress */
                                   group by task_id)

select task.number                                                           as task_number
     , task.title                                                            as task_title
     , ts.name                                                               as status_name
     , author.email                                                          as author_email
     , assignee.email                                                        as assignee_email
     , to_char(task.created_at, 'dd.MM.yyyy HH24:mi:ss')                     as created_at
     , to_char(first_progress_task.min_at, 'dd.MM.yyyy HH24:mi:ss')          as in_progress_at
     , to_char(task.completed_at, 'dd.MM.yyyy HH24:mi:ss')                   as completed_at
     , to_char(task.completed_at - task.created_at, 'dd.MM.yyyy HH24:mi:ss') as at
  from tasks task
  join task_statuses ts on ts.id = task.status
  join users author on author.id = task.created_by_user_id
  join users assignee on assignee.id = task.assigned_to_user_id
  join in_progress_first_task first_progress_task on task.id = first_progress_task.task_id
 where task.status in (4) /* Done */
 order by at desc
 limit 100;
```

## Задание 2: Выборка для проверки вложенности
Задания могу быть простыми и составными. Составное задание содержит в себе дочерние - так получается иерархия заданий.
Глубина иерархии ограничено Н-уровнями, поэтому перед добавлением подзадачи к текущей задачи нужно понять, может ли пользователь добавить задачу уровнем ниже текущего или нет. Для этого нужно написать выборку для метода проверки перед добавлением подзадания, которая бы вернула уровень вложенности указанного задания и полный путь до него от родительского задания.

Замечания:
- ИД проверяемого задания передаем в sql как параметр _:parent_task_id_
- если задание _Е_ находится на 5м уровне, то путь должен быть "_//A/B/C/D/E_".

Выборка должна содержать:
- только 1 строку
- поле "Уровень задания" (level) - уровень указанного в параметре задания
- поле "Путь" (path)

### Решение
```sql
  with recursive tasks_tree
                     as (select t.id
                              , t.parent_task_id
                              , 1                 as level
                              , concat('/', t.id) as path
                           from tasks t
                          where t.id = :parent_task_id
                          union all
                         select t.id
                              , t.parent_task_id
                              , tt.level + 1               as level
                              , concat('/', t.id, tt.path) as path
                           from tasks t
                           join tasks_tree tt on t.parent_task_id = tt.id)

select level             as "Уровень задания"
     , concat('/', path) as "Путь"
  from tasks_tree
 where level = (select max(level)
                  from tasks_tree);
```

## Задание 3 (на 10ку): Денормализация
Наш трекер задач пользуется популярностью и количество только активных задач перевалило уже за несколько миллионов. Продакт пришел с очередной юзер-стори:
```
Я как Диспетчер в списке активных задач всегда должен видеть задачу самого высокого уровня из цепочки отдельным полем

Требования:
1. Список активных задач включает в себя задачи со статусом "В работе"
2. Список должен быть отсортирован от самой новой задачи к самой старой
3. В списке должны быть поля:
  - Номер задачи (task_number)
  - Заголовок (task_title)
  - Номер родительской задачи (parent_task_number)
  - Заголовок родительской задачи (parent_task_title)
  - Номер самой первой задачи из цепочки (root_task_number)
  - Заголовок самой первой задачи из цепочки (root_task_title)
  - Email, на кого назначена задача (assigned_to_email)
  - Когда задача была создана (created_at)
 4. Должна быть возможность получить данные с пагинацией по N-строк (@limit, @offset)
```

Обсудив требования с лидом тебе прилетели 2 задачи:
1. Данных очень много и нужно денормализовать таблицу tasks
   Добавить в таблицу tasks поле `root_task_id bigint not null`
   Написать скрипт заполнения нового поля root_task_id для всей таблицы (если задача является рутом, то ее id должен совпадать с root_task_id)
2. Написать запрос получения данных для отображения в списке активных задач
   (!) Выяснилось, что дополнительно еще нужно возвращать идентификаторы задач, чтобы фронтенд мог сделать ссылки на них (т.е. добавить поля task_id, parent_task_id, root_task_id)

<details>
  <summary>FAQ</summary>

**Q: Что такое root_task_id?**

A: Например, есть задача с id=10 и parent_task_id=9, задача с id=9 имеет parent_task_id=8 и т.д. до задача id=1 имеет parent_task_id=null. Для всех этих задач root_task_id=1.

**Q: Не понял в каком формате нужен результат?**

Ожидаемый результат выполнения SQL-запроса:

| task_id | task_number | task_title | parent_task_id | parent_task_number | parent_task_title | root_task_id | root_task_number | root_task_title | assigned_to_email | created_at          |
|---------|-------------|------------|----------------|--------------------|-------------------|--------------|------------------|-----------------|-------------------|---------------------|
| 1       | A123        | Тест 123   | null           | null               | null              | 1            | A123             | Тест 123        | test@test.tt      | 01.01.2023 08:00:00 |
| 2       | B123        | B-тест     | 1              | A123               | Тест 123          | 1            | A123             | Тест 123        | user@test.tt      | 01.01.2023 11:00:00 |
| 3       | C123        | 123-тест   | 2              | B123               | B-тест            | 1            | A123             | Тест 123        | dev@test.tt       | 01.01.2023 11:10:00 |
| 10      | 1-2345      | New task   | null           | null               | null              | 10           | 1-2345           | New task        | test@test.tt      | 12.02.2024 11:00:00 |

**Q: Все это можно делать в одной миграции?**

А: Нет, каждая DDL операция - отдельная миграция, DML-операция тоже долзна быть в отдельной миграции.

</details>

### Скрипты миграций
```sql
alter table tasks
 add column root_task_id bigint not null default 0;

with recursive tasks_tree
                   as (select t.id as root_task_id
                            , t.id as id
                         from tasks t
                        where t.parent_task_id is null
                        union all
                       select tt.root_task_id as root_task_id
                            , t.id
                         from tasks t
                         join tasks_tree tt on t.parent_task_id = tt.id)

update tasks
   set root_task_id = tt.root_task_id
  from tasks_tree tt
 where tasks.id = tt.id;
```

### Запрос выборки
```sql
   select task.id                                           as task_id
        , task.number                                       as task_number
        , task.title                                        as task_title
        , task.parent_task_id                               as parent_task_id
        , task_parent.number                                as parent_task_number
        , task_parent.title                                 as parent_task_title
        , task_root.id                                      as root_task_id
        , task_root.number                                  as root_task_number
        , task_root.title                                   as root_task_title
        , assigned_user.email                               as assigned_to_email
        , to_char(task.created_at, 'dd.MM.yyyy HH24:mi:ss') as created_at
     from tasks as task
left join tasks as task_parent on task.parent_task_id = task_parent.id
     join tasks as task_root on task.root_task_id = task_root.id
     join users as assigned_user on task.assigned_to_user_id = assigned_user.id
    where task.status in (3) /* InProgress */
 order by task.created_at desc
   offset @offset
    limit @limit;
```
