# Chapter 7

## States of a form

1. Empty form (unbound form)
2. Filled form (bound form)
3. Submitted form with errors (bound but not a valid form)
4. Submitted form without errors (bound and valid form)

**Note** The code in the following sections assume that `form_class` is already existent in the view class; remember to specify such an attribute in real code.

### Process

- When a user submits a form, the view callable gets the form data inside `request.POST` and gets initialized with a dictionary-like object
  - Forms defined with `method=GET` sents form data in the URL itself; only used to retrieve data
  - Forms defined with `method=POST` has form data sent along with the HTTP request, not seen by the user (e.g., creating or updating data)

```python
class PersonDetailsForm(forms.Form):
  name = forms.CharField(max_length=100)
  age = forms.IntegerField()
```

```python
class ClassBasedFormView(generic.View):
  template_name = 'form.html'
  def get(self, request):
    form = PersonDetailsForm()
    return render(request, self.template_name, {'form': form})
  def post(self, request):
    form = PersonDetailsForm(request.POST)
    if form.is_valid():
      return redirect('success')
    else:
      return render(request, self.template_name, {'form': form})
```

### Example: Dynamic Field Generation

Since form fields can be modified at runtime, adding or changing forms can be done as the form initializes
I.e., every form instance has the attribute `fields`, a dictionary that contains all the form fields

This example illustrates a case when an argument named "upgrade" will add a checkbox to a user details form.
In the `forms.py` file:

```python
class PersonDetailsForm(forms.Form):
  name = forms.CharField(max_length=100)
  age = forms.IntegerField()
  def __init__(self, *args, **kwargs):
    upgrade = kwargs.pop("upgrade", False)
    super().__init__(*args, **kwargs)
    # Show first class option?
    if upgrade:
      self.fields["first_class"] = forms.BooleanField(label="Fly First Class?")
```

Either directly call `PersonDetailsForm(upgrade=True)`, or do

```python
class PersonDetailsEdit(generic.FormView):
  def get_form_kwargs(self):
    kwargs = super().get_form_kwargs()
    kwargs["upgrade"] = True
    return kwargs
```

This pattern can also be implemented in user-based forms, i.e., when a form is only visible to a certain group of users.

### Example: Multiple Form Actions Per View

While using the same view for separate actions, it's possible to identify which form's values are submitted based on the `submit` button's name, such as through the `append` method below:

```python
class SubscribeForm(forms.Form):
  email = forms.EmailField()
  def __init__(self, *args, **kwargs):
    super().__init__(*args, **kwargs)
    self.helper = FormHelper(self)    # a special Django-crispy-form function that defines the rendering behavior
    self.helper.layout.append(Submit('subscribe_butn', 'Subscribe'))
```

```python
class NewsletterView(generic.TemplateView):
  subcribe_form_class = SubscribeForm
  unsubcribe_form_class = UnSubscribeForm
  template_name = "newsletter.html"
  def get(self, request, *args, **kwargs):
    kwargs.setdefault("subscribe_form", self.subcribe_form_class())
    kwargs.setdefault("unsubscribe_form", self.unsubcribe_form_class())
    return super().get(request, *args, **kwargs)
```

Where `setdefault` is a default python function that changes the value of a key-value pair, and inserts the key if it does not exist.

As the forms enter the template context as keyword arguments, they will be returned as a part of `request.POST` once any of the two forms is submitted:

```python
def post(self, request, *args, **kwargs):
  form_args = {
    'data': self.request.POST,
    'files': self.request.FILES,
  }
  if "subscribe_butn" in request.POST:
    form = self.subcribe_form_class(**form_args)
    if not form.is_valid():
      return self.get(request, subscribe_form=form)
    return redirect("success_form1")
  elif "unsubscribe_butn" in request.POST:
    pass
  return super().get(request)
```

### Example: CRUD Views

These views are a part of Django's generic views (`generic.[*]View`).

## Data Cleaning

1. any data that comes from the user should not be trusted initially
2. the field values in `request.POST` and `request.GET` are just strings, and a type conversion may be needed

--> simple solution: call the `.cleaned_data` attribute

```python
>>> fill = {"name": "Blitz", "age": "30"}
>>> g = PersonDetailsForm(fill)
>>> g.is_valid()
True
>>> g.cleaned_data
{'age': 30, 'name': 'Blitz'}
```

## CSRF

CSRF tokens are not recommended for `GET` forms becuase such actions should not change the server state, and forms submitted through `GET` will expose the CSRF token.

> Spoiler: since the frontend of most (of my) future applications will be written in React.js, this section could be irrelevant. But just for the sake of reading the book, I guess.
