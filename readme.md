Laravel-Translatable
====================

**If you want to store translations of your models into the database, this package is for you.**

This is a Laravel package for translatable models. Its goal is to remove the complexity in retrieving and storing multilingual model instances. With this package you write less code, as the translations are being fetched/saved when you fetch/save your instance.

### Docs

* [Demo](#demo)
* [Laravel compatibility](#laravel-compatibility)
* [Tutorials](#tutorials)
* [Installation](#installation-in-4-steps)
* [Configuration](#configuration)
* [Features list](#features-list)
* [FAQ / Support](#faq)

## Demo

**Getting translated attributes**

```php
  $article = Article::where('code', 'gr')->first();
  echo $article->translate('en')->title; // Greece
  
  App::setLocale('en');
  echo $article->title;     // Greece

  App::setLocale('de');
  echo $article->title;     // Griechenland
```

**Saving translated attributes**

```php
  $article = Article::where('created_at', '>', date("Y-m-d H:i:s"))->first();
  echo $article->translate('en')->title; // Greece
  
  $article->translate('en')->title = 'abc';
  $article->save();
  
  $article = Article::where('code', 'gr')->first();
  echo $article->translate('en')->title; // abc
```

**Filling multiple translations**

```php
  $data = [
    'code' => 'gr',
    'en'  => ['title' => 'Greece'],
    'fr'  => ['title' => 'Grèce'],
  ];

  $article = Article::create($data);
  
  echo $article->translate('fr')->title; // Grèce
```

## Laravel compatibility

 Laravel  | Translatable
:---------|:----------
 5.4      | 0.0.1*

## Tutorials

- Check the tutorial about laravel-translatable in laravel-news: [*How To Add Multilingual Support to Eloquent*](https://laravel-news.com/2015/09/how-to-add-multilingual-support-to-eloquent/)
- [How To Build An Efficient and SEO Friendly Multilingual Architecture For Your Laravel Application](https://mydnic.be/post/how-to-build-an-efficient-and-seo-friendly-multilingual-architecture-for-your-laravel-application)

## Installation in 4 steps

### Step 1: Install package

Add the package in your composer.json by executing the command.

```bash
composer require edipresse/laravel-translatable
```

Next, add the service provider to `app/config/app.php`

```
Edipresse\Translatable\TranslatableServiceProvider::class,
```

### Step 2: Migrations

In this example, we want to translate the model `Article`. We will need an extra table `country_translations`:

```php
Schema::create('countries', function(Blueprint $table)
{
    $table->increments('id');
    $table->string('code');
    $table->timestamps();
});

Schema::create('country_translations', function(Blueprint $table)
{
    $table->increments('id');
    $table->integer('country_id')->unsigned();
    $table->string('title');
    $table->string('locale')->index();

    $table->unique(['country_id','locale']);
    $table->foreign('country_id')->references('id')->on('countries')->onDelete('cascade');
});
```

### Step 3: Models

1. The translatable model `Article` should [use the trait](http://www.sitepoint.com/using-traits-in-php-5-4/) `Edipresse\Translatable\Translatable`. 
2. The convention for the translation model is `ArticleTranslation`.


```php
// models/Article.php
class Article extends Eloquent {
    
    use \Edipresse\Translatable\Translatable;
    
    public $translatedAttributes = ['title'];
    protected $fillable = ['code'];
    
    /**
     * The relations to eager load on every query.
     *
     * @var array
     */
    // (optionaly)
    // protected $with = ['translations'];

}

// models/ArticleTranslation.php
class ArticleTranslation extends Eloquent {

    public $timestamps = false;
    protected $fillable = ['title'];

}
```

The array `$translatedAttributes` contains the titles of the fields being translated in the "Translation" model.

### Step 4: Configuration

We copy the configuration file to our project.

Laravel 5.*
```bash
php artisan vendor:publish --tag=translatable 
```

*Note: There isn't any restriction for the format of the locales. Feel free to use whatever suits you better, like "eng" instead of "en", or "el" instead of "gr".  The important is to define your locales and stick to them.*

## Configuration

### The config file

You can see the options for further customization in the [config file](src/config/translatable.php).

### The translation model

The convention used to define the class of the translation model is to append the keyword `Translation`.

So if your model is `\MyApp\Models\Article`, the default translation would be `\MyApp\Models\ArticleTranslation`.

To use a custom class as translation model, define the translation class (including the namespace) as parameter. For example:

```php
<?php 

namespace MyApp\Models;

use Edipresse\Translatable\Translatable;
use Illuminate\Database\Eloquent\Model as Eloquent;

class Article extends Eloquent
{
    use Translatable;

    public $translationModel = 'MyApp\Models\ArticleAwesomeTranslation';
}

```

## Features list

**Please read the installation steps first, to understand what classes need to be created.**

### Available methods 

```php
// Before we get started, this is how we determine the default locale.
// It is set by laravel or other packages.
app()->getLocale(); // 'fr' 

// To use this package, first we need an instance of our model
$germany = Article::where('code', 'de')->first();

// This returns an instance of ArticleTranslation of using the default locale.
// So in this case, french. If no french translation is found, it returns null.
$translation = $germany->translate();

// It is possible to define a default locale per model by overriding the model constructor.
public function __construct(array $attributes = [])
{
    parent::__construct($attributes);
    
    $this->defaultLocale = 'de';
}

// It is also possible to define a default locale for our model on the fly:
$germany->setDefaultLocale('de');

// If an german translation exists, it returns an instance of 
// ArticleTranslation. Otherwise it returns null.
$translation = $germany->translate('de');

// If a german translation doesn't exist, it attempts to get a translation  
// of the fallback language (see fallback locale section below).
$translation = $germany->translate('de', true);

// Alias of the above.
$translation = $germany->translateOrDefault('de');

// Returns instance of ArticleTranslation of using the default locale.
// If no translation is found, it returns a fallback translation
// if enabled in the configuration.
$translation = $germany->getTranslation();

// If an german translation exists, it returns an instance of 
// ArticleTranslation. Otherwise it returns null.
// Same as $germany->translate('de');
$translation = $germany->getTranslation('de', true);

// To set the translation for a field you can either update the translation model.
// Saving the model will also save all the related translations.
$germany->translate('en')->name = 'Germany';
$germany->save();

// Alternatively we can use the shortcut
$germany->{'name:en'} = 'Germany';
$germany->save();

// There are two ways of inserting mutliple translations into the database
// First, using the locale as array key.
$greece = $country->fill([
    'en'  => ['name' => 'Greece'],
    'fr'  => ['name' => 'Grèce'],
]);

// The second way is to use the following syntax.  
$greece = $country->fill([
    'name:en' => 'Greece',
    'name:fr' => 'Grèce',
]);

// Returns true/false if the model has translation about the current locale. 
$germany->hasTranslation();

// Returns true/false if the model has translation in french. 
$germany->hasTranslation('fr');

// If a german translation doesn't exist, it returns
// a new instance of ArticleTranslation.
$translation = $germany->translateOrNew('de');

// Returns a new ArticleTranslation instance for the selected
// language, and binds it to $germany
$translation = $germany->getNewTranslation('it');

// The eloquent model relationship. Do what you want with it ;) 
$germany->translations();

// Remove all translations linked to an object
$germany->deleteTranslations();

// Delete one or multiple translations
$germany->deleteTranslations('de');
$germany->deleteTranslations(['de', 'en']);

// Gel all the translations as array
$germany->getTranslationsArray();
// Returns
[
 'en' => ['name' => 'Germany'],
 'de' => ['name' => 'Deutschland'],
 'fr' => ['name' => 'Allemagne'],
];

// Creates a clone and clones the translations
$replicate = $germany->replicateWithTranslations(); 

```

### Available scopes

```php
// Returns all countries having translations in english
Article::translatedIn('en')->get();

// Returns all countries not being translated in english
Article::notTranslatedIn('en')->get();

// Returns all countries having translations
Article::translated()->get();

// Eager loads translation relationship only for the default
// and fallback (if enabled) locale
Article::withTranslation()->get();

// Returns an array containing pairs of country ids and the translated
// title attribute. For example: 
// [
//     ['id' => 1, 'title' => 'Greece'], 
//     ['id' => 2, 'title' => 'Belgium']
// ]
Article::listsTranslations('title')->get()->toArray();

// Filters countries by checking the translation against the given value 
Article::whereTranslation('title', '=', 'Test Article')->first();

// Or where translation
Article::whereTranslation('title', 'like','%Gree%')
  ->orWhereTranslation('title', 'France')->get();

// Filters countries by checking the translation against the given string with wildcards
Article::whereTranslationLike('title', '%Gree%')->first();

// Or where translation like
Article::whereTranslationLike('title', '%eece%')->orWhereTranslationLike('title', '%ance%')->get();
```

### Magic properties

To use the magic properties, you have to define the property `$translatedAttributes` in your
 main model:

 ```php
 class Article extends Eloquent {

     use \Edipresse\Translatable\Translatable;

     public $translatedAttributes = ['title'];
 }
 ```

```php
// Again we start by having a country instance
$germany = Article::where('code', 'de')->first();

// We can reference properties of the translation object directly from our main model.
// This uses the default locale and is the equivalent of $germany->translate()->title
$germany->title; // 'Germany'

// We can also quick access a translation with a custom locale
$germany->{'title:de'} // 'Deutschland'
```

### Fallback locales

If you want to fallback to a default translation when a translation has not been found, enable this in the configuration
using the `use_fallback` key. And to select the default locale, use the `fallback_locale` key.

Configuration example:

```php
return [
    'use_fallback' => true,

    'fallback_locale' => 'en',    
];
```

You can also define *per-model* the default for "if fallback should be used", by setting the `$useTranslationFallback` property:

```php
class Article {

    public $useTranslationFallback = true;

}
```

#### Fallback per property 

Even though we try having all models nicely translated, some fields might left empty. What's the result? You end up with missing translations for those fields!

The property fallback feature is here to help. When enabled, translatable will return the value of the fallback language 
for those empty properties. 

The feature is enabled by default on new installations. If your config file was setup before v7.1, make sure to add 
the following line to enable the feature:

```php
'use_property_fallback' => true,
```

Of course the fallback locales must be enabled to use this feature.
 
 If the property fallback is enabled in the configuration, then translatable
 will return the translation of the fallback locale for the fields where the translation is empty. 

#### Article based fallback

Since version v5.3 it is possible to use country based locales. For example, you can have the following locales:

- English: `en`
- Spanish: `es`
- Mexican Spanish: `es-MX`
- Colombian Spanish: `es-CO`

To configuration for these locales looks like this:

```php
    'locales' => [ 
        'en',
        'es' => [
            'MX',
            'CO',
        ],
    ];
```

We can also configure the "glue" between the language and country. If for instance we prefer the format `es_MX` instead of `es-MX`, 
the configuration should look like this:

```php
   'locale_separator' => '_',
```

What applies for the fallback of the locales using the `en-MX` format? 

Let's say our fallback locale is `en`. Now, when we try to fetch from the database the translation for the 
locale `es-MX` but it doesn't exist,  we won't get as fallback the translation for `en`. Translatable will use as a 
fallback `es` (the first part of `es-MX`) and only if nothing is found, the translation for `en` is returned.
 
#### Add ons

Thanks to the community a few packages have been written to make usage of Translatable easier when working with forms:

- [Propaganistas/Laravel-Translatable-Bootforms](https://github.com/Propaganistas/Laravel-Translatable-Bootforms)
- [TypiCMS/TranslatableBootForms](https://github.com/TypiCMS/TranslatableBootForms)
 
## FAQ

#### I need some example code!

Examples for all the package features can be found [in the code](https://github.com/edipresse/laravel-translatable/tree/master/tests/models) used for the [tests](https://github.com/edipresse/laravel-translatable/tree/master/tests).

#### I need help!

Got any question or suggestion? Feel free to open an [Issue](https://github.com/edipresse/laravel-translatable/issues/new).

#### I want to help!

You are awesome! Watched the repo and reply to the issues. You will help offering a great experience to the users of the package. `#communityWorks`

#### I am getting collisions with other trait methods!

Translatable is fully compatible with all kinds of Eloquent extensions, including Ardent. If you need help to implement Translatable with these extensions, see this [example](https://gist.github.com/edipresse/9659552).

#### How do I sort by translations?

A tip here is to make the MySQL query first and then do the Eloquent one.

To fetch a list of records ordered by a translated field, you can do this: 

```mysql
SELECT * from countries
JOIN country_translations as t on t.country_id = countries.id 
WHERE locale = 'en'
GROUP BY countries.id
ORDER BY t.title desc
```

The corresponding eloquent query would be:

```php
Article::join('country_translations as t', function ($join) {
        $join->on('countries.id', '=', 't.country_id')
            ->where('t.locale', '=', 'en');
    }) 
    ->groupBy('countries.id')
    ->orderBy('t.title', 'desc')
    ->with('translations')
    ->get();
```

#### How can I select a country by a translated field?

For example, let's image we want to find the Article having a ArticleTranslation title equal to 'Portugal'.

```php
Article::whereHas('translations', function ($query) {
    $query->where('locale', 'en')
    ->where('title', 'Portugal');
})->first();
```

You can find more info at the Laravel [Querying Relations docs](http://laravel.com/docs/5.1/eloquent-relationships#querying-relations).

#### Why do I get a mysql error while running the migrations?

If you see the following mysql error:

```
[Illuminate\Database\QueryException]
SQLSTATE[HY000]: General error: 1005 Can't create table 'my_database.#sql-455_63'
  (errno: 150) (SQL: alter table `country_translations` 
  add constraint country_translations_country_id_foreign foreign key (`country_id`) 
  references `countries` (`id`) on delete cascade)
```

Then your tables have the MyISAM engine which doesn't allow foreign key constraints. MyISAM was the default engine for mysql versions older than 5.5. Since [version 5.5](http://dev.mysql.com/doc/refman/5.5/en/innodb-default-se.html), tables are created using the InnoDB storage engine by default.

##### How to fix

For tables already created in production, update your migrations to change the engine of the table before adding the foreign key constraint.

```php
public function up()
{
    DB::statement('ALTER TABLE countries ENGINE=InnoDB');
}

public function down()
{
    DB::statement('ALTER TABLE countries ENGINE=MyISAM');
}
```

For new tables, a quick solution is to set the storage engine in the migration:

```php
Schema::create('language_translations', function(Blueprint $table){
  $table->engine = 'InnoDB';
  $table->increments('id');
    // ...
});
```

The best solution though would be to update your mysql version. And **always make sure you have the same version both in development and production environment!**
