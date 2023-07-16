# Chapter 4

> Note: some content is expected as preliminary knowledge and is not included in this section of notes

## CBV

- The biggest advantage of using a class is clear when it comes to customization
- Generic views are often implemented in an OOP manner, which you can use with just a bit of tweaking
- Generic views and CBVs can be rewritten using third-party libraries or custom code
- Illustrates the DRY (don't repeat yourself) rule

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

- Mixins: often parentless classes in Python 3 that takes advantage of Python's multiple inheritance property
- Will be flexible and applicable to many situations; left parameters' functions will override those on the right

### Example: User Authentication

```
class LoginRequiredMixin:
  def dispatch(self, request, *args, **kwargs):
    if not request.user.is_authenticated():
      raise PermissionDenied
    return super().dispatch(request, *args, **kwargs)

class UserProfileView(LoginRequiredMixin, DetailView):
  # This view will be seen only if you are logged-in
  pass
```

> _Principles of Least Privilege_: give users the least number of privileges as possible, and be explicit on which group of users can perform a certain action

### Example: Context Enhancers

Think of it as service objects, but for views instead of models

## URLs

Always name your Django URLs so that they can be reverse-searched

```
>>> from django.urls import reverse
>>> reverse("hello_fn")
/hello-fn/
>>> reverse("year_view", kwargs={"year":"793"})
/year/793/
```
