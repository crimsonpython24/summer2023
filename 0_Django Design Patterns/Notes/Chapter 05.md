# Chapter 5

> Note: some content is expected as preliminary knowledge and is not included in this section of notes

## How Templates Work

Django renders templates while being the actual template engine:

1. A **Loader** object in the backend will search for the template
2. If the **Loader** is successful, this will result in a **Template** object, which contains the parsed and compiled template
3. To render a **Template**, the **Context** object is necessary, and it behaves like a dictionary
4. The `render` method of a **Template** object receives the context and renders the output
