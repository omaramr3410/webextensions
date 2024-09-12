# User Scripts API

## Background

### User Scripts and User Scripts Managers

[User scripts](https://en.wikipedia.org/wiki/Userscript) are (usually relatively small) snippets of code that are injected into web pages in order to modify the page's appearance or behavior.  User script managers are a type of extension that is used to manage the collection of these user scripts, determining when and how to inject them on web pages.

User scripts can be created directly by the user or found in a number of different user script repositories around the web.

### Manifest V3

Manifest Version 3 (MV3) restricts the ability for extensions to use remotely-hosted code since it is a major security risk (cannot review or audit code that isn't in the extension package).  This is directly at odds with user script managers, which fundamentally require the execution of arbitrary code (the user scripts themselves).  This creates a feature gap between MV2 and MV3.

## Goal

User scripts satisfy an important use case for power users, allowing them to quickly and easily tweak the functionality of a web page.  Therefore, we propose adding a **new API to register user scripts with arbitrary code** targeted specifically for user scripts.  However, we will also take a number of steps to help reduce the abuse-ability of this API and ensure we don't lose the benefits of the remotely-hosted code restrictions from MV3.

### Multi-phase design

User script support will have a multiphase design and implementation process.  The initial implementation has the following goals:

- Achieving functional parity with MV2 for user scripts managers, enabling migration to Manifest V3
- Setting the foundations needed for future enhancements that will allow the browser to take more responsibility for the user script injection
- Limiting abusability

### Initial Requirements

The rest of this proposal focuses on the first iteration with the following requirements:

- **(A)** A mechanism to execute code in the main world
- **(B)** The ability to execute code (with a separate CSP) in a world different from the main world and the extension's isolated world
- **(C)** A separate user script permission
- **(D)** Communication between JavaScript worlds

## Proposal

### New Namespace

User scripting related features will be exposed in a new API namespace, tentatively named `userScripts`.  The proposal authors favor the use of a new namespace for several reasons.

1. **Better set developer expectations.**  The clear separation between user and content scripts will reduce confusion for developers for which API methods to use and what the capabilities / restrictions of each are.  This naming also more clearly communicates that this capability is not meant as a general purpose way to inject arbitrary scripts.

2. **Stronger enforcement of remotely hosted code abuse.**  A distinct namespace allows people to more clearly see where user scripts features are being used and more easily spot abuse patterns.

3. **Easier engineering implementation.**  Browser vendors should be able to restrict the execution world and different API methods behind features.

4. **Introduce a user script world with custom CSP.**  Allow user scripts to opt-in to a more secure world that is only available in this namespace.

### API Schema

#### Types

```
dictionary RegisteredUserScript {
  boolean? allFrames;
  ScriptSource[] js;
  string[]? excludeMatches;
  string id;
  string[]? matches;
  // Default to `document_idle`
  RunAt runAt;
  // Allows `USER_SCRIPT` (default) and `MAIN`
  // and returns error for `ISOLATED`.
  ExecutionWorld? world;
  // Implemented as disjunction: runs in documents whose URL matches
  // "matches" or "includeGlobs", and not "excludeMatches" nor "excludeGlobs".
  string[]? includeGlobs;
  string[]? excludeGlobs;
}
dictionary UserScriptFilter {
  string[]? ids;
}
// Must specify exactly one of: `file` or `code`.
dictionary ScriptSource {
  string? code;
  string? file;
}
```

#### Methods
When `callback` is omitted from these methods, a `Promise` is returned instead of `undefined`.

```
browser.userScripts.register(
  scripts: RegisteredUserScript[],
  callback?: function
)
browser.userScripts.unregister(
  filter?: UserScriptFilter[],
  callback?: function
)
browser.userScripts.getScripts(
  filter?: UserScriptFilter[],
  callback?: function // called with (RegisteredUserScript[]).
)
browser.userScripts.update(
  scripts: RegisteredUserScript[],
  callback?: function
)
```

### Requirements

#### A. A mechanism to execute code in the main world

- User scripts can be registered in the `MAIN` world.
- Extension can customize the `USER_SCRIPT` world's CSP to inject a script tag into the host page (expanded in requirement B)

#### B. The ability to execute code (with a separate CSP) in a world different from the main world and the extension's isolated world

##### `USER_SCRIPT` World

- Exempt from the page's CSP, can customize its own CSP (e.g can allow injection of `unsafe-inline`).
- Can communicate with the extension using extension messaging APIs (expanded under [Messaging](#Messaging)).
- Isolated from the web page (similar to other isolated worlds), but will be largely un-permissioned. It will not have access to any extension APIs, except messaging APIs mentioned, and will not have any cross-origin exceptions (these don't exist in Chrome, but do in other browsers).  In the future, we may bring more APIs (such as chrome.dom APIs or a dedicated API to execute a script in the main world), but these APIs will not grant any additional privilege to the script beyond what it has to access the page content.
- Can communicate with different JS worlds via `window.postMessage()`. A dedicated API to communicate between worlds is being considered as part of future work.
- Shares DOM with the web page.  Code from both worlds cannot directly interact with each other, except through DOM APIs.
- When an asymmetric security relationship may exist, the `MAIN` world is considered to be less privileged than the `USER_SCRIPT` world

##### Configuration

```
browser.userScripts.configureWorld({
  csp?: string,
  messaging?: boolean,
})
```
where
- If csp is defined, it is used in the `USER_SCRIPT` world. Otherwise, the `ISOLATED` world CSP is used.
- If `messaging` is true, messaging APIs are exposed. Otherwise, if false or undefined, messaging APIs are not exposed.
- Configuration persists across sessions.

In the future, if we allow multiple user script worlds (see section in Future Work below), this method can be expanded to allow for a user script world identifier to customize a single user script world.

##### Messaging

User scripts can send messages to the extension using extension messaging APIs: `browser.runtime.sendMessage()` and `browser.runtime.connect()`. We leverage the runtime API (instead of introducing new userScripts.onMessage- and userScripts.sendMessage-style values) in order to keep extension messaging in the same API.  There is precedent in this (using the same API namespace to send messages from a different (and less trusted) context, as `chrome.runtime` is also the API used to send messages from web pages.
Extensions can receive messages from user scripts with new event handlers: `browser.runtime.onUserScriptMessage()` and `browser.runtime.onUserScriptConnect()`. We want new events instead of using `browser.runtime.onMessage()` to make it clear the message is coming from a user script in a less-trusted context.  There is precedent for this in the form of onMessageExternal and onConnect external.

#### C. A separate user script permission

- A new extension permission will be added, and will be required in order to register user scripts.  This permission should be scoped specifically to the purpose of user scripts (as opposed to a general "remotely-hosted code" permission).
- Browser vendors then can use it to:
  - Inform their review pipeline that this is a particularly risky extension, and extra caution should be taken to ensure this is a valid use of the API.
  - Reduce the likelihood of extensions using it for other purposes and allows browser vendors to act on any permissions that do as being non-compliant
  - Present the user with different permission warnings, UI, or other gating, if they deem it necessary.

#### D. Communication between JavaScript worlds

As mentioned in requirement A, the user script world can communicate with different JS worlds via [`window.postMessage()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) and DOM APIs.  New communication methods are being considered as potential future enhancements. This is clunky and imperfect, but allows extensions to migrate to MV3 leveraging the same logic they have in MV2 today.  New, dedicated communication methods are being considered as potential future enhancements.

###  Other considerations

- When multiple scripts match and have the same runAt schedule, the execution order is:
  - Scripts registered via the content_scripts key in the manifest file
  - Scripts registered via [`scripting.registerContentScripts()`](https://developer.chrome.com/docs/extensions/reference/scripting/#method-registerContentScripts), following the order they were registered in. Updating a content script doesn't change its registration order.
  - Scripts registered via `userScripts.register()`, following the order they were registered in. Updating a user script doesn’t change its registration order.
- User scripts are always persisted across sessions, since the opposite behavior would be uncommon. (We may explore providing an option to customize this in the future.)

### Browser level restrictions

From here, each browser vendor should be able to implement their own restrictions.  Chrome is exploring limiting the access to this API when the user has enabled developer mode (bug), but permission grants are outside of the scope of this API proposal.

## (Potential) Future Enhancements

### `USER_SCRIPT`/ `ISOLATED` World Communication

In the future, we may want to provide a more straightforward path for communication with a `USER_SCRIPT` world (as opposed to the [`window.postMessage()`](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage) approach highlighted above).

### Separate `USER_SCRIPT` Worlds

In addition to specifying the execution world of `USER_SCRIPT`, we could allow extensions to inject in unique worlds by providing an identifier.  Scripts injected with the same identifier would inject in the same world, while scripts with different world identifiers inject in different worlds.  This would allow for greater isolation between user scripts (if, for instance, the user had multiple unrelated user scripts injecting on the same page).

### Execute user scripts one time

Currently, user scripts are registered and executed every time it matches the origin in a persistent way.  We may explore a way to execute a user script only one time to provide a new capability to user scripts (e.g `browser.userScripts.execute()`).

### Establish common behaviors for the CSP of scripts injected into the main world by an extension

Create certain HTML elements even if their src, href or contents violates CSP of the page so that the users don't have to nuke the site's CSP header altogether.

### Dedicated API to execute code in the `MAIN` world from another world

Provide an explicit API for an extension to execute code on the main world from a `USER_SCRIPT` or `ISOLATED` world.

### Direct access to the `MAIN` world from another world

Provide a way for the `USER_SCRIPT` or `ISOLATED` world to directly access the MAIN world (e.g. via an iframe contentWindow-like object).

## Discussion Guidelines

This is the first iteration of the userScripts API designed in https://github.com/w3c/webextensions/issues/279.
Future enhancements can be tracked in the WECG [issue tracker](https://github.com/w3c/webextensions/issues).
