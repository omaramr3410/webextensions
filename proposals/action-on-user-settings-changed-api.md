# Extension API Proposal: action.onUserSettingsChanged

## Background

Proposal based on [issue #346](https://github.com/w3c/webextensions/issues/346): New API: UserSettings.isOnToolbar change event

## API Overview

### Objective

Fire an event when user-specified settings relating to an extension's action changed.

### Use Cases

#### Pinning / unpinning extension

Extensions can know if its action is pinned by calling `chrome.action.getUserSettings`. If the extension wants to know when this value changes, it has to poll the API method. Instead, it could register for an event.

## Implementation

### API Schema

#### Types

```javascript
dictionary UserSettingsChange {
  isOnToolbar?: boolean
}
```

#### Events

```javascript
// Fires when user-specified settings relating to an extension's action changed.
<browser>.action.onUserSettingsChanged.addListener(
  callback: function, // where callback contains the user site setting changed: (change: UserSettingsChange) => void
);

// Deregisters the event listener.
<browser>.action.onUserSettingsChanged.removeListener(
  callback: function, // where callback looks like: () => void
);
```

### New Permissions

| Permission Added | Proposed Warning |
| ---------------- | ------------------------------------------------------- |
| N/A              | This is an addition to the action API, which is exposed if the extension includes the "action" key in the manifest. |

### Design and Implementation

Currently, `UserSettings` only has one property. This makes things easy, as we just need to trigger the event when such property changes. If new properties are added, we will trigger the event every time any of the properties change and return an object with the modified values.

### Other Alternatives Considered

Provide both newValue and oldValue in the change event so extension’s event handler can be (more) stateless. However, we decided to go with the alternative selected because:

- Given there is only one property and that property is a (never undefined) boolean, that functionally gets us the same benefit as the more broad interface
- It more resembles most other APIs events (with the exception of the storage API events)

## User Experience

This doesn't introduce any new elements, but does tell the extension an UI element may have changed.

## Security and Privacy

This API does not directly expose any sensitive data to the extension.
