git 61de1342a35dd91c0189c67b979afa243edb61db

---

# Инструкции по обновлению

- [Обновление до 5.1.11](#upgrade-5.1.11)
- [Обновление до 5.1.0](#upgrade-5.1.0)
- [Обновление до 5.0.16](#upgrade-5.0.16)
- [Обновление до 5.0 с 4.2](#upgrade-5.0)
- [Обновление до 4.2 с 4.1](#upgrade-4.2)
- [Обновление до 4.1.29 с <= 4.1.x](#upgrade-4.1.29)
- [Обновление до 4.1.26 с <= 4.1.25](#upgrade-4.1.26)
- [Обновление до 4.1 с 4.0](#upgrade-4.1)

<a name="upgrade-5.1.11"></a>
## Обновление до 5.1.11

В Laravel 5.1.11 появилась поддержка [авторизации](/docs/{{version}}/authorization) и [классов политик авторизации](/docs/{{version}}/authorization#policies).

> **Примечание:** Этот апгрейд **опционален**, если вы не используете авторизацию Laravel в своем приложении, то можете его не делать.

#### Создание папки для классов политик

Создайте папку `app/Policies`.

#### AuthServiceProvider и фасад Gate

Создайте сервис-провайдер `AuthServiceProvider` (контент можно взять [с гитхаба](https://raw.githubusercontent.com/laravel/laravel/master/app/Providers/AuthServiceProvider.php)) в папке `app/Providers`. 

Добавьте `AuthServiceProvider` в массив `providers` в `config/app.php`.
Там же зарегистрируйте фасад `Gate` в массиве `aliases`:

    'Gate' => Illuminate\Support\Facades\Gate::class,

#### Обновите модель User

Добавьте в модель `User` трейт `Illuminate\Foundation\Auth\Access\Authorizable` и контракт `Illuminate\Contracts\Auth\Access\Authorizable`:

    <?php

    namespace App;

    use Illuminate\Auth\Authenticatable;
    use Illuminate\Database\Eloquent\Model;
    use Illuminate\Auth\Passwords\CanResetPassword;
    use Illuminate\Foundation\Auth\Access\Authorizable;
    use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableContract;
    use Illuminate\Contracts\Auth\Access\Authorizable as AuthorizableContract;
    use Illuminate\Contracts\Auth\CanResetPassword as CanResetPasswordContract;

    class User extends Model implements AuthenticatableContract,
                                        AuthorizableContract,
                                        CanResetPasswordContract
    {
        use Authenticatable, Authorizable, CanResetPassword;
    }

#### Обновите базовый контроллер

Добавьте в `App\Http\Controllers\Controller` трейт `Illuminate\Foundation\Auth\Access\AuthorizesRequests`:

    <?php

    namespace App\Http\Controllers;

    use Illuminate\Foundation\Bus\DispatchesJobs;
    use Illuminate\Routing\Controller as BaseController;
    use Illuminate\Foundation\Validation\ValidatesRequests;
    use Illuminate\Foundation\Auth\Access\AuthorizesRequests;

    abstract class Controller extends BaseController
    {
        use AuthorizesRequests, DispatchesJobs, ValidatesRequests;
    }


<a name="upgrade-5.1.0"></a>
## Обновление до 5.1.0

#### Ориентировочное время обновления: меньше часа.

### Обновите `bootstrap/autoload.php`

Обновите переменную `$compiledPath` в файле `bootstrap/autoload.php`:

	$compiledPath = __DIR__.'/cache/compiled.php';

### Создайте папку `bootstrap/cache`

Внутри папки `bootstrap` создайте папку `cache` directory (`bootstrap/cache`). Положите туда файл `.gitignore` со следующим содержимым:

	*
	!.gitignore

У этой папки должны быть установлены права на запись, она будет использоваться для хранения различных оптимизированных файлов, таких как `compiled.php`, `routes.php`, `config.php` и `services.json`.

### Сервис-провайдер BroadcastServiceProvider

В `config/app.php` добавьте `Illuminate\Broadcasting\BroadcastServiceProvider` в массив `providers`.

### Аутентификация

Если вы используете `AuthController` с трейтом `AuthenticatesAndRegistersUsers`, то вам нужно сделать несколько изменений. Изменения коснулись валидации и создания новых пользователей.

Во-первых, больше не нужно передавать в контроллер `Guard` и `Registrar`. Вы можете удалить эти зависимости из конструктора контроллера.

Во-вторых, класс `App\Services\Registrar` в 5.1 больше не используется. Вам нужно скопировать у него методы `validator` и `create` и вставить их в `AuthController`. И, разумеется, проследить, чтобы необходимые классы (`Validator` и `User`) были импортированы при помощи конструкции `use`.

#### Password Controller

`PasswordController` больше не используется. Вы можете его удалить, если хотите.

### Валидация

Если вы изменили метод `formatValidationErrors` в базовом контроллере, поставьте в аргументах `Illuminate\Contracts\Validation\Validator` вместо `Illuminate\Validation\Validator`.

Если вы изменили метод `formatErrors` в базовом классе запросов, поставьте в аргументах `Illuminate\Contracts\Validation\Validator` вместо `Illuminate\Validation\Validator`.

### Eloquent

#### Метод `create`

Метод `create` теперь может вызываться без параметров. Если вы переопределили его в своих моделях, то добавьте в параметры дефолтное значение:

	public static function create(array $attributes = [])
	{
		// 
	}

Если вы не переопределяли этот метод, ничего делать не нужно.

#### Метод `find`

Метод `find` теперь принадлежит к Query Builder, а не к Eloquent.

Если вы переопределили метод `find` в своих моделях и делаете `parent::find()` внутри него, вы должны изменить этот вызов следующим образом:

	public static function find($id, $columns = ['*'])
	{
		$model = static::query()->find($id, $columns);

		// ...

		return $model;
	}

Если вы не переопределяли этот метод, ничего делать не нужно.	

#### Метод `lists` 

The `lists` method now returns a `Collection` instance instead of a plain array for Eloquent queries. If you would like to convert the `Collection` into a plain array, use the `all` method:

`lists` теперь возвращает коллекцию, а не массив. Если вы хотите получать именно массив, воспользуйтесь методом `all()` после него:

    User::lists('id')->all();

Но имейте в виду, что метод `lists` конструктора запросов по прежнему возвращает массив.

#### Форматирование даты

Раньше вы могли задать свой формат полей с датой путём переопределения метода `getDateFormat` в вашей модели. Это по-прежнему возможно. Но теперь это можно сделать более удобнее - просто определите свойство `$dateFormat` в вашей модели.

Обратите внимание, что форматирование дат стало применяться во время сериализации модели в массив или JSON. Это может изменить формат некоторых данных в JSON-файлах в вашем приложении. Чтобы установить нужный формат данных для сериализованных моделей, переопределите метод `serializeDate(DateTime $date)` в вашей модели.

### Коллекции

#### Метод `sortBy`

Метод `sortBy` возвращает не модифицированную прежнюю коллекцию, а созданную с нуля:

    $collection = $collection->sortBy('name');

#### Метод `groupBy`

Метод `groupBy` теперь возвращает объект `Collection`, а не массив, для каждого элемента в родительском `Collection`. Если вы хотите преобразовать их в массивы, воспользуйтесь методом 'map':

	$collection->groupBy('type')->map(function($item)
	{
		return $item->all();
	});

#### Метод `lists`

Метод `lists` тоже возвращает объект `Collection`, а не массив. Если вы хотите преобразовать его в массив, воспользуйтесь методом `all()`:

	$collection->lists('id')->all();

### Commands & Handlers

Переименуйте папку `app/Commands` в `app/Jobs`. However, you are not required to move all of your commands to the new location, and you may continue using the `make:command` and `handler:command` Artisan commands to generate your classes.

Переименуйте `app/Handlers` в `app/Listeners` and now only contains event listeners. However, you are not required to move or rename your existing command and event handlers, and you may continue to use the `handler:event` command to generate event handlers.

By providing backwards compatibility for the Laravel 5.0 folder structure, you may upgrade your applications to Laravel 5.1 and slowly upgrade your events and commands to their new locations when it is convenient for you or your team.

### Blade

The `createMatcher`, `createOpenMatcher`, and `createPlainMatcher` methods have been removed from the Blade compiler. Use the new `directive` method to create custom directives for Blade in Laravel 5.1. Consult the [extending blade](/docs/{{version}}/blade#extending-blade) documentation for more information.

### Tests

Add the protected `$baseUrl` property to the `tests/TestCase.php` file:

    protected $baseUrl = 'http://localhost';

### Translation Files

The default directory for published language files for vendor packages has been moved. Move any vendor package language files from `resources/lang/packages/{locale}/{namespace}` to `resources/lang/vendor/{namespace}/{locale}` directory. For example, `Acme/Anvil` package's `acme/anvil::foo` namespaced English language file would be moved from `resources/lang/packages/en/acme/anvil/foo.php` to `resources/lang/vendor/acme/anvil/en/foo.php`.

### Amazon Web Services SDK

If you are using the AWS SQS queue driver or the AWS SES e-mail driver, you should update your installed AWS PHP SDK to version 3.0.

If you are using the Amazon S3 filesystem driver, you will need to update the corresponding Flysystem package via Composer:

- Amazon S3: `league/flysystem-aws-s3-v3 ~1.0`

### Deprecations

The following Laravel features have been deprecated and will be removed entirely with the release of Laravel 5.2 in December 2015:

<div class="content-list" markdown="1">
- Route filters have been deprecated in preference of [middleware](/docs/{{version}}/middleware).
- The `Illuminate\Contracts\Routing\Middleware` contract has been deprecated. No contract is required on your middleware. In addition, the `TerminableMiddleware` contract has also been deprecated. Instead of implementing the interface, simply define a `terminate` method on your middleware.
- The `Illuminate\Contracts\Queue\ShouldBeQueued` contract has been deprecated in favor of `Illuminate\Contracts\Queue\ShouldQueue`.
- Iron.io "push queues" have been deprecated in favor of typical Iron.io queues and [queue listeners](/docs/{{version}}/queues#running-the-queue-listener).
- The `Illuminate\Foundation\Bus\DispatchesCommands` trait has been deprecated and renamed to `Illuminate\Foundation\Bus\DispatchesJobs`.
- `Illuminate\Container\BindingResolutionException` has been moved to `Illuminate\Contracts\Container\BindingResolutionException`.
- The service container's `bindShared` method has been deprecated in favor of the `singleton` method.
- The Eloquent and query builder `pluck` method has been deprecated and renamed to `value`.
- The collection `fetch` method has been deprecated in favor of the `pluck` method.
- The `array_fetch` helper has been deprecated in favor of the `array_pluck` method.
</div>

<a name="upgrade-5.0.16"></a>
## Обновление до 5.0.16

Обновите файл `bootstrap/autoload.php`:

	$compiledPath = __DIR__.'/../vendor/compiled.php';

<a name="upgrade-5.0"></a>
## Обновление до 5.0 с 4.2

### Общие положения

Общие рекомендации к переходу на Laravel 5.0 таковы - нужно создать новое приложение Laravel и скопировать в него классы и файлы из старого приложения Laravel 4.2, а именно контроллеры, роуты, Eloquent-модели, команды Artisan, статические файлы (css/js), отображения (views) и т.п. в новые места, по пути соответственно их изменив.

Итак, [установите приложение Laravel 5.0](/docs/{{version}}/installation) в новую папку на своей локальной машине. Дальше мы рассмотрим процесс миграции поподробнее.

### Зависимости Composer и пакеты

Отредактируйте composer.json - добавьте туда зависимости и пакеты, которые использует ваше приложение. Убедитесь, что версии laravel-пакетов, которые вы используете, совместимы с Laravel 5.0.

После редактирования запустите `composer update`.

### Неймспейсы

В отличие от Laravel 5.0, Laravel 4.x по умолчанию не использовал неймспейсы в контроллерах и моделях. Для упрощения перехода вы можете оставить эти классы в глобальном неймспейсе Laravel? отредактировав секцию `classmap` в `composer.json`.

### Настройка 

#### Переменные среды выполнения

Скопируйте `.env.example` в файл `.env`. Этот файл - эквивалент старого `.env.php`. Установите в нём ваши переменные среды выполнения из старого файла, и отредактируйте новые, как то APP_KEY (ключ шифрования приложения), APP_ENV (название среды выполнения - теперь она задается в `.env` файле), тип драйвера кэша и сессий. 

> **Примечание:** Название среды выполнения теперь задается не в привязке к имени машины как в Laravel 4.х, а в файле `.env` в параметре `APP_ENV`.

[Документация по настройке среды выполнения](/docs/{{version}}/configuration#environment-configuration).

> **Примечание:** Вы должны создать корректный `.env` перед тем как разворачивать (deploy) ваше Laravel 5 приложение на продакшн (основном) сервере.

#### Конфиги

Laravel 5.0 больше не использует систему разграничения конфигов для разных сред выполнения в различных папках (`app/config/{название_среды_выполнения}/`). Теперь все конфигурационные файлы находятся в папке `config`. Значения для конфигов вносятся в `.env` и читаются в конфигах конструкцией `env('key', 'default value')`. Для примера, посмотрите файл `config/database.php`.

Запомните, если вы добавляете ключ в `.env` файл, добавляйте его и в `.env.example`, чтобы при развертывании приложения в новой среде выполнения в конфигах оказались все нужные значения.

### Роуты

Скопируйте `routes.php` в `app/Http/routes.php`.

### Контроллеры

Скопируйте контроллеры в папку `app/Http/Controllers`. Далее вы можете 
1. Поместить их в неймспейс контроллеров, добавив каждому в начало `namespace App\Http\Controllers;` 
или 
2. Добавить папку в `app/Http/Controllers` в директиву `classmap` файла `composer.json` для того, чтобы назначить её папкой глобального неймспейса. Убрать неймспейс у базового контроллера `app/Http/Controllers/Controller.php`. В файле `app/Providers/RouteServiceProvider.php` установите свойство `namespace` в `null`.

Проследите, чтобы все контроллеры наследовались от базового контроллера `Controller`.

### Фильтры роутов

Возьмите ваши фильтры роутов из `app/filters.php` и разместите их в методе `boot()` файла `app/Providers/RouteServiceProvider.php`. Добавьте `use Illuminate\Support\Facades\Route;` в начало этого файла, чтобы корректно работал фасад `Route`.

Если вы не создавали своих фильтров, а использовали только встроенные (`auth`, `csrf` и т.д.), то переносить их не надо. Они остались, только теперь называются middleware (посредники). Чтобы задействовать их в своих роутах, замените `'before'` на `'middleware'` (например, `['before' => 'auth']` меняется на `['middleware' => 'auth'].`).

Фильтры в целом не пропали из Laravel, вы можете использовать их в `before` и `after` как раньше. Просто дефолтные фильтры роутов переместились в middleware.

### Глобальная фильтрация CSRF

Теперь [CSRF защита](/docs/{{version}}/routing#csrf-protection) включена по дефолту для всех роутов. Чтобы вернуть старое поведение и не проверять CSRF, удалите следующую строку из массива `middleware` класса `App\Http\Kernel`:

	'App\Http\Middleware\VerifyCsrfToken',

Если вы хотите делать выборочную проверку на CSRF, добавьте в `$routeMiddleware` класса `App\Http\Kernel` следующее:

	'csrf' => 'App\Http\Middleware\VerifyCsrfToken',

После этого вы можете использовать в роутах конструкцию `['middleware' => 'csrf']` для проверки CSRF.

[Документация по middleware](/docs/{{version}}/middleware).

### Модели Eloquent

Чтобы максимально просто перенести модели, создайте папку `app/Models`, скопируйте их туда и добавьте эту папку в директиву `classmap` файла `composer.json`.

Замените во всех моделях, которые используют псевдоудаление, `SoftDeletingTrait` на `Illuminate\Database\Eloquent\SoftDeletes`.

#### Кэширование в Eloquent

Eloquent больше не использует метод `remember()` для кэширования запросов. Вы должны явно кэшировать результаты запросов при помощи `Cache::remember`.

[Документация по кэшированию](/docs/{{version}}/cache).

### Модель User и аутентификация

Модель User нужно обновить, так как в Latavel 5.0 изменилась система аутентификации.

**Удалите это из блока `use` модели**

```php
use Illuminate\Auth\UserInterface;
use Illuminate\Auth\Reminders\RemindableInterface;
```

**Добавьте это в блок `use` модели:**

```php
use Illuminate\Auth\Authenticatable;
use Illuminate\Auth\Passwords\CanResetPassword;
use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableContract;
use Illuminate\Contracts\Auth\CanResetPassword as CanResetPasswordContract;
```

**Удалите UserInterface и RemindableInterface**

**Добавьте имплементацию:**

```php
implements AuthenticatableContract, CanResetPasswordContract
```

**Добавьте следующие трейты внутрь класса:**

```php
use Authenticatable, CanResetPassword;
```

**Если вы использовали их, удалите `Illuminate\Auth\Reminders\RemindableTrait` и `Illuminate\Auth\UserTrait` из вашего блока use и объявления класса.**

### Laravel Cashier

Имя трейта и интерфейса, которые использует [Laravel Cashier](/docs/{{version}}/billing) теперь изменены. Вместо трейта `BillableTrait` используйте `Laravel\Cashier\Billable`. Вместо имплементации интерфейса `Laravel\Cashier\BillableInterface` используйте `Laravel\Cashier\Contracts\Billable`.

### Artisan-команды

Скопируйте ваши артизан-команды в папку `app/Console/Commands` и добавьте эту папку в `classmap` вашего `composer.json`.

Затем скопируйте список команд из `start/artisan.php` в массив `command` файла `app/Console/Kernel.php`.

### Миграции и seeds

Удалите имеющиеся миграции (2 штуки) и скопируйте ваши миграции в `database/migrations`, а seeds в `database/seeds`.

### Глобальные IoC-биндинги

Если вы что-то добавляли в [IoC](/docs/{{version}}/container) в файле `start/global.php`, переместите этот код в метод `register` файла `app/Providers/AppServiceProvider.php`. Вам, возможно, потребуется импортировать фасад `App`.

Если хотите, можете разложить эти биндинги по разным сервис-провайдерам, исходя из их логической принадлежности.

### Шаблоны (views)

Скопируйте ваши шаблоны в папку `resources/views`.

### Тэги Blade

В Laravel 5.0 изменены функции тэгов Blade. Теперь `{{ }}` экранируют вывод, как и `{{{ }}}`. Чтобы вывести неэкранированные данные используйте `{!! !!}`, но имейте в виду, что вывод неэкранированных данных может быть серьёзной дырой в безопасности.

Если же вы хотите продолжить использовать старый синтаксис тэгов Blade, добавьте в конец `AppServiceProvider@register`:

```php
\Blade::setRawTags('{{', '}}');
\Blade::setContentTags('{{{', '}}}');
\Blade::setEscapedContentTags('{{{', '}}}');
```

Тэг коммента `{{--` больше не используется.

### Файлы локализации

Скопируйте ваши файлы локализации из `app/lang` в `resources/lang`.

### Папка Public 

Скопируйте содержимое вашей папки `public` **кроме файла index.php** в папку `public` Laravel 5.0.

### Тесты 

Скопируйте тесты из `app/tests` в папку `tests`.

### Остальные файлы

Скопируйте остальные файлы проекта (`.scrutinizer.yml`, `bower.json` и т.п.) на те же места.

Вы можете располагать Sass, Less или CoffeeScript файлы в любом месте, но если не можете выбрать - `resources/assets` будет хорошим выбором.

### Формы и HTML-хелперы

Для работы с фасадами `Form::` и `HTML::` вам нужно установить дополнительный пакет [Laravel Collective](http://laravelcollective.com/docs/5.0/html), они теперь не входят во фреймворк. 

Для этого добавьте `"laravelcollective/html": "~5.0"` в секцию `require` файла `composer.json`. Вам также нужно добавить сервис-провайдер в массив 'providers' конфига `config/app.php`:

    'Collective\Html\HtmlServiceProvider',

а также добавить два алиаса туда же:

    'Form'      => 'Collective\Html\FormFacade',
    'Html'      => 'Collective\Html\HtmlFacade',

### CacheManager

Если ваше приложение использует `Illuminate\Cache\CacheManager` в DI, используйте `Illuminate\Contracts\Cache\Repository` вместо него.

### Пагинация

Замените вызовы `$paginator->links()` на `$paginator->render()`.

Замените `$paginator->getFrom()` и `$paginator->getTo()` на `$paginator->firstItem()` и `$paginator->lastItem()` соответственно.

Уберите префикс "get" у `$paginator->getPerPage()`, `$paginator->getCurrentPage()`, `$paginator->getLastPage()` и `$paginator->getTotal()` (т.е. `$paginator->perPage()` и т.д.).

### Очередь Beanstalk

Laravel 5.0 теперь требует `"pda/pheanstalk": "~3.0"` вместо `"pda/pheanstalk": "~2.1"`

### Remote

Компонент Remote больше не используется (deprecated).

### Workbench

Компонент Workbench больше не используется (deprecated).

<a name="upgrade-4.2"></a>
## Обновление до 4.2 с 4.1

### PHP 5.4+

Для Laravel 4.2 необходима версия PHP 5.4.0 или более новая.

### Encryption Defaults

Добавьте новую опцию `cipher` в файле конфигурации `app/config/app.php`. Значение этой опции должно быть `MCRYPT_RIJNDAEL_256`.

	'cipher' => MCRYPT_RIJNDAEL_256

Этот параметр может быть использован для управления шифрованием, используемый по умолчанию в Laravel.

> **Примечание:** в Laravel 4.2 шифрованием по умолчанию является `MCRYPT_RIJNDAEL_128` (AES), Который считается самым безопасным. Изменение шифрования обратно на `MCRYPT_RIJNDAEL_256` является необходимым для расшифровки cookies/values, которые были использованы в Laravel <= 4.1

### Soft Deleting Models Now Use Traits

If you are using soft deleting models, the `softDeletes` property has been removed. Теперь вы должны использовать `SoftDeletingTrait`, например:

	use Illuminate\Database\Eloquent\SoftDeletingTrait;

	class User extends Eloquent {
		use SoftDeletingTrait;
	}

Вы также должны вручную добавить `deleted_at` в `dates`:

	class User extends Eloquent {
		use SoftDeletingTrait;

		protected $dates = ['deleted_at'];
	}

API для всех мягких операций удаления остается таким же.

> **Примечание:**  `SoftDeletingTrait` не могут быть применены на базовой модели. Он должен быть использован на модели класса.

### View / Pagination Environment Renamed

If you are directly referencing the `Illuminate\View\Environment` class or `Illuminate\Pagination\Environment` class, update your code to reference `Illuminate\View\Factory` and `Illuminate\Pagination\Factory` instead. These two classes have been renamed to better reflect their function.

### Additional Parameter On Pagination Presenter

If you are extending the `Illuminate\Pagination\Presenter` class, the abstract method `getPageLinkWrapper` signature has changed to add the `rel` argument:

	abstract public function getPageLinkWrapper($url, $page, $rel = null);

### Iron.Io Queue Encryption

If you are using the Iron.io queue driver, you will need to add a new `encrypt` option to your queue configuration file:

    'encrypt' => true

<a name="upgrade-4.1.29"></a>
## Upgrading To 4.1.29 From <= 4.1.x

Laravel 4.1.29 improves the column quoting for all database drivers. This protects your application from some mass assignment vulnerabilities when **not** using the `fillable` property on models. If you are using the `fillable` property on your models to protect against mass assignment, your application is not vulnerable. However, if you are using `guarded` and are passing a user controlled array into an "update" or "save" type function, you should upgrade to `4.1.29` immediately as your application may be at risk of mass assignment.

To upgrade to Laravel 4.1.29, simply `composer update`. No breaking changes are introduced in this release.

<a name="upgrade-4.1.26"></a>
## Upgrading To 4.1.26 From <= 4.1.25

Laravel 4.1.26 introduces security improvements for "remember me" cookies. Before this update, if a remember cookie was hijacked by another malicious user, the cookie would remain valid for a long period of time, even after the true owner of the account reset their password, logged out, etc.

This change requires the addition of a new `remember_token` column to your `users` (or equivalent) database table. After this change, a fresh token will be assigned to the user each time they login to your application. The token will also be refreshed when the user logs out of the application. The implications of this change are: if a "remember me" cookie is hijacked, simply logging out of the application will invalidate the cookie.

### Upgrade Path

First, add a new, nullable `remember_token` of VARCHAR(100), TEXT, or equivalent to your `users` table.

Next, if you are using the Eloquent authentication driver, update your `User` class with the following three methods:

	public function getRememberToken()
	{
		return $this->remember_token;
	}

	public function setRememberToken($value)
	{
		$this->remember_token = $value;
	}

	public function getRememberTokenName()
	{
		return 'remember_token';
	}

> **Note:** All existing "remember me" sessions will be invalidated by this change, so all users will be forced to re-authenticate with your application.

### Package Maintainers

Two new methods were added to the `Illuminate\Auth\UserProviderInterface` interface. Sample implementations may be found in the default drivers:

	public function retrieveByToken($identifier, $token);

	public function updateRememberToken(UserInterface $user, $token);

The `Illuminate\Auth\UserInterface` also received the three new methods described in the "Upgrade Path".

<a name="upgrade-4.1"></a>
## Upgrading To 4.1 From 4.0

### Upgrading Your Composer Dependency

To upgrade your application to Laravel 4.1, change your `laravel/framework` version to `4.1.*` in your `composer.json` file.

### Replacing Files

Replace your `public/index.php` file with [this fresh copy from the repository](https://github.com/laravel/laravel/blob/master/public/index.php).

Replace your `artisan` file with [this fresh copy from the repository](https://github.com/laravel/laravel/blob/master/artisan).

### Adding Configuration Files & Options

Update your `aliases` and `providers` arrays in your `app/config/app.php` configuration file. The updated values for these arrays can be found [in this file](https://github.com/laravel/laravel/blob/master/app/config/app.php). Be sure to add your custom and package service providers / aliases back to the arrays.

Add the new `app/config/remote.php` file [from the repository](https://github.com/laravel/laravel/blob/master/app/config/remote.php).

Add the new `expire_on_close` configuration option to your `app/config/session.php` file. The default value should be `false`.

Add the new `failed` configuration section to your `app/config/queue.php` file. Here are the default values for the section:

	'failed' => array(
		'database' => 'mysql', 'table' => 'failed_jobs',
	),

**(Optional)** Update the `pagination` configuration option in your `app/config/view.php` file to `pagination::slider-3`.

### Controller Updates

If `app/controllers/BaseController.php` has a `use` statement at the top, change `use Illuminate\Routing\Controllers\Controller;` to `use Illuminate\Routing\Controller;`.

### Password Reminders Updates

Password reminders have been overhauled for greater flexibility. You may examine the new stub controller by running the `php artisan auth:reminders-controller` Artisan command. You may also browse the [updated documentation](/docs/{{version}}/security#password-reminders-and-reset) and update your application accordingly.

Update your `app/lang/en/reminders.php` language file to match [this updated file](https://github.com/laravel/laravel/blob/master/app/lang/en/reminders.php).

### Environment Detection Updates

For security reasons, URL domains may no longer be used to detect your application environment. These values are easily spoofable and allow attackers to modify the environment for a request. You should convert your environment detection to use machine host names (`hostname` command on Mac, Linux, and Windows).

### Simpler Log Files

Laravel now generates a single log file: `app/storage/logs/laravel.log`. However, you may still configure this behavior in your `app/start/global.php` file.

### Removing Redirect Trailing Slash

In your `bootstrap/start.php` file, remove the call to `$app->redirectIfTrailingSlash()`. This method is no longer needed as this functionality is now handled by the `.htaccess` file included with the framework.

Next, replace your Apache `.htaccess` file with [this new one](https://github.com/laravel/laravel/blob/master/public/.htaccess) that handles trailing slashes.

### Current Route Access

The current route is now accessed via `Route::current()` instead of `Route::getCurrentRoute()`.

### Composer Update

Once you have completed the changes above, you can run the `composer update` function to update your core application files! If you receive class load errors, try running the `update` command with the `--no-scripts` option enabled like so: `composer update --no-scripts`.

### Wildcard Event Listeners

The wildcard event listeners no longer append the event to your handler functions parameters. If you require finding the event that was fired you should use `Event::firing()`.
