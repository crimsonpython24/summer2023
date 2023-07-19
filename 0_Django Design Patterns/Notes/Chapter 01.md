# Chapter 1

## How Django Works

1. Sends a request to web server
2. Web server (e.g. Nginx) hands the request to a WSGI server
3. WSGI servers, which can run python applications, passes the request through layers of middleware and into the Django application
4. URL configuration selects a view that handles the request and transforms it into a `HttpRequest` object
5. The view does whatever it is programmed to do
6. The `HttpResponse` object gets rendered into a string and leaves the Django application
7. The webpage is rendered

## Gang of Four (GoF) Design Patterns in Django

| GoF Pattern      | Django Component | Explanation                                                                                                              |
| ---------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Command Pattern  | HttpRequest      | Encapsulates a request in an object (hence the name: a command, the request in this case, is returned)                   |
| Observer Pattern | Signals          | Listeners are notified ("observers") and updated once one watched object changes state                                   |
| Template Pattern | CBV              | Subclassing within an algorithm (the overarching web app) that achieves a specific purpose and does not alter the parent |

## MVC (Model-View-Controller)

- Django both belongs and doesn't belong to MVC
- MVC: decouples the presentation layer from the application
  - Model: the data-related logic; e.g., data transferred between view and controller (generally, the type of data that exists in an API)
  - View: UI logic (buttons, boxes, etc.)
  - Controller: handles incoming requests and renders the final output using data from the model
    > Model updates view, user sees view, user uses controller, controller manipulates model, and so on
- Django utilizes the **Model-Template-View** (MTV) architecture; the names match the corresponding Django application feature/component
  - Each request of a webpage is indepdent from other requests due to HTTP's nature
  - Docs: "a 'view' is the Python callback function for a particular URL" and not necessarily how the data looks

## Fowler Design Patterns in Django

| Fowler Pattern          | Django Component  | Explanation                                                                                                  |
| ----------------------- | ----------------- | ------------------------------------------------------------------------------------------------------------ |
| Active Record           | Django Models     | A data source pattern that bundles database fields, access scope, and functions together in a database model |
| Class Table Inheritance | Model Inheritance | Name suggests                                                                                                |
| Identity Field          | ID Field          | Name suggests (can also use slug, if guaranteed unique)                                                      |
| Template View           | Django Templates  | Name suggests                                                                                                |
