# Element Timing V1 Explainer

tdresser@, npm@

Last updated July 5, 2018

The *Element Timing* API enables monitoring when large or developer-specified image elements are displayed on screen.


### Objectives

1.  Inform developers when specific 'hero' elements are first displayed on the screen. To keep the first version of this API simpler and have an easier path towards standardization, we restrict to HTML 'img' elements. Web developers can better understand which are the critical images of their sites, so after annotating them the browser can provide timing information about those elements.
1.  Enable analytics providers to measure display time of key images, without explicit opt in from web developers. In many cases it's not feasible for developers to modify their HTML just to get better performance insights, so it's important to provide basic information even for websites that cannot annotate their elements.


### How do we register images for observation?

There are two ways an image can be registered for observation: via an HTML attribute and when the image takes a large portion of the viewport.

#### Attributes

When an 'img' element with the 'elementtiming' attribute is added to the DOM, it will be observed. It should be noted that setting the elementtiming attribute does not work retroactively: once an element has loaded and is rendered, setting the attribute will have no effect. Thus, it is strongly encouraged to set the attribute before the element is added to the document (in HTML, or if set on Javascript, before adding it to the document). Sample code:

```
Example:
<img src="my_image.jpg" elementtiming="foobar">

const observer = new PerformanceObserver((list) => {
  let perfEntries = list.getEntries().forEach(function(entry) {
      // Send the information to analytics, or in this case just log it to console.
      // |entry.startTime| contains the timestamp of when the image is displayed.
      console.log("My image took " + entry.startTime + " to render!");
   });
});
observer.observe({entryTypes: ['element']});
```

This is the preferred method of annotating hero elements, as it gives developers the power to decide which elements they consider important; however, in order to provide additional flexibility, especially for RUM analytics providers, we propose including one additional approach for registering elements.


#### Implicit registration of large image elements

We register a subset of HTML images by default to allow analytics providers to gather information without having to request HTML changes from sites. We use images that occupy a large percentage of the viewport upon being rendered. In particular, upon rendering, we register images that occupy a significant percentage of the viewport (say 15% as a placeholder, exact value TBD).

### Questions

#### What about occluded elements?

The entry creation will not be based strictly on _visibility_: for instance if the element is there, but hidden by a full-screen pop-up on the page, or is the style visibility is set to none, or the opacity is 0.

