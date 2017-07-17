# Getting Started with the Nexmo Conversation JS SDK

In this getting started guide we'll demonstrate how to add users to the simple conversation app we built.

## Concepts

This guide will introduce you to the following concepts.

* **Invites** - you can invite users to a conversation
* **Application Events** - events that fire on an Application, before you are a Member of a Conversation
* **Conversation Events** - events that fire on an Conversation, after you are a Member


## Before you being

* Ensure you have run through the [previous guide](1-simple-conversation.md)


## 1 - Setup

_Note: The steps within this section can all be done dynamically via server-side logic. But in order to get the client-side functionality we're going to manually run through setup._

### 1.1 - Create another User

Create another user who will participate within the conversation.

```bash
$ curl -X POST https://api.nexmo.com/beta/users\
  -H 'Authorization: Bearer '$APP_JWT \
  -H 'Content-Type:application/json' \
  -d '{"name":"alex"}'
```

The output will look as follows:

```json
{"id":"USR-9a88ad39-31e0-4881-b3ba-3b253e457603","href":"http://conversation.local/v1/users/USR-9a88ad39-31e0-4881-b3ba-3b253e457603"}
```

Take a note of the `id` attribute as this is the unique identifier for the user that has been created. We'll refer to this as `USER_ID` later.

### 1.2 - Generate a User JWT

Generate a JWT for the user and take a note of it. Remember to change the `YOUR_APP_ID` value in the command.

```bash
$ SECOND_USER_JWT="$(nexmo jwt:generate ./private.key sub=alex exp=$(($(date +%s)+86400)) acl='{"paths": {"/v1/sessions/**": {}, "/v1/users/**": {}, "/v1/conversations/**": {}}}' application_id=YOUR_APP_ID)"
```

*Note: The above command saves the generated JWT to a `SECOND_USER_JWT` variable. It also sets the expiry of the JWT to one day from now.*

You can see the JWT for the user by running the following:

```bash
$ echo $SECOND_USER_JWT
```

### 1.3 - Add the User to the Conversation

Finally, let's invite the user to the conversation that we created. Remember to replace `CONVERSATION_ID` and `USER_ID` values.

```bash
$ curl -X POST https://api.nexmo.com/beta/conversations/CONVERSATION_ID/members\
 -H 'Authorization: Bearer '$APP_JWT -H 'Content-Type:application/json' -d '{"action":"invite", "user_id":"USER_ID", "channel":{"type":"app"}}'
```

The response to this request will confirm that the user has been `INVITED` the "Nexmo Chat" conversation.

```json
{"id":"MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d","user_id":"USR-f4a27041-744d-46e0-a75d-186ad6cfcfae","state":"INVITED","timestamp":{"invited":"2017-06-17T22:23:41.072Z"},"channel":{"type":"app"},"href":"http://conversation.local/v1/conversations/CON-8cda4c2d-9a7d-42ff-b695-ec4124dfcc38/members/MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d"}
```

You can also check this by running the following request, replacing `CONVERSATION_ID`:

```bash
$ curl https://api.nexmo.com/beta/conversations/CONVERSATION_ID/members\
 -H 'Authorization: Bearer '$APP_JWT
```

Where you should see a response similar to the following:

```json
[{"user_id":"USR-f4a27041-744d-46e0-a75d-186ad6cfcfae","name":"MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d","user_name":"alex","state":"INVITED"}]
```


## 2 - Update the JavaScript App

We will use the application we already created for [the first getting started guide](1-simple-conversation.md). With the basic setup in place we can now focus on updating the client-side application.

### 2.1 - Add placeholder UI to list Conversations

Update `index.html` with a placeholder section to list conversations.


```html
  <style>
    .conversations {
        display: block;
    }
  </style>
  <section class="conversations">
      <h1>Conversations</h1>
  </section>

```

### 2.2 - Update the stubbed Out Login

Now, let's update the login workflow to accommodate a second user.

Define a variable with a value of the second User JWT that was created earlier and set the value to the `SECOND_USER_JWT` that was generated earlier. Also, create a variable called `conversationList` that will eventually reference the conversations object.

```html
<script>
...var conversation = null;

var SECOND_USER_JWT = 'SECOND USER JWT';

var conversationList = null;
</script>
```

Update the `authenicate` function. We'll  return the `USER_JWT` value if the `username` is `'jamie'` or `SECOND_USER_JWT` for any other `username`.

```html
<script>
var conversationList = null;

function authenticate(username) {
  return username.toLowerCase() === "jamie" ? USER_JWT : SECOND_USER_JWT;
}
</script>
```

Next, update the login form to show the conversation elements instead of the message elements when the form is submitted.

```js

document.getElementById('login')
    .addEventListener('submit', function(e) {
        e.preventDefault();

        var username = document.getElementById('username').value;
        var userToken = authenticate(username);
        if(userToken) {
            document.getElementsByClassName('conversations')[0].style.display = 'block';
            document.getElementsByClassName('login')[0].style.display = 'none';
            login(userToken);
        }
        else {
            alert('unknown user ' + username);
        }
    }, false);
```

### 2.4 - Connect and Login to Nexmo

Within the `login` function, create an instance of the `ConversationClient` and login the current user in using the User JWT.

```js
function login(userToken) {

  var rtc = new ConversationClient({debug: false});
  rtc.login(userToken).then(function(app) {
    console.log('*** Logged into app', app);
  }).catch(function(error) {
    console.error(error);
  });

}
```

### 2.5 - Accessing the Conversation Object

The next step is to have a user to retrieve the Conversation that was created. A user can be a member of many conversations and the Conversation API persists that membership across user sessions.

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

}
```

Then retrieve the conversation from the list of existing conversations that the user is a member of. If it does exist we resolve with that conversation. If the conversation does not exist we throw an `Error`.

```js
  }).then(function(conversations) {
      console.log('*** Retrieved conversations', conversations);

      if (conversations[CONVERSATION_ID] !== undefined) {
          console.log('*** Conversation found', conversations[CONVERSATION_ID].name, CONVERSATION_ID);

          // Resolve the conversation
          return conversations[CONVERSATION_ID];
      }
      else {
          throw new Error('*** Could not find expected conversation', CONVERSATION_ID);
      }
  }).catch(function(error) {
    console.error(error);
  });

}
```

### 2.6 - Receiving and Sending `text` Events

Once we have found the conversation, ensure the `conversation` variable is updated to correctly reference that conversation object. We then want to listen for `text` event on the `conversation` and show them in the UI.

```js
  }).then(function(conv) {
      conversation = conv;

      console.log('*** Conversation Member', conversation.me);

      // Bind to events on the conversation
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

}
```

Finally, when the user clicks the `send` button in the UI send whatever text has been placed in the `textarea`. This is achieved by calling `sendText` on the `conversation` reference.

```js
  }).catch(function(error) {
    console.error(error);
  });
  document.getElementById('send')
    .addEventListener('click', function() {
      var message = document.getElementById('message').value;
      conversation.sendText(message);
    }, false);

}
</script>
```

Run `index.html` in two side-by-side browser windows to see the conversation take place.

## Where next?

* Have a look at the [Nexmo Conversation JS SDK API Reference](https://conversation-js-docs.herokuapp.com/)
