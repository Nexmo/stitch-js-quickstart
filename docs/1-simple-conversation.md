# Getting Started with the Nexmo Conversation JS SDK

In this getting started guide we'll demonstrate how to build a simple conversation app with IP messaging using the Nexmo Conversation JavaScript SDK.

## Concepts

This guide will introduce you to the following concepts.

* **Nexmo Applications** - contain configuration for the application that you are building
* **JWTs** ([JSON Web Tokens](https://jwt.io/)) - the Conversation API uses JWTs for authentication. JWTs contain all the information the Nexmo platform needs to authenticate requests. JWTs also contain information such as the associated Applications, Users and permissions.
* **Users** - users who are associated with the Nexmo Application. It's expected that Users will have a one-to-one mapping with your own authentication system.
* **Conversations** - A thread of conversation between two or more Users.
* **Members** - Users that are part of a conversation.

## Before you begin

* Ensure you have [Node.JS](https://nodejs.org/) installed
* Create a free Nexmo account - [signup](https://dashboard.nexmo.com)
* Install the Nexmo CLI:

    ```bash
    $ npm install -g nexmo-cli@beta
    ```

    Setup the CLI to use your Nexmo API Key and API Secret. You can get these from the [setting page](https://dashboard.nexmo.com/settings) in the Nexmo Dashboard.

    ```bash
    $ nexmo setup api_key api_secret
    ```

## 1 - Setup

_Note: The steps within this section can all be done dynamically via server-side logic. But in order to get the client-side functionality we're going to manually run through setup._

### 1.1 - Create a Nexmo Application

Create a Nexmo application within the Nexmo platform to use within this guide.

```bash
$ nexmo app:create "My Convo App" https://example.com.com/answer https://example.com/event --type=rtc --keyfile=private.key
```

The output of the above command will be something like this:

```bash
Application created: 2c59f277-5a88-4fab-88c4-919ee28xxxxx
Private Key saved to: private.key
```

The first item is the Application ID which you should take a note of. We'll refer to this as `YOUR_APP_ID` later. The second value is a private key location. The private key is used to generate JWTs that are used to authenticate your interactions with Nexmo.

### 1.2 - Generate an Application JWT

Generate a JWT using your Application ID (`YOUR_APP_ID`).

```bash
$ APP_JWT="$(nexmo jwt:generate ./private.key application_id=YOUR_APP_ID exp=$(($(date +%s)+86400)))"
```

*Note: The above command saves the generated JWT to a `APP_JWT` variable. It also sets the expiry of the JWT (`exp`) to one day from now.*

### 1.3 - Create a Conversation

Create a conversation within the application:

```bash
$ curl -X POST https://api.nexmo.com/beta/conversations\
 -H 'Authorization: Bearer '$APP_JWT -H 'Content-Type:application/json' -d '{"name":"nexmo-chat", "display_name": "Nexmo Chat"}'
```

This will result in a JSON response that looks something like the following. Take a note of the `id` attribute as this is the unique identifier for the conversation that has been created. We'll refer to this as `CONVERSATION_ID` later.

```json
{"id":"CON-8cda4c2d-9a7d-42ff-b695-ec4124dfcc38","href":"http://conversation.local/v1/conversations/CON-8cda4c2d-9a7d-42ff-b695-ec4124dfcc38"}
```

### 1.4 - Create a User

Create a user who will participate within the conversation.

```bash
$ curl -X POST https://api.nexmo.com/beta/users\
  -H 'Authorization: Bearer '$APP_JWT \
  -H 'Content-Type:application/json' \
  -d '{"name":"jamie"}'
```

The output will look as follows:

```json
{"id":"USR-9a88ad39-31e0-4881-b3ba-3b253e457603","href":"http://conversation.local/v1/users/USR-9a88ad39-31e0-4881-b3ba-3b253e457603"}
```

Take a note of the `id` attribute as this is the unique identifier for the user that has been created. We'll refer to this as `USER_ID` later.

### 1.5 - Add the User to the Conversation

Finally, let's add the user to the conversation that we created. Remember to replace `CONVERSATION_ID` and `USER_ID` values.

```bash
$ curl -X POST https://api.nexmo.com/beta/conversations/CONVERSATION_ID/members\
 -H 'Authorization: Bearer '$APP_JWT -H 'Content-Type:application/json' -d '{"action":"join", "user_id":"USER_ID", "channel":{"type":"app"}}'
```

The response to this request will confirm that the user has `JOINED` the "Nexmo Chat" conversation.

```json
{"id":"MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d","user_id":"USR-f4a27041-744d-46e0-a75d-186ad6cfcfae","state":"JOINED","timestamp":{"joined":"2017-06-17T22:23:41.072Z"},"channel":{"type":"app"},"href":"http://conversation.local/v1/conversations/CON-8cda4c2d-9a7d-42ff-b695-ec4124dfcc38/members/MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d"}
```

You can also check this by running the following request, replacing `CONVERSATION_ID`:

```bash
$ curl https://api.nexmo.com/beta/conversations/CONVERSATION_ID/members\
 -H 'Authorization: Bearer '$APP_JWT
```

Where you should see a response similar to the following:

```json
[{"user_id":"USR-f4a27041-744d-46e0-a75d-186ad6cfcfae","name":"MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d","user_name":"jamie","state":"JOINED"}]
```

### 1.6 - Generate a User JWT

Generate a JWT for the user and take a note of it. Remember to change the `YOUR_APP_ID` value in the command.

```bash
$ USER_JWT="$(nexmo jwt:generate ./private.key sub=jamie exp=$(($(date +%s)+86400)) acl='{"paths": {"/v1/sessions/**": {}, "/v1/users/**": {}, "/v1/conversations/**": {}}}' application_id=YOUR_APP_ID)"
```

*Note: The above command saves the generated JWT to a `USER_JWT` variable. It also sets the expiry of the JWT to one day from now.*

You can see the JWT for the user by running the following:

```bash
$ echo $USER_JWT
```

## 2 - Create the JavaScript App

With the basic setup in place we can now focus on the client-side application.

### 2.1 - An HTML Page with a Basic UI

Create an `index.html` page and add a very basic UI for the conversation functionality.

The UI contains:

* A simple login area. We'll be stubbing out a fake login process, but in a real application it would be expected for you to integrate with your chosen login system.
* A list of messages. All the messages will be output to this area.
* An input area. We'll use this to send a new message

```html
<style>
    #login, #messages {
        width: 80%;
        height: 300px;
    }

    .messages {
        display: none;
    }

    #message {
        width: 80%;
        height: 30px;
        float: left;
    }

    #send {
        height: 30px;
    }
</style>

<section class="login">
    <h1>Login</h1>
    <form id="login">
        <input type="text" id="username" placeholder="Username" required />
        <input type="submit" value="Login" />
    </form>
</section>

<section class="messages">
    <h1>Messages</h1>
    <div id="messages"></div>
    <textarea id="message"></textarea>
    <button id="send">Send</button>
</section>
```

### 2.2 - Add the Nexmo Conversation JS SDK

Install the Nexmo Conversation JS SDK

```bash
$ npm install nexmo-conversation
```

Include the Conversation JS SDK

```html
<script src="./node_modules/nexmo-conversation/dist/conversationClient.js"></script>
```

### 2.3 - Stubbed Out Login

Next, let's stub out the login workflow.

Define a variable with a value of the User JWT that was created earlier and set the value to the `USER_JWT` that was generated earlier. Create a `CONVERSATION_ID` with the value of the Conversation ID that was created earlier to indicate the conversation we're going to be using. Also, create a variable called `conversation` that will eventually reference the conversation object.

```html
<script>
var USER_JWT = 'YOUR USER JWT';

var CONVERSATION_ID = 'YOUR CONVERSATION ID';

var conversation = null;
</script>
```

Create an `authenticate` function that takes a `username`. For now, stub it out to always return the `USER_JWT` value. Also create a `login` function that takes a `userToken` (a JWT).

```html
<script>
var USER_JWT = 'YOUR USER JWT';

var CONVERSATION_ID = 'YOUR CONVERSATION ID';

var conversation = null;

function authenticate(username) {
  return USER_JWT;
}

function login(userToken) {
}
</script>
```

Next, bind to `submit` events on the form element with the ID of `login`. When this form is submitted get the value of the `username` input and call `authenticate` to get the user token. Finally, hide the login elements, show the message elements and call `login`, passing the user token.

```js
function login(userToken) {
}

document.getElementById('login')
    .addEventListener('submit', function(e) {
        e.preventDefault();

        var username = document.getElementById('username').value;
        var userToken = authenticate(username);
        if(userToken) {
            document.getElementsByClassName('messages')[0].style.display = 'block';
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
      document.getElementById('message').value = '';
    }, false);

}
</script>
```

Run `index.html` in two side-by-side browser windows to see the conversation take place.

## Where next?

* Have a look at the [Nexmo Conversation JS SDK API Reference](https://conversation-js-docs.herokuapp.com/)
