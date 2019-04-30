# kc-hacks
Useful hacks for things you might not know can be done in Kentico Cloud

## Starts with filter at the Delivery API

[Filtering in Delivery API](https://developer.kenticocloud.com/reference#content-filtering) doesn't provide any [startswith] operator. Normally, you would want it this way:

```
?elements.myelement[startswith]=prefix
```

Still it is possible to filter this way. Try to use the following instead to provide this behavior:

```
?elements.myelement[range]=prefix,prefix~
```

We are using `~` as the last printable ASCII char to give the next subsequent string outside of the range.

NOTE: In other languages or if it is likely that the value contains `~`, you may need to use some higher Unicode char encoded in the query string instead of `~`.

## Filtering value for a custom element with a richer data model

Imagine you have a custom element with JSON value containing some extra metadata needed for the default custom element visualization (which should work even without contacting the 3rd party system). Here is an example of such value (pretty-printed, the actual value is on one line):

```
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

```
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

```
?elements.myelement[range]=["YI5uxeAkRZ-0E1-7oQ5FXg",["YI5uxeAkRZ-0E1-7oQ5FXg"~
```

It's like having a cake, and eating it, too ...
