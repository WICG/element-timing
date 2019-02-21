# Element Timing for Images: Explainer

npm@, tdresser@

The **Element Timing** API enables monitoring when large or developer-specified image elements are displayed on screen.


### Objectives

1.  Inform developers when specific 'hero' elements are first displayed on the screen. To keep the first version of this API simpler and have an easier path towards standardization, we restrict to image elements: \<img\>, \<image\> inside \<svg\>, and background-images. Web developers can better understand which are the critical images of their sites, so after annotating them the browser can provide timing information about those elements.
1.  Enable analytics providers to measure display time of key images, without explicit opt in from web developers. In many cases it's not feasible for developers to modify their HTML just to get better performance insights, so it's important to provide basic information even for websites that cannot annotate their elements.


### How do we register images for observation?

There are two ways an image can be registered for observation: via an HTML attribute and when the image takes a large portion of the viewport when it is first displayed. For background-image: when it affects the style of an element which has the HTML attribute, then all of the images from that background-image are registered for observation (since background-image can include multiple URLs, multiple images may be registered for observation).

#### Attributes

When an element with the 'elementtiming' attribute is added to the DOM, it will be observed if it is an image, and its background-images will be observed if it has them. It should be noted that setting the 'elementtiming' attribute does not work retroactively: once an element has loaded and is rendered, setting the attribute will have no effect. Thus, it is strongly encouraged to set the attribute before the element is added to the document (in HTML, or if set on Javascript, before adding it to the document).

#### Implicit registration of large image elements

We register a subset of HTML images by default to allow RUM analytics providers to gather information without having to request HTML changes from sites. We use images that occupy a large percentage of the viewport upon being rendered. In particular, upon rendering, we register images that occupy a significant percentage of the viewport (say 15% as a placeholder, exact value TBD).

### What information is exposed?

a PerformanceElementTiming entry has the following relevant attributes:
* |name|: the initial URL for the resource request.
* |startTime|: the rendering timestamp.
* |intersectionRect|: the display rectangle of the image within the viewport.
* |responseEnd|: the timestamp of when the last byte of the resource response was received, same as ResourceTiming's [responseEnd](https://w3c.github.io/resource-timing/#dom-performanceresourcetiming-responseend).

Sample code:

```
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

#### Origin restrictions

Allowing third-party origins to measure the time an arbitrary image resource takes to render could expose certain private content such as whether a user is logged into a website. Therefore, for privacy and security reasons, only rendering timing for entries corresponding to resources that pass the [timing allow check](https://w3c.github.io/resource-timing/#dfn-timing-allow-check) are exposed.

However, to enable a more holistic picture, the rest of the information is exposed for arbitrary images. That is, its |startTime| will be 0 but the other attributes will be set as described above.

### Questions

#### What about occluded elements?

The entry creation will not be based strictly on _visibility_: for instance if the element is there, but hidden by a full-screen pop-up on the page, or is the style visibility is set to none, or the opacity is 0.

