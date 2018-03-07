# Getting Started with the Nexmo Conversation JS SDK

In this getting started guide we'll cover adding video events to the Conversation we created in the [simple conversation with audio](4-enable-audio.md) getting started guide. We'll deal with video and media stream events, the ones that come via the conversation, and the ones we send to the conversation.

## Concepts

This guide will introduce you to the following concepts.

- **Video** - enabling and disabling Video streams in a Conversation
- **Media Stream Events** - `media:stream:on` events that fire on a Member in a Conversation when media streams are available.

## Before you begin

- Ensure you have run through the [previous guide](4-enable-audio.md)

## 1 - Update the JavaScript App

We will use the application we already created for [the fourth getting started guide](4-enable-audio.md). All the basic setup has been done in the previous guides and should be in place. We can now focus on updating the client-side application.

### 1.1 - Add video UI

First, we'll add the UI for user to enable and disable video, as well as two `<video>` elements that we'll use to play the Video streams we're sending and receiving in the conversation. Let's add the UI at the top of the messages area.

```html
<section id="messages">
  <div>
    <button id="enable-video">Enable Video</button>
    <button id="disable-video">Disable Video</button>
  </div>
  <div>
    <video id="self-video" autoplay muted></video>
    <video id="conversation-video" autoplay></video>
  </div>
  ...
</section>

```

And add the buttons and `<video>` elements in the class constructor

```javascript
constructor() {
...
  this.selfVideo = document.getElementById('self-video')
  this.conversationVideo = document.getElementById('conversation-video')
  this.enableVideoButton = document.getElementById('enable-video')
  this.disableVideoButton = document.getElementById('disable-video')
}
```

### 1.2 - Add enable and disable video handler

We'll then update the `setupUserEvents` method to trigger `conversation.media.enable({video: "both"})` when the user clicks the `Enable Video` button. We'll also add the corresponding `conversation.media.enable({video: "both"})` trigger for disabling the video stream.

```javascript
setupUserEvents() {
...
  this.enableVideoButton.addEventListener('click', () => {
    this.conversation.media.enable({
      video: "both"
    }).then(this.eventLogger('member:media')).catch(this.errorLogger)
  })
  this.disableVideoButton.addEventListener('click', () => {
    this.conversation.media.disable({
      video: "both"
    }).then(this.eventLogger('member:media')).catch(this.errorLogger)
  })
}
```

### 1.3 - Add disable audio handler

Next, we'll add the ability for a user to disable the audio stream as well. In order to do this, we'll update the `setupUserEvents` method to trigger `conversation.media.disable()` when the user clicks the `Disable Audio` button.

```javascript
setupUserEvents() {
...
  this.disableButton.addEventListener('click', () => {
    this.conversation.media.disable().then(this.eventLogger('member:media')).catch(this.errorLogger)
  })
}
```

### 1.4 - Add member:media listener

With these first parts we're sending `member:media` events into the conversation. Now we're going to register a listener for them as well that updates the `messageFeed`. In order to do that, we'll add a listener for `member:media` events at the end of the `setupConversationEvents` method

```javascript
setupConversationEvents(conversation) {
  ...

  conversation.on("member:media", (member, event) => {
    console.log(`*** Member changed media state`, member, event)
    const text = `${member.user.name} <b>${event.body.audio ? 'enabled' : 'disabled'} audio in the conversation</b><br>`
    this.messageFeed.innerHTML = text + this.messageFeed.innerHTML
  })

}
```

If we want the conversation history to be updated, we need to add a case for `member:media` in the `showConversationHistory` switch:

```javascript
showConversationHistory(conversation) {
  ...
  switch (events[Object.keys(events)[i - 1]].type) {
    ...
    case 'member:media':
      eventsHistory += `${conversation.members[events[Object.keys(events)[i - 1]].from].user.name} @ ${date}: <b>${events[Object.keys(events)[i - 1]].body.audio ? "enabled" : "disabled"} audio</b><br>`
      break;
    ...
  }
}
```


### 1.5 - Open the conversation in two browser windows

Now run `index.html` in two side-by-side browser windows, making sure to login with the user name `jamie` in one and with `alice` in the other. Enable video on both and start talking. You'll also see events being logged in the browser console.

Thats's it! Your page should now look something like [this](https://github.com/Nexmo/conversation-js-quickstart/blob/master/examples/5-enable-video/index.html).

## Where next?

- Have a look at the [Nexmo Conversation JS SDK API Reference](https://developer.nexmo.com/sdk/stitch/javascript/)

## ICE Servers
If for some reason the WebRTC P2P connection doesn't work on your network, you need to overwrite the [ICE Servers](https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment) when you create you `ConversationClient` instance:

```javascript
new ConversationClient({
  debug: false,
  iceServers: {
    urls: 'turn:138.68.169.35:3478?transport=tcp',
    credential: 'bar',
    username: 'foo2'
  }
})
```
