
## Adding examples to the Showcase
In both the folders for Channels and Shorts, you will find a `config.json` file that can be used to add more examples.
When adding an example, add the following line to the file:

```json
{ "label": "Example name", "url": "https://showcase.bbvms.com/ch/123.js", "slug": "example-name" }
```

The `slug` will be appended to the url when selecting one of the examples and should only contain lower case letter and not use spaces, but hypens instead. Each slug should also be unique. This will allow you to directly link to a specific example.
