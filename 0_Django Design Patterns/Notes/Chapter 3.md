# Chapter 3

## M > V and C

- models are classes that provide an OOP way of dealing with databases
- model modules can be imported from the command line in such cases
  - django interactive shell
  - test scripts
  - async tasks such as those through Celery

## Class Diagrams

Drawing a class diagram helps establish an early form of the database structure and offers chances for revision
**Entity-relationship model (ER-model)**: a one-to-many (`1-n`) relationship where the `n` side is where the foreign key is declared

## Normalized Models

- Helps efficiently store data: once fully normalized, there will not be any _redundancies_
  - Each model should only contain data that is logically related to it
  - I.e., there will not be multiple fields that contain data representing the same purpose
    > Pattern: design models to be fully normalized, then selectively denormalize for performance reasons

### First Normal Form

A table must have:

- No cell with multiple values
- A primary key (most commonly UID) defined as a single column or a set of columns (aka composite key)
- I.e., if there is a cell (by design or not) that has multiple values, split each value into an individual row

### Second Normal Form

A table must have:

- Everything in the first normal form (1NF)
- All non-primary key columns must be dependent on the entire primary key
- E.g., if a column is not dependent on a composite PK, then extract it into a separate table

### Third Normal Form

A table must have:

- Everything in the second normal form (2NF)
- All non-primary key columns must be independent of each other
- E.g., if a column is dependent on other non-primary key columns, extract it into a separate table

### Example

**Original table:**

| Name      | Origin      | Power               | First Used At                                              |
| --------- | ----------- | ------------------- | ---------------------------------------------------------- |
| Blitz     | Alien       | Freeze; Flight      | +40, -73, USA, 2014/07/03; +40, -73, USA, 2013/03/12       |
| Hexa      | Scientist   | Telekinesis; Flight | +35, +139, Japan, 2010/02/17; +35, +139, Japan, 2010/02/19 |
| Traveller | Billionaire | Time travel         | +43, +1, France, 2010/11/10                                |

**After 1NF:**

| Name      | Origin      | Power       | Latitude | Longitude | Country | Time       |
| --------- | ----------- | ----------- | -------- | --------- | ------- | ---------- |
| Blitz     | Alien       | Freeze      | +40      | -73       | USA     | 2014/07/03 |
| Blitz     | Alien       | Flight      | +40      | -73       | USA     | 2013/03/12 |
| Hexa      | Scientist   | Telekinesis | +35      | +139      | Japan   | 2010/02/17 |
| Hexa      | Scientist   | Flight      | +35      | +139      | Japan   | 2010/02/19 |
| Traveller | Billionaire | Time travel | +43      | +1        | France  | 2010/11/10 |

**After 2NF:**

Main table:

| Name      | Power       | Latitude | Longitude | Country | Time       |
| --------- | ----------- | -------- | --------- | ------- | ---------- |
| Blitz     | Freeze      | +40      | -73       | USA     | 2014/07/03 |
| Blitz     | Flight      | +40      | -73       | USA     | 2013/03/12 |
| Hexa      | Telekinesis | +35      | +139      | Japan   | 2010/02/17 |
| Hexa      | Flight      | +35      | +139      | Japan   | 2010/02/19 |
| Traveller | Time travel | +43      | +1        | France  | 2010/11/10 |

Origin table:

| Name      | Origin      |
| --------- | ----------- |
| Blitz     | Alien       |
| Hexa      | Scientist   |
| Traveller | Billionaire |

> Origin is dependent on `name` only, but not the composite PK `name` and `power`, therefore it is extracted into a separate table with a single PK `name`

**After 3NF:**

Main table:

| User ID | Power       | Location ID | Time       |
| ------- | ----------- | ----------- | ---------- |
| 1       | Freeze      | 1           | 2014/07/03 |
| 1       | Flight      | 1           | 2013/03/12 |
| 2       | Telekinesis | 2           | 2010/02/17 |
| 2       | Flight      | 2           | 2010/02/19 |
| 3       | Time travel | 3           | 2010/11/10 |

Origin table:

| User ID | Name      | Origin      |
| ------- | --------- | ----------- |
| 1       | Blitz     | Alien       |
| 2       | Hexa      | Scientist   |
| 3       | Traveller | Billionaire |

Location table:

| Location ID | Latitude | Longitude | Country |
| ----------- | -------- | --------- | ------- |
| 1           | +40      | -73       | USA     |
| 2           | +35      | +139      | Japan   |
| 3           | +43      | +1        | France  |

> The `latitude` and `longitude` columns are dependent on the `country` column, which is NOT a PK; therefore, the data is extracted into a separate table.
> The User ID is also replaced for convenience in this step, but it does not directly have to do with 3NF (Django will assign each user with a PK by default)

## Performance and Denormalization

_Normalize while deisgning, but denormalize while optimizing_

- As the number of models incrase, the number of joins needed to answer a query also increase
  - E.g., to find the number of superheroes with Freeze in the USA, 4 joins must be performed
  - If there is a complex query spanning several tables (e.g., count of superpowers by country) then creating a separate, denormalized table might improve performance
  - Consider denormalization if querying a field is taking too long or will be a common function in the website
- Can include other queries using Django's **object-relational mapping (ORM)**, which has functions like `all()` `exclude()` `get()` `filter()`
  - An efficient way to interact with a database through python queries, which is also available through the command line
  - Handles data of medium to low complexity

## Model Mixins

This is to resolve the case where models have the same fields/methods duplicated; solution: extract common fields and methods into reusable model mixins

**Concrete (multi-table) inheritance**: each object maps to their own database tables; an implicit join is needed each time the **base** fields are accessed

