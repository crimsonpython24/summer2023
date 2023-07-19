# Chapter 9

**Representational state transer (REST)**: a set of established API guidelines; a RESTful API adheres to the REST architectural properties

## Six Constraints

| Constraint        | Explanation                                                                                                                                                          |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Client-server     | Enforces the separation of concerns; the client and server components are updated independently, which simplifies server components                                  |
| Stateless         | Each request from the client to the server must contain all the necessary info complete the request (i.e., the server cannot utilize previously stored context info) |
| Cacheable         | Any response should implicitly label itself as cacheable or non-cacheable (cacheable: reuse the response data later for euqivalent requests)                         |
| Layered system    | Allows an architecture to be composed of hierarchical layers by constraining component behavior                                                                      |
| Code on demand    | Allows for code or applets to be sent to the client by the server                                                                                                    |
| Uniform interface | Fundamental set of constraints that simplifies the system architecture                                                                                               |

Even if most modern APIs do not _totally_ comply to the six rules, they still adhere to a few architectural concepts:

| Concept            | Explanation                                                                                                                       |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| Resources          | Any object must be accessible by an **Uniform Resource Identifier (URI)**, such as a serialized object or a collection of objects |
| Request operations | HTTP operations such as `GET` `PUT` `POST` `DELETE`                                                                               |
| Error codes        | REST APIs use standard HTTP error codes                                                                                           |
| Hypermedia         | Responses may contain hyperlinks or URIs to other resources (e.g., for pagination)                                                |

## Versioning

As API changes, it's a good idea to mark down the version of APIs to track issues.

```
GET /superheroes/3 HTTP/1.1
Host: example.com
Accept: application/json
<!-- either of -->
api-version: 3
Accept: application/vnd.superhero-api.v3+json
```

## Django Rest Framework

> Some content are expected as preliminary knowledge and are omitted

### API Documentation

Invoke a human-browsable API through:

```python
from rest_framework.documentation import include_docs_urls
urlpatterns = [
  path('api-docs/', include_docs_urls(title='Superbook API')),
]
```

### Infinite Scrolling (AJAX)

1. Use JavaScript to listen to the `scroll` event
2. When the mark is reached, asynchronously request the next page link through AJAX
3. A Django service or REST API handles the link, and returns the appropriate next page link
4. The new content is appended to the page

Use an AJAX backend provided by Django that supplies the appropriate page of content (e.g. `ListView`) with the `paginate_by` parameter set to the number of objects per page.
