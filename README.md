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

## Mapping delivery response to JS SDK response

It is possible to use `mappingService` to [transform HTTP response to JS SDK object representation](https://github.com/Kentico/kontent-delivery-sdk-js/pull/218/files).

Imagine you have a custom element with JSON value containing some extra metadata needed for [the custom model for custom element visualization] (which should work even without contacting the 3rd party system).
Here is an showcase of getting the red color value from your color picker custom element:

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

const deliveryEndpointURL = 'https://deliver.kontent.ai/<PROJECTID>/items/<ITEM_CODENAME>';
const response = fetch(deliveryEndpointURL,
  .then((response) => response.json())
    .then((json) => {
      const baseResponse = {
        data: json,
        headers: [],
        json,
        status: 200,
      };
      const client = new DeliveryClient({
        projectId: ''.
        elementResolver: (elementWrapper: ElementModels.IElementMapWrapper) => {
          if (elementWrapper.contentItemSystem.type === 'your-content-type' && elementWrapper.rawElement.name === 'your-element-name') {
            return new ColorElement(elementWrapper);
          }

          return undefined;
        }
      });
      const resolvedItem = client.mappingService
        .viewContentItemResponse(baseResponse, {});

      const redColorPart = redresolvedItem.item['your-element-name'].red;
    });
```
