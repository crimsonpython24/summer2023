# Chapter 6

> Note: some content is expected as preliminary knowledge and is not included in this section of notes

## The Admin View

Always remember to register the model using `admin.site.register`.

The admin view can be customized as such:

```
# admin.py
class SightingAdmin(admin.ModelAdmin):
  list_display = ('superhero', 'power', 'location', 'sighted_on')
  date_hierarchy = 'sighted_on'
  search_fields = ['superhero']
  ordering = ['superhero']
```

Some of the fields may also be set in the `Meta` subclass within a model, such as:

```
# models.py
class Sighting(models.Model):
  class Meta:
    unique_together = ("superhero", "power")
    ordering = ["-sighted_on"]
    verbose_name = "Sighting & Encounter"
    verbose_name_plural = "Sightings & Encounters"
```

The `get_absolute_url` function is handy if one will switch between the `admin` site and the corresponding page on the non-admin website.
The `verbose_name` can change the model's name from camel case to another name; specifying the plural will prevent funny grammatical errors.

> A well-defined set of `Meta` attributes will also represent better in the shell, log files, and so on

Keep the people who owns admin access into a small pool of people, since using the `admin` interface will give the user a direct view of the database,
which may be confusing/unreadable, as well as inconsistent (in cases where the frontend places specific input guidelines and stores them in a particular manner).
I.e., try to stay away from using the admin interface unless it's for simple data entry

## Feature flag

A **feature flag** is similar to a switch, to which a feature should be availble to a select group of customers

Use cases:

| Case                | Explanation                                                           | Functions (if any)    |
| ------------------- | --------------------------------------------------------------------- | --------------------- |
| Trials              | Conditionally active for certain users, e.g. beta testers             | `if flag_is_active`   |
| A/B testing         | Users are selected randomly to experience a certain feature           | `if sample_is_active` |
| Performance testing | Gradually increase the pool of users that receive a new update        |                       |
| Limit Externalities | Site-wide feature switch: e.g., if Amazon S3 is down, disable uploads | `if switch_is_active` |
