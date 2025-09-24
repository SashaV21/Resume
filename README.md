###  1. Блок: Пользователи и Доступ (IAM)

Это ядро системы, отвечающее за то, **кто** использует систему и **что** ему доступно.

#### **`users` (Пользователи)**
Хранит информацию о каждом зарегистрированном пользователе.
* `id` (Primary Key, PK): Уникальный идентификатор
* `email` (varchar, unique): Email, используется как логин
* `password_hash` (varchar): Хеш пароля (никогда не храните пароли в открытом виде!)
* `full_name` (varchar, nullable): ФИО пользователя
* `company_id` (Foreign Key, FK -> `companies.id`, nullable): Ссылка на компанию, если пользователь — часть организации. `NULL` для физ. лиц.
* `created_at` (timestamp): Дата регистрации

#### **`roles` (Роли)**
Справочник ролей для разграничения прав.
* `id` (PK)
* `name` (varchar, unique): Название роли (e.g., "Admin", "Business User", "Free User")

#### **`user_roles` (Связь Пользователи-Роли)**
Позволяет назначать пользователю несколько ролей (хотя в вашем случае, скорее всего, будет одна).
* `user_id` (FK -> `users.id`)
* `role_id` (FK -> `roles.id`)

#### **`companies` (Компании)**
Информация о клиентских компаниях.
* `id` (PK)
* `name` (varchar): Название компании
* `subscription_plan_id` (FK -> `subscription_plans.id`): Ссылка на их тарифный план
* `created_at` (timestamp)

#### **`subscription_plans` (Тарифные планы)**
Описание вариантов подписки.
* `id` (PK)
* `name` (varchar, unique): Название (e.g., "Пробный", "Малый бизнес", "Крупный бизнес")
* `description` (text): Описание
* `max_checks_per_month` (integer, nullable): Лимит проверок в месяц. `NULL` — безлимит.
* `api_access` (boolean): Доступно ли API на этом тарифе



---

### 2. Блок: Контент (БЗ и Блог)

Здесь хранится вся информация, которую пользователи читают и ищут.

#### **`articles` (Статьи и Законы)**
Единая таблица для всего контента.
* `id` (PK)
* `title` (varchar): Заголовок
* `content` (text): Содержимое
* `type` (enum: 'blog', 'kb_law', 'kb_practice'): Тип контента, чтобы их различать
* `author_id` (FK -> `users.id`): Автор (администратор/редактор)
* `published_at` (timestamp): Дата публикации
* `created_at` (timestamp)
* `updated_at` (timestamp)

#### **`tags` (Теги)**
Справочник тегов для удобной фильтрации.
* `id` (PK)
* `name` (varchar, unique): Название тега

#### **`article_tags` (Связь Статьи-Теги)**
Связывает статьи с тегами (связь "многие-ко-многим").
* `article_id` (FK -> `articles.id`)
* `tag_id` (FK -> `tags.id`)

---

### 3. Блок: ИИ-Помощник и Проверки

Самый важный блок, связанный с основной функцией сервиса.

#### **`ai_checks` (Запросы на проверку)**
Каждая проверка, инициированная пользователем или через API.
* `id` (PK)
* `user_id` (FK -> `users.id`, nullable): Кто инициировал проверку (если не через API)
* `company_id` (FK -> `companies.id`, nullable): К какой компании относится проверка (для статистики)
* `source_type` (enum: 'text', 'file', 'api'): Откуда пришел текст
* `input_text` (text, nullable): Сам текст, если `source_type = 'text'`
* `input_file_url` (varchar, nullable): Ссылка на загруженный файл
* `status` (enum: 'processing', 'completed', 'needs_review', 'failed'): Текущий статус
* `created_at` (timestamp): Дата проведения проверки

#### **`check_results` (Результаты проверки)**
Детальный отчет, связанный с каждой проверкой.
* `id` (PK)
* `check_id` (FK -> `ai_checks.id`, one-to-one): Жесткая связь с конкретной проверкой
* `result_class` (enum: 'violation', 'no_violation', 'undefined'): Класс фрагмента
* `risk_score` (integer): Уровень риска (0-100)
* `original_fragment` (text): Оригинальный текст
* `violation_fragment` (text, nullable): Фрагмент с нарушением
* `law_link` (varchar, nullable): Ссылка на норму закона
* `recommendation` (text, nullable): Переформулированный фрагмент

#### **`manual_reviews` (Ручные проверки)**
Очередь для запросов, где ИИ не уверен.
* `id` (PK)
* `check_id` (FK -> `ai_checks.id`): Проверка, требующая внимания
* `assignee_id` (FK -> `users.id`, nullable): Администратор, который взял в работу
* `status` (enum: 'queued', 'in_progress', 'resolved'): Статус ручной проверки
* `notes` (text): Комментарии администратора

---

### 4. Блок: API и Статистика

Все, что касается интеграции и сбора данных.

#### **`api_keys` (Ключи API)**
Ключи для доступа к API для крупных клиентов.
* `id` (PK)
* `company_id` (FK -> `companies.id`): Какой компании принадлежит ключ
* `key_hash` (varchar, unique): Хеш ключа доступа
* `is_active` (boolean): Активен ли ключ
* `created_at` (timestamp)

#### **`statistics_reports` (Отчеты по статистике)**
Сохраненные сгенерированные отчеты.
* `id` (PK)
* `company_id` (FK -> `companies.id`): Отчет для конкретной компании
* `generated_by_user_id` (FK -> `users.id`): Кто сгенерировал
* `report_data_json` (jsonb): Сами данные отчета в формате JSON (проценты, динамика и т.д.)
* `exported_file_url` (varchar, nullable): Ссылка на выгруженный файл (PDF, CSV)
* `start_date` (date): Начало периода отчета
* `end_date` (date): Конец периода отчета
* `created_at` (timestamp)

### Сводная схема связей:

* Пользователь (`user`) принадлежит Компании (`company`), у которой есть Тарифный план (`subscription_plan`).
* Проверка (`ai_check`) инициируется Пользователем (`user`) и относится к его Компании (`company`).
* У каждой Проверки (`ai_check`) есть один Результат (`check_result`).
* "Сложная" Проверка (`ai_check`) может попасть в очередь на Ручную проверку (`manual_review`).
* Администратор (пользователь с ролью `admin`) пишет Статьи (`articles`) и обрабатывает Ручные проверки (`manual_reviews`).


