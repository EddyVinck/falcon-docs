---
title: Analytics
---

## Google Analytics

To add Google Analytics to your Falcon application you need to configure it in your `client/config/default.json`:

```json
{
  "googleAnalytics": {
    "__typename": "ConfigGoogleAnalytics",
    "trackerID": "<Your Google Analytics Code>"
  }
}
```

Then you need to add the following code to your `App.js`:

Then you need to add the following code to your `App.js`:

```jsx
import withPageview from "@deity/falcon-client/src/components/withPageview";

const App = () => (
  ...
);

export default withPageview(App);
```

That's it! No further configuration necessary in Falcon or Google Analytics.

## Google Tag Manager

[See more](https://marketingplatform.google.com/about/tag-manager/)

### Configuration

you can configure Google Tag Manager via `config` property in `config/default.json`.

`googleTagManager: object`

- `id: string`: (default `null`) Google Tag Manager ID

```json
{
  "googleTagManager": {
    "id": "id"
  }
}
```
