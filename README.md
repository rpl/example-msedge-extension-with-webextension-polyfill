Example MSEdge extension with webextension-polyfill
===================================================

- [test-msedge-extension example](https://github.com/rpl/example-msedge-extension-with-webextension-polyfill/tree/master/test-msedge-extension):
  a minimal webextension example which shows how to include the webextension-polyfill into a "legacy" (non Chromium-based) MSEdge version.

As you may already know, [Microsoft has recently announced that its development of MSEdge has been stopped, and it will be replaced by a
browser based on Chromium in the near future](https://blogs.windows.com/windowsexperience/2018/12/06/microsoft-edge-making-the-web-better-through-more-open-source-collaboration/)

This means that the MSEdge versions based on Chromium will likely support the extensions APIs as they work on Chrome, instead of being an indipendent
implementation of the proposed Browser Extension APIs.

But in the meantime, the current MSEdge extension APIs implementation is exposed as a `browser` global as in Firefox, but it is lacking the expected support for the
Promise-based APIs (as provided in Firefox on the `browser` API namespaces), and unfortunately this makes the [webextension-polyfill](https://github.com/mozilla/webextension-polyfill)
unable to generate the "Promised API wrappers" as it does on Chromium-based browser, e.g. See:

- https://github.com/mozilla/webextension-polyfill/issues/3
- https://github.com/mozilla/webextension-polyfill/pull/114

Applying chances to the webextension-polyfill specific to the current MSEdge browser, which doesn't have many users and many supported extensions, isn't very compelling
([even more given that there is no CI service that is able to run integration tests on MSEdge, as MSEdge can't be installed in Windows Server systems](https://superuser.com/questions/1247448/how-to-install-microsoft-edge-on-windows-server-2016)) and so the above
webextensions-polyfill issue and pull request has been closed as WONTFIX.

Nevertheless, if you are an extension developer that has to still work on an MSEdge extension until the switch to Chromium actually happpens then,
(as ["the cover of the 'The Hitchhiker's Guide to the Galaxy' says"](https://en.wikipedia.org/wiki/The_Hitchhiker%27s_Guide_to_the_Galaxy)):

      Don't panic! :-)

there is still hope for you: the `--ms-reload` manifest key may help with that.

## MSEdge Extensions' "--ms-preload" manifest key

MSEdge has a weird and not very documented feature: the `"--ms-preload"` manifest key:

```
{
  "manifest_version": 2,
  "name": "test-msedge-extension",
  ...
  "-ms-preload": {
    "backgroundScript": "backgroundScriptsAPIBridge.js",
    "contentScript": "contentScriptsAPIBridge.js"
  }
}
```

The `--ms-preload` key is used to specify a JS file to preload in the related extension contexts, and it is used by the [Microsoft Edge Extension Toolkit](https://docs.microsoft.com/en-us/microsoft-edge/extensions/guides/porting-chrome-extensions) to inject a script that creates more Chromium-compatible API wrappers
on top of the Edge APIs.

By tweaking the two scripts provided by the "Microsoft Edge Extension Toolkit", we can set the `browser` global to `undefined` right aftert the Chromium-compatible API wrappers
has been generated and ensuring that the webextension-polyfill will be able to generate the Promised-based API wrappers as expected, e.g. in the backgroundScriptsAPIBridge.js
we can add the following snippet near the end of the script:

```js
var chrome = new EdgeBackgroundBridge();

// Set the original browser property to undefined to enable the webextension-polyfill
// to polyfill the generated chrome "bridged" APIs.
Object.defineProperty(this, "browser", {
    value: undefined,
});
```

As a small side note, the content scripts defined in the manifest appears to run before the contentScript preload, and so we need to be sure that the "contentScriptsAPIBridge.js"
script is loaded before the webextension-polyfill by explicitly including it in the related manifest property:

```js
{
  ...
  "content_scripts": [
    {
      "matches": [
        "https://developer.mozilla.org/*"
      ],
      "js": [
        "contentScriptsAPIBridge.js",
        "browser-polyfill.js",
        "content.js"
      ]
    }
  ],
  ...
}
```

And change the "contentScriptAPIBridge.js" script to prevent it from generating the Chromium-compatible API bridge twice for the same extension context, e.g.:

```js
// Do not run the Edge bridge if it has been already executed.
if (!this.myBrowser) {
  // All the Microsoft's contentScriptAPIBridge.js goes here.
  ...

  var chrome = new EdgeContentBridge();

  // Set the original browser property to undefined to enable the webextension-polyfill
  // to polyfill the generated chrome "bridged" APIs.
  Object.defineProperty(this, "browser", {
    value: undefined,
  });
}
```

In the [test-msedge-extension example](https://github.com/rpl/example-msedge-extension-with-webextension-polyfill/tree/master/test-msedge-extension) directory included in this repo,
there is a minimal test extension that shows the approach described above.