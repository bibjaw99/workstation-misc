## user.js to add preference

- open `about:support` and open the profile directory

```js
// user preference for UI stuff
user_pref("full-screen-api.ignore-widgets", true);
user_pref("browser.uidensity", 1);
user_pref("browser.tabs.closeWindowWithLastTab", false);
user_pref("browser.fullscreen.autohide", false);
// user preference to disable AI stuff
user_pref("browser.ml.enable", false);
user_pref("browser.ml.chat.enabled", false);
user_pref("browser.ml.linkPreview.enabled", false);
user_pref("browser.ml.pageAssist.enabled", false);
user_pref("browser.ml.smartAssist.enabled", false);
user_pref("extensions.ml.enabled", false);
user_pref("browser.tabs.groups.smart.enabled", false);
user_pref("browser.search.visualSearch.featureGate", false);
user_pref("browser.urlbar.quicksuggest.mlEnabled", false);
user_pref("pdfjs.enableAltText", false);
user_pref("places.semanticHistory.featureGate", false);
user_pref("sidebar.revamp", false);
```
