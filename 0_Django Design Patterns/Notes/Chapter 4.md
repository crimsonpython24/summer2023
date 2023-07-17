# Chapter 4

> Note: some content is expected as preliminary knowledge and is not included in this section of notes

## CBV

HTTP methods that are not explicitly defined/inherited in the method will not be included

```
class HelloView(View):
  def get(self, request, name="World"):
    return HttpResponse("Hello {}!".format(name))

path('hello-cl/', views.HelloView.as_view()),
```

```
>>> from django.test import Client
>>> c = Client()
>>> c.get("http://0.0.0.0:8000/hello-cl/").content
b'Hello World!'
>>> c.post("http://0.0.0.0:8000/hello-cl/").content
Method Not Allowed (POST): /hello-cl/
b''
```

- The biggest advantage of using a class is clear when it comes to customization
- Generic views are often implemented in an OOP manner, which you can use with just a bit of tweaking
- Generic views and CBVs can be rewritten using third-party libraries or custom code
- Illustrates the DRY (don't repeat yourself) rule through methods such as inheritance and mixins (later in this section)

| Type   | Class Name        |
| ------ | ----------------- |
| Base   | View              |
| Base   | TemplateView      |
| Base   | RedirectView      |
| List   | ListView          |
| Detail | DetailView        |
| Edit   | FormView          |
| Edit   | CreateView        |
| Edit   | UpdateView        |
| Edit   | DeleteView        |
| Edit   | ArchiveIndexVIew  |
| Date   | YearArchiveView   |
| Date   | MonthArchiveView  |
| Date   | WeekArchiveView   |
| Date   | DayArchiveView    |
| Date   | TodayArchiveView  |
| Date   | DateDetailView    |
| Auth   | LoginView         |
| Auth   | LogoutView        |
| Auth   | Password[...]View |

## View Mixins

- Mixins: parentless classes in Python 3 that takes advantage of Python's multiple inheritance property
- Will be flexible and applicable to many situations
- Left parameters' functions will override those on the right
- Always check mixins' source code to ensure that there are no method or context-variable clashes, as the ability to combine classes may become confusing

### Example: User Authentication

In cases where code expressions are preferable (e.g., clarity, readability), it is recommended to write the mixin instead of using a stock decorator

```
class LoginRequiredMixin:
  def dispatch(self, request, *args, **kwargs):
    if not request.user.is_authenticated():
      raise PermissionDenied
    return super().dispatch(request, *args, **kwargs)
```

Mixins may be utilized in cases like:

```
class UserProfileView(LoginRequiredMixin, DetailView):
  # This view will be seen only if you are logged-in
  pass
class LoginFormView(AnonymousRequiredMixin, FormView):
  # This view will NOT be seen if you are loggedin
  authenticated_redirect_url = "/feed"
```

Or rather, customize such a mixin if desired:

```
class CheckOwnerMixin:
  def get_object(self, queryset=None):
    obj = super().get_object(queryset)
    # can replace with other things, e.g., if user.is_staff
    if not obj.owner == self.request.user:
      raise PermissionDenied
    return obj
```

> _Principles of Least Privilege_: give users the least number of privileges as possible, and be explicit on which group of users can perform a certain action

### Example: Context Enhancers

Think of it as service objects, but for views instead of models

```
class FeedMixin(object):
  def get_context_data(self, **kwargs):
    context = super().get_context_data(**kwargs)
    context["feed"] = models.Post.objects.viewable_posts(
        self.request.user)
    return context
```

Here, the `get_context_data` method

1. first creates the context from the keyword arguments; `context` will be a dictionary consisting of the key-value pairs that enter the function
2. then updates the context dictionary with the `feed` variable, and finally passed to the template context if desired

## Designing URLs

Always name your Django URLs so that they can be reverse-searched in the Django CLI and search engine-indexable

> It is possible to completely switch to the simplified Django syntax over regular expressions for pattern matching, as the latter sacrifices readability and have their limitations in certain cases.

Test for parameters' values as such:

```
# pattern: `year/<int:year>/`
class YearView(View):
  def get(self, request, year):
    try:
      d = datetime(year=year, month=1, day=1)
      reply = "First day of the year {} is {}!".format(
          year, d.ctime())
    except ValueError:
      reply = "Error: Invalid year!"
    return HttpResponse(reply)
```

### Namespaces

```
>>> from django.urls import reverse
>>> reverse("hello_fn")
/hello-fn/
>>> reverse("year_view", kwargs={"year":"793"})
/year/793/
>>> reverse("viewschapter:hello_fn")    # for imports
/hello-fn/
```

Also make sure that the namespace is unique, as functions such as `redirect("about_fn")` will have to rely on such fields being unique

Avoid using **HTTP Verbs** e.g., `GET` `POST` `PUT` `PATCH` `DELETE` `SUBMIT` as a part of the URL, since they are not part of the URL itself but rather HTTP actions.
In these cases, try to find a noun substitution instead (or any verb that dos not hold a special Django/HTTP significance)
