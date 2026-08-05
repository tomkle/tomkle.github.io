
## Adding examples to the Showcase
In both the folders for Channels and Shorts, you will find a `config.json` file that can be used to add more examples.
When adding an example, add the following line to the file:

```
{ "label": "Example name", "url": "https://showcase.bbvms.com/sh/1.js?displayFormat=list", "slug": "example-name" }
```

Both folders contain a channels-config.json and shorts-config.json which you can edit to add more examples
Each example should have the following information:
Slug: Lowercased vesion of the title. Use dashes for spaces Will be added to the URL of the showcase exaples to allow for sharing examples
Title: The title will be used to show in the dropdown to select the 
URL: The url to the Channel/Shorts player without any embed code 
The logic on the pages will automatically generate the right embed code with the URL. 