```python
class Location(models.Model):
  name = models.CharField(max_length=50)
  address = models.CharField(max_length=80)

class Restaurant(Location):
  serves_hot_dogs = models.BooleanField(default=False)
  serves_pizza = models.BooleanField(default=False)
```

**Proxy inheritance**: only adds new behavior to the parent class

```python
class Book(models.Model):
  title = models.CharField(max_length=30)
  author = models.CharField(max_length=30)

class MyBook(Book):
  class Meta:
    proxy = True
    ordering = ["title"]
```

**Abstract inheritance**: uses the special Abstract base class to share data and behavior among models; does not create corresponding tables in the database and do not need a join statement

```python
class VehicleInfo(models.Model):
  name = models.CharField(max_length=20)
  color = models.CharField(max_length=20)
  class Meta:
    abstract = True

class Vehicle(VehicleInfo):
  customer = models.ForeignKey(
    Customer,
    related_name='vehicle',
    on_delete=models.CASCADE
    )
```

### Smaller Mixins

Mixins should be easily composable and not violating the single-responsibility principle; otherwise, refactor it into smaller classes so that each mixin/inheritance layer only does one thing (and does it well)

## User Profiles

- Recommended to set the `primary_key` explicitly to `True` to prevent issues in PSQL
- Either a field must be nullable, or they should contain a default value to prevent database errors
- Once a instance is created and needs another function, e.g. creating a profile, put the handler in `signals.py` instead of also putting it in the models

```python
@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_profile_handler(sender, instance, created, **kwargs):
  if not created:
    return
  # Create the profile object, only if it is newly created
  profile = models.Profile(user=instance)
  profile.save()
```

```python
# apps.py
class ProfilesConfig(AppConfig):
  name = "profiles"
  verbose_name = 'User Profiles'
  def ready(self):
    from . import signals
```

```python
# settings.py
INSTALLED_APPS = [
  'profiles.apps.ProfilesConfig',
]
```

## Multiple Profile Types

The most common way is to use abstract classes to separate concerns, and extend all the abstract classes into a concrete class

```python
class BaseProfile(models.Model):
  # ...
  class Meta:
    abstract = True

class ProfileTypeOne(models.Model):
  # the functions/fields that only goes into this profile type
  class Meta:
    abstract = True

class ProfileTypeTwo(models.Model):
  # the functions/fields that only goes into this profile type
  class Meta:
    abstract = True

class Profile(BaseProfile, ProfileTypeOne, ProfileTypeTwo):
  pass
```

## Service Objects

> If models do more than one thing, testing and maintenance will get harder --> refactor related methods into a specialized `Service` object

"Fat models, thin views" were commonly told to beginners as views should not contain any other than _presentation logic_. Consider refactoring into a `Service` object if your models:

1. interacts with external services
2. contains helper tasks that do not deal with the database, e.g., generating a short URL or random captcha
3. making a short-lived object without a database state, e.g., JSON response
4. functionality spanning multiple model instances but doesn't belong to any
5. long-running background tasks such as Django-celery

### Example

From this:

```python
class Profile(models.Model):
  ...
  def is_superhero(self):
    url = "http://api.herocheck.com/?q={0}".format(
        self.user.username)
    return webclient.get(url)
```

Into this:

```python
# services.py
API_URL = "http://api.herocheck.com/?q={0}"
class SuperHeroWebAPI:
  @staticmethod
  def is_hero(username):
    url = API_URL.format(username)
    return webclient.get(url)
```

```python
# models.py
from .services import SuperHeroWebAPI
  def is_superhero(self):
    return SuperHeroWebAPI.is_hero(self.user.username)
```

Methods of a service object are **stateless**, meaning, they perform the action solely based on function parameters without using any class properties, and are henceforth declared as static methods.
They are also preferably self-contained, which allows them to be tested without having to create a database object but to be called directly.

## Retrieval Patterns

### Property Field

From this:

```python
class BaseProfile(models.Model):
  birthdate = models.DateField()
  # ...
  def get_age(self):
    today = datetime.date.today()
      return (today.year - self.birthdate.year) - int(
          (today.month, today.day) <
          (self.birthdate.month, self.birthdate.day))
```

(note: this function will be in `models.py` because the function needs the database fields to calculate the age, and therefore isn't static)

Into:

```python
@property
def age(self):
  # ...
```

So that the function can be invoked directly through `profile.age` instead of `profile.get_age()`.

Noteworthy, property functions are not available to the ORM, and therefore cannot be accessed through Pythonic functions or the CLI.
However, one good reason to use `property` decorators is to hide internal classes, following the **Law of Demeter (LoD)**, which states that you can only access your own direct members using _only one dot_
e.g., rather than accessing `profile.birthdate.year`, define a `profile.birthyear` property, which helps hide the underlying structure of the `birthdate` field

Adding on top of `property`, you can also cache calculations which:

1. does not change within an instance once it has finished calculating
2. are expensive to calculate or format

```python
@cached_property
def function(self):
  # Expensive operation e.g. external service call
```

Where the cached value is stored within the python instance's memory

## Custom Model Managers

This is required when certain queries or models are defined and accessed repeatedly, and that might need to be chained with other filters.

```python
# managers.py
class PostQuerySet(QuerySet):
  def public_posts(self):
    return self.filter(privacy="public")
PostManager = PostQuerySet.as_manager
```

```python
# models.py
class Post(Postable):
  # ...
  objects = PostManager()
```

So now the accessing function will only be `public = Post.objects.public_posts()`, which is considerably shorter than a one-liner using `.filter()`.
The returned QuerySet can also be chained, e.g., `public_apology = Post.objects.public_posts().filter(message_startswith="Sorry")`

## Migrations

Always run `python manage.py makemigrations <app>` and `python manage.py migrate <app>` after modifying models.
