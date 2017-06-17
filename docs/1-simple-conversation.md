## Getting Started with the Nexmo Conversation JS SDK

In this getting started guide we'll demonstrate how to build a simple conversation app with IP messaging using the Nexmo Conversation JavaScript SDK. In doing so we'll touch on concepts such as Nexmo Applications, JWTs, Users, Conversations and conversation Members.

### Before you being

* Ensure you have Node.JS installed
* Create a free Nexmo account - [signup](https://dashboard.nexmo.com)

### Setup

Install the Nexmo CLI:

```bash
$ npm install -g nexmo-cli@beta
```

Setup the CLI to use your Nexmo API Key and API Secret. You can get these from the [setting page](https://dashboard.nexmo.com/settings) in the Nexmo Dashboard.

```bash
$ nexmo setup api_key api_secret
```

Create an application within the Nexmo platform.

```bash
$ nexmo app:create "My Convo App" http://your-domain.com/answer http://your-domain.com/event --type=rtc --keyfile=private.key
```

Nexmo Applications contain configuration for the application that you are building. The output of the above command will be something like this:

```bash
Application created: 2c59f277-5a88-4fab-88c4-919ee28xxxxx
Private Key saved to: private.key
```

The first item is the Application Id and the second is a private key that is used generate JWTs that are used to authenticate your interactions with Nexmo.

Create a JWT using your Application Id.

```bash
$ APP_JWT="$(nexmo jwt:generate ./private.key application_id=YOUR_APP_ID)"
```

*Note: The above command saves the generated JWT to a `APP_JWT` variable.*

Create a user who will participate within the conversation.

```bash
curl -X POST https://api.nexmo.com/beta/users\
  -H 'Authorization: Bearer '$APP_JWT \
  -H 'Content-Type:application/json' \
  -d '{"name":"jamie"}'
```

Generate a JWT for the user and take a note of it.

```bash
nexmo jwt:generate ./private.key sub=jamie acl='{"paths": {"/v1/sessions/**": {}, "/v1/users/**": {}, "/v1/conversations/**": {}}}' application_id=YOUR_APP_ID
```

### Create the JavaScript App

Install the Nexmo Conversation JS SDK

```bash
$ npm install nexmo-conversation
```

Create an `index.html` page and add a very basic UI for the conversation functionality.

```html
<style>
#messages { width: 80%; height: 300px; }
#message { width: 80%; height: 30px; float: left; }
#send { height: 30px; }
</style>
<div id="messages"></div>
<textarea id="message"></textarea>
<button id="send">Send</button>
```

Include the Conversation JS SDK

```html
<script src="./node_modules/nexmo-conversation/dist/conversationClient.js"></script>
```

Define a variable with a value of the User JWT that was created earlier. Also define a variable to represent a `nexmo-chat` conversation.

```html
<script>
var userToken = "USER_TOKEN";
var conversation = {name: 'nexmo-chat'};
```

Create an instance of the `ConversationClient` and login the current user in using the User JWT.

```js
var rtc = new ConversationClient({debug: false});
rtc.login(userToken).then(function(app) {
  console.log('*** Logged into app', app);
}).catch(function(error) {
  console.error(error);
});
```

The next step is to have a user become a member of a conversation by joining it. A user can be a member of many conversations and the Conversation API persists that membership across user sessions. Therefore it can't be assumed that the user isn't already a member of the `nexmo-chat` conversation.

So, the first thing to do is get a list of conversations that the logged-in user is either already a member or has been invited to join.

```js
rtc.login(userToken).then(function(app) {
  console.log('*** Logged into app', app);

  // Get a list of conversations within the application
  return app.getConversations();
}).then(function(conversations) {
  console.log('*** Retrieved conversations', conversations);
}).catch(function(error) {
  console.error(error);
});
```

Then find if the conversation that we are looking to join is within the list of existing conversations that the user is a member of. If it does exist we resolve with that conversation. If the conversation does not exist we resolve by creating a new conversation.

```js
}).then(function(conversations) {
  console.log('*** Retrieved conversations', conversations);

  // Check if the user is either already a member or has been invited to join
  var uuid = Object.keys(conversations).find(function(id) {
    return conversations[id].name === conversation.name;
  });
  if(uuid !== undefined) {
    // Conversation exists
    console.log('*** Conversation exists with UUID', uuid);
    return conversations[uuid];
  }
  else {
    // It doesn't exist. Create it.
    console.log('*** Creating a new conversation', conversation);
    return rtc.app.newConversation(conversation);
  }
}).catch(function(error) {
  console.error(error);
});
```

If the user is already a member of the conversation through a previous session there is no need to join it. Otherwise we must join the user to the conversation and make them a member.

```js
    console.log('*** Creating a new conversation', conversation);
    return rtc.app.newConversation(conversation);
  }
}).then(function(conv) {
  conversation = conv;

  console.log('*** Joining conversation', conversation, conversation.me);
  if(conversation.me && conversation.me.state === 'JOINED') {
    // Already a member of the conversation
    return conversation.me;
  }
  else {
    // Need to join
    return conversation.join(rtc.app.me);  
  }
}).catch(function(error) {
  console.error(error);
});
```

Once the member has joined the channel we want to listen for `text` event on the `conversation` and show them in the UI.

```js
    // Need to join
    return conversation.join(rtc.app.me);  
  }
}).then(function(member) {
  console.log('*** Joined as ' + member.name);

  conversation.on('text', function(sender, message) {
    console.log('*** Message received', sender, message);
    var messagesEl = document.getElementById('messages');
    var text = sender.name + ': ' + message.text + '\n' +
               messagesEl.innerText;
    messagesEl.innerText = text;
  });
}).catch(function(error) {
  console.error(error);
});
```

Finally, when the user clicks the `send` button in the UI send whatever text has been placed in the `textarea`.

```js
}).catch(function(error) {
  console.error(error);
});
document.getElementById('send')
  .addEventListener('click', function() {
    var message = document.getElementById('message').value;
    conversation.sendText(message);
  }, false);
</script>
```

Run `index.html` in two side-by-side browser windows to see the conversation take place.

## Where next?

* Have a look at the [Nexmo Conversation JS SDK API Reference](https://conversation-js-docs.herokuapp.com/)
* Take a look at the next getting started guide to learn [how to invite users to a conversation](../../getting-started-2.md).
