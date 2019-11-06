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

```jsx
import withPageview from "@deity/falcon-client/src/components/withPageview";

const App = () => (
  ...
);

export default withPageview(App);
```

That's it! No further configuration necessary in Falcon or Google Analytics.

### Advanced Google Analytics usage

Falcon exposes a `withAnalytics` higher-order component in order to send custom analytics data to Google Analytics. This component provides the `ga` property for your component, which can be used to send custom events:

```jsx
import React from "react";
import withAnalytics from "@deity/falcon-client/src/components/withAnalytics";

class Custom extends React.Component {
  handleClick(e) {
    // "ga" provided by "withAnalytics" HOC
    const { ga } = this.props;
    ga.send("event", {
      ec: "My event category",
      ea: "My event action",
      el: "My event label",
      ev: "My event value"
    });
  }

  render() {
    return <button onClick={() => this.handleClick()}>Click me!</button>;
  }
}

export default withAnalytics(Custom);
```

Please refer to a complete list of hit types [here](https://github.com/lukeed/ganalytics/#gasendtype-params).

## Google Tag Manager

[See more](https://marketingplatform.google.com/about/tag-manager/)

### Configuration

you can configure Google Tag Manager via `config` property in `config/default.json`.

`googleTagManager: object`

- `id: string`: (default `null`) Google Tag Manager ID

```json
{
  "googleTagManager": {
    "__typename": "ConfigGoogleTagManager",
    "id": "id"
  }
}
```

This is all you need to do to add Google Tag Manager to your Falcon project. Falcon-Client will handle the rest automatically.
