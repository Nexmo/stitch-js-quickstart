# Getting Started with the Nexmo Conversation JS SDK

In this getting started guide we'll cover adding the screen sharing capability to the Conversation we created in the simple conversation with video getting started guide.
We’ll build a Chrome extension and learn how to utilize it for capturing our screen, working with Firefox’s native desktop capture and update our media object with the screen stream.

## Concepts

This guide will introduce you to the following concepts.
- [Chrome extension](https://developer.chrome.com/extensions) - creating and submitting a dedicated chrome extension for capturing the user’s screen.

## Before you begin

- Ensure you have run through the [previous guide.](https://github.com/Nexmo/stitch-js-quickstart/blob/master/docs/5-enable-video.md)
- Ensure you are familiar with what a [Chrome extension](https://developer.chrome.com/extensions) is. It is recommended to go over the official [quick-start by Google.](https://developer.chrome.com/extensions/getstarted) We will focus on creating an extension suited for our needs and will not go over all of the available options.  

## 1 - Creating the Stitch SDK compatible Chrome extension

We will have to create a new folder for the extension itself, configured to comply with the Stitch SDK.

### 1.1 -  Creating the manifest file

We’ll have to create a manifest.json file, this file will contains the instructions on how to create the extension for Chrome. We’ll have to fill the following template and save it as manifest.json:

``` json
{
  "name": "Screen Desktop Capture Name",
  "description": "Allows you to capture your desktop for use in video applications", "version": "x.x.x",
  "manifest_version": 2,
  "permissions": [
    "desktopCapture",
    "activeTab",
    "tabs",
    "http://*/*", "https://*/*"
  ],
  "externally_connectable": {
    "matches": ["*://localhost/*","https://*.my-dns.com/*"]
  },
  "background": {
    "scripts": ["screen-capture.js"], 
    "persistent": false
  }
}
```

The ```externally_connectable``` attribute describes the patterns the extension can communicate with. The ```scripts``` attribute contains the file name of the code that the extension will execute.

### 1.2 - Adding the extension’s JavaScript code

We’ll have to create “screen-capture.js”, which will eventually be the code the extension runs. The snippet below is already configured to work the Stitch SDK and does not have to be modified.
On a ‘screenshare-extension-installed’ request, the extension will reply with a ‘success’ answer. On any other request, the extension will reply with the proper stream id that captures the screen.

```javascript
chrome.runtime.onMessageExternal.addListener((message, sender, sendResponse) => {
  If (message === 'screenshare-extension-installed') { //This is the message the sdk will send to check if the extension is installed
    sendResponse({
      type: 'success', // This is the result the sdk expects to receive
      version: '0.1.0' // Important that the version sent is '0.1.0'. This is what the sdk expects
    });
  } else {
    const sources = message.sources;
    const tab = sender.tab;
    chrome.desktopCapture.chooseDesktopMedia(sources, tab, (streamId) => { // This is the way the extension captures the screen
      if (!streamId) {
        sendResponse({
          type: 'error',
          message: 'Failed to get stream ID'
        });
      } else {
        sendResponse({
          type: 'success',
          streamId: streamId
        });
      }
    });
  }
  return true;
});
```

### 1.3 - Adding UI and additional changes

An extension can have many forms of user interfaces, as described in the [documentation](https://developer.chrome.com/extensions/getstarted#user_interface). This is the stage where all necessary files for the Stitch SDK have already been created, but more changes could be added- depending on you and your application’s needs. Feel free to add those in this stage.

### 1.4 - Testing the extension locally

This is a good time to test our extensions locally, as documented in the [quick start](https://developer.chrome.com/extensions/getstarted), make any additional changes here before submitting.

### 1.5 - Submitting the Chrome extension

After testing your extension and configuring it to comply with the Stitch SDK we’re ready to submit our extension to the Chrome Web Store. Follow the [documentation](https://developer.chrome.com/webstore/publish) to do so. Our users will now have the ability to download this extension and use it with our Stitch SDK enhanced application!

### 2 - Adding the screen share capability to our application

We will have to make two simple changes to support screen sharing!

### 2.1 -  Adding the extension’s id to ConversationClient

When we create a new ConversationClient, as described in the [previous tutorial](https://github.com/Nexmo/stitch-js-quickstart/blob/master/docs/1-simple-conversation.md), we should pass the extension ID to the constructor, as follows:

``` javascript
conversationClient = new ConversationClient({
  debug: true,
  screenshareExtensionId: 'THE_ID'});
  ```

  ### 2.2 -  Adding the extension’s id to ConversationClient

  Now, when we want to share the screen, all we have to do is update our media, as follows:

  ```javascript
  conversation.media.update({ screenshare: true});
  ```

  This will enable screen share with all the screen share option allowed by Chrome (screen, window or tab). What happens with Firefox though?

  If you want to enable the screen share with only part of the option you can call enable and pass the option you want.

  ```javascript
  conversation.media.update({screenshare: {direction: !this.screenShared, sources: ['tab']}});
  ```
