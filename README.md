# Kentico Kontent Hacks

Useful hacks for things you might not know can be done in Kentico Kontent

## Starts with filter at the Delivery API

[Filtering in Delivery API](https://docs.kontent.ai/reference/kentico-kontent-apis-overview#content-filtering) doesn't provide any [startswith] operator. Normally, you would want it this way:

```plain
?elements.myelement[startswith]=prefix
```

Still it is possible to filter this way. Try to use the following instead to provide this behavior:

```plain
?elements.myelement[range]=prefix,prefix~
```

We are using `~` as the last printable ASCII char to give the next subsequent string outside of the range.

NOTE: In other languages or if it is likely that the value contains `~`, you may need to use some higher Unicode char encoded in the query string instead of `~`.

NOTE 2: The prefix value can't contain comma `,` not even URL encoded as the request parser can't differentiate between comma to separate range values, and comma within individial value

## Filtering value for a custom element with a richer data model

Imagine you have a custom element with JSON value containing some extra metadata needed for the default custom element visualization (which should work even without contacting the 3rd party system). Here is an example of such value (pretty-printed, the actual value is on one line):

```json
{
  "experiment": {
    "id": "YI5uxeAkRZ-0E1-7oQ5FXg",
    "name": "Home page A/B test"
  },
  "variant": {
    "id": "1",
    "name": "With blurred image"
  }
}
```

Filtering by experiment ID at Delivery API with this kind of value is impossible, because it doesn't support [contains] for string and  JSON.stringify order is not deterministic.

What you can do, however, is to make the value like this, separating the filtering value, and place it at the start of the value in a deterministic way (again, pretty-printed):

```json
[
  "YI5uxeAkRZ-0E1-7oQ5FXg",
  {
    "experiment": {
      "id": "YI5uxeAkRZ-0E1-7oQ5FXg",
      "name": "Home page A/B test"
    },
    "variant": {
      "id": "1",
      "name": "With blurred image"
    }
  }
]
```

With this, the value can be still handled with JSON.stringify and JSON.parse, but starts deterministically like this (not pretty-printed) `["YI5uxeAkRZ-0E1-7oQ5FXg"` and you can now filter by the experiment ID with the Starts with filter shown above.

```plain
?elements.myelement[range]=["YI5uxeAkRZ-0E1-7oQ5FXg",["YI5uxeAkRZ-0E1-7oQ5FXg"~
```

It's like having a cake, and eating it, too ...

## Mapping item data to JS SDK representation

It is possible to transform item data to JS SDK object representation using JS SDK delivery client's `mappingService` introduced in [this pull request](https://github.com/Kentico/kontent-delivery-sdk-js/pull/218/files).

Let's imagine the situation when your item data comes from other source than Delivery API directly, which may be some proxy for secured Delivery API, or a mock.

The value of custom element contains stringified complex JSON and you want to use [the custom model for it](https://github.com/Kentico/kontent-delivery-sdk-js/blob/master/DOCS.md#using-custom-models-for-custom-elements) or any other feature that JS SDK offers.

Following code snippet showcase transforming item data and then using custom model mapping to get the red color value from your color picker custom element.

```js
import { ElementModels, Elements } from '@kentico/kontent-delivery';

class ColorElement extends Elements.CustomElement {

    public red;
    public green;
    public blue;

    constructor(
       public elementWrapper
    ) {
      super(elementWrapper);

      const value = elementWrapper.rawElement.value; // "{\"red\":167,\"green\":96,\"blue\":197}"
      const parsed = JSON.parse(value);
      this.red = parsed.red;
      this.green = parsed.green;
      this.blue = parsed.blue;
    }
}

const item = { // Data returned from the proxy, or mock
  "item" : {
    "system": {
      "id": "ef23e568-6aa2-42cd-a120-7823c0ef19f7",
      "name": "Color",
      "codename": "color",
      "language": "en-US",
      "type": "color",
      "sitemap_locations": [],
      "last_modified": "2019-03-27T13:10:01.791Z"
    },
    "elements": {
      "picked": {
        "type": "text",
        "name": "Picked",
        "value": "\"red\":167,\"green\":96,\"blue\":197}"
      }
    }
  },
  "modular_content": {}
};

const baseResponse = {
  data: item,
  headers: [],
  item,
  status: 200,
};

const client = new DeliveryClient({
  projectId: '',
  elementResolver: (elementWrapper) => {
    if (elementWrapper.contentItemSystem.type === 'color' && elementWrapper.rawElement.name === 'picked') {
      return new ColorElement(elementWrapper);
    }

    return undefined;
  }
});
const resolvedItem = client.mappingService
  .viewContentItemResponse(baseResponse, {});

const redColorPart = resolvedItem.item.picked.red;
```
