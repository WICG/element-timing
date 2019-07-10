# Element Timing: Explainer

The **Element Timing** API enables monitoring when developer-specified image elements or groups of text nodes are displayed on screen. The way text nodes are grouped is described below.


### Objectives

1.  Inform developers when specific elements are first displayed on the screen. We support the following types of images: \<img\>, \<image\> inside \<svg\>, and background-images. We also support observing certain groups of text nodes. Web developers can better understand which are the critical images and text of their sites, so after annotating them the browser can provide timing information about those. This image and text converage means that the majority of web content can be timed via this API.
1.  Enable analytics providers to measure display time of key images or text, without explicit opt in from web developers. In many cases it's not feasible for developers to modify their HTML just to get better performance insights, so it's important to provide basic information even for websites that cannot annotate their elements.


### How do we register elements for observation?

An element can be registered for observation via the `elementtiming` HTML attribute. Having this attribute set will signal the browser to expose render timing information for this element (its source image, background image, and/or text). It should be noted that setting the `elementtiming` attribute does not work retroactively: once an element has loaded and is rendered, setting the attribute will have no effect. Thus, it is strongly encouraged to set the attribute before the element is added to the document (in HTML, or if set on Javascript, before adding it to the document). Having the attribute implies that:

* If the element is an image, there will be an entry for the image.
* If the element is affected by (possibly multiple) background images, there will be one entry for each of those background images.
* If the element is associated to at least one text node, there will be one entry for the group of associated text nodes.

### Image considerations

We define the <a name="image-time">**image rendering timestamp**</a> as the next paint that occurs after the image has become fully loaded. This is important to distinguish as progressively rendered images may have multiple initial renderings before the image has even been fully received by the user agent.

Allowing third-party origins to measure the time an arbitrary image resource takes to render could expose sensitive information such as whether a user is logged into a website. Therefore, for privacy and security reasons, the [image rendering timestamp](#image-time) is only exposed in entries corresponding to resources that pass the [timing allow check](https://w3c.github.io/resource-timing/#dfn-timing-allow-check). However, to enable a more holistic picture, the rest of the information is exposed for arbitrary images.

### Text considerations

We say that a text node <a name="belong">**belongs to**</a> its [containing block](https://www.w3.org/TR/CSS2/visudet.html#containing-block-details). This means that an element could have 0 or many associated text nodes with it.

We say that an element is *text-painted* if at least one text node [belongs to](#belong) and has been painted at least once. Thus, the <a name="text-time">**text rendering timestamp**</a> of an element is the time when it becomes *text-painted*.

Let the *text rect* of a text node be the display rectangle of that node within the viewport. We define the <a name="text-rect">**text rect**</a> of an element as the smallest rectangle which contains the geometric union of the text rects of all text nodes which [belong to](#belong) the element.

### What information is exposed?

A `PerformanceElementTiming` entry has the following attributes:
* `name`: for images, "image-paint". For text: "text-paint".
* `entryType`: it will always be the string "element".
* `startTime`: for images, the [image rendering timestamp](#image-time), or 0 when the resource does not pass the [timing allow check](https://w3c.github.io/resource-timing/#dfn-timing-allow-check). For text, the [text rendering timestamp](#text-time).
* `duration`: it will always be set to 0.
* `intersectionRect`: for images, the display rectangle of the image within the viewport. For text, the [text rect](#text-rect) of the associated text (only counting text nodes which have been painted at least once).
* `responseEnd`: for images, the timestamp of when the last byte of the resource response was received, same as ResourceTiming's [responseEnd](https://w3c.github.io/resource-timing/#dom-performanceresourcetiming-responseend). For text, 0.
* `identifier`: the value of the `elementtiming` attribute of the element.
* `naturalWidth`: the [intrinsic](https://drafts.csswg.org/css2/conform.html#intrinsic) width of the image. It matches with the corresponding DOM [attribute](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-naturalwidth) for img. 0 for text.
* `naturalHeight`: the [intrinsic](https://drafts.csswg.org/css2/conform.html#intrinsic) height of the image. It matches with the corresponding DOM [attribute](https://html.spec.whatwg.org/multipage/embedded-content.html#dom-img-naturalheight) for img. 0 for text.
* `id`: the element's ID.
* `element`: points to the element. This will be "null" if the element is [disconnected](https://dom.spec.whatwg.org/#connected).
* `url`: for images, the initial URL for the resource request. For text, this will be an empty string.

Note: for background images, the element is the one being affected by the background image style.

Sample code:

```javascript
<img src="my_image.jpg" elementtiming="foobar">

const observer = new PerformanceObserver((list) => {
  let perfEntries = list.getEntries().forEach(function(entry) {
      // Send the information to analytics, or in this case just log it to console.
      // |entry.startTime| contains the timestamp of when the image is displayed.
      if (entry.identifier === 'foobar')
        console.log("My image took " + entry.startTime + " to render!");
   });
});
observer.observe({entryTypes: ['element']});
```

### Questions

#### What about Shadow DOM?

Unfortunately Shadow DOM elements are not currently supported. 

#### What about invisible or occluded elements?

The entry creation might be affected by _visibility_: for instance, elements are not exposed if the style visibility is set to none, or the opacity is 0. However, occlusion will not affect entry creation: an entry is seen if the element is there, but hidden by a full-screen pop-up on the page.

#### What are some differences between text and image observation?

* Whereas for images it is OK to care only about those which are associated to a resource, for text that is definitely not the case. An image not associated to a resource will need to be constructed manually and it is uncommon for such an image to be part of the key content of a website. Note that we consider inline images (those with data URIs) to have an associated resource, it is just inlined. On the other hand, text is relevant regardless of whether it is associated to a webfont or not.

* Image rendering steps are different from text rendering steps. For images, initial paints may not include all of the image content and may be low quality. It is only once we have fully loaded and decoded the image that we can be certain that the content being displayed is meaningful. In contrast, any text painted is meaningful. It should be noted that this is not perfect because there could be block level elements containing some text that renders quickly and some text that is blocked on webfonts. In this case, for the purposes of Element Timing, the text blocked by webfonts will be completely ignored.

* Text is actually simpler when it comes to security and privacy considerations. Cross-origin images could reveal a lot of information about users, but text and webfonts embedded in a website do not. Therefore, whereas for images we had to consider resources that passed the timing allow check versus resources that did not, for text this distinction is not needed.

