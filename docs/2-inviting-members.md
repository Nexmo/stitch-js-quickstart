# Getting Started with the Nexmo Conversation JS SDK

In this getting started guide we'll cover creating a second user and inviting them to the Conversation we created in the [simple conversation](1-simple-conversation.md) getting started guide. From there we'll list the conversations that are available to the user and upon receiving an invite to new conversations we'll automatically join them.

## Concepts

This guide will introduce you to the following concepts.

* **Invites** - you can invite users to a conversation
* **Application Events** - `member:invited` events that fire on an Application, before you are a Member of a Conversation
* **Conversation Events** - `member:joined` and `text` events that fire on an Conversation, after you are a Member


## Before you begin

* Ensure you have run through the [previous guide](1-simple-conversation.md)


## 1 - Setup

_Note: The steps within this section can all be done dynamically via server-side logic. But in order to get the client-side functionality we're going to manually run through the setup._

### 1.1 - Create another User

If you're continuing on from the previous guide you may already have a `APP_JWT`. If not, generate a JWT using your Application ID (`YOUR_APP_ID`).

```bash
$ APP_JWT="$(nexmo jwt:generate ./private.key application_id=YOUR_APP_ID exp=$(($(date +%s)+86400)))"
```

Create another user who will participate within the conversation.

```bash
$ curl -X POST https://api.nexmo.com/beta/users\
  -H 'Authorization: Bearer '$APP_JWT \
  -H 'Content-Type:application/json' \
  -d '{"name":"alice"}'
```

The output will look as follows:

```json
{"id":"USR-9a88ad39-31e0-4881-b3ba-3b253e457603","href":"http://conversation.local/v1/users/USR-9a88ad39-31e0-4881-b3ba-3b253e457603"}
```

Take a note of the `id` attribute as this is the unique identifier for the user that has been created. We'll refer to this as `USER_ID` later.

### 1.2 - Generate a User JWT

Generate a JWT for the user. The JWT will be stored to the `SECOND_USER_JWT` variable. Remember to change the `YOUR_APP_ID` value in the command.

```bash
$ SECOND_USER_JWT="$(nexmo jwt:generate ./private.key sub=alice exp=$(($(date +%s)+86400)) acl='{"paths": {"/v1/sessions/**": {}, "/v1/users/**": {}, "/v1/conversations/**": {}}}' application_id=YOUR_APP_ID)"
```

*Note: The above command sets the expiry of the JWT to one day from now.*

You can see the JWT for the user by running the following:

```bash
$ echo $SECOND_USER_JWT
```

## 2 - Update the JavaScript App

We will use the application we already created for [the first getting started guide](1-simple-conversation.md). With the basic setup in place we can now focus on updating the client-side application.

### 2.1 - Add placeholder UI to list Conversations

Update `index.html` with a placeholder section to list conversations.


```html
  <style>
    .conversations {
        display: none;
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
...
var conversation = null;

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

Next, update the login form handler to show the conversation elements instead of the message elements when the form is submitted.

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

### 2.3 - Update the JS needed to list the Conversations

In the previous quick start guide we retrieved the conversation from the list of existing conversations that the user is a member of using a hard-coded `CONVERSATION_ID`. This time we're going to list the conversations that the user is a member, allowing the user to select the conversation they want to join. We're going to replace the following part in the code:

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

With the following that takes the list of conversations and saves it in the `conversationList` variable we created earlier. It then proceeds in creating the HTML wrapper element for the conversations. The code then cycles through the conversations, creating HTML elements for each of them, binding a click event listener, `selectConversation` to each of them, and then adds the element to the wrapper element. Finally, the code checks if there were any conversations added, and if not lists a message and then appends the wrapper element to the UI.

```js
    }).then(function(conversations) {
        console.log('*** Retrieved conversations', conversations);
        conversationList = conversations

        var conversationsElement = document.createElement("ul");
        for (var id in conversations) {
            var conversationElement = document.createElement("li");
            conversationElement.textContent = conversations[id].display_name;
            conversationElement.addEventListener("click", selectConversation.bind(id));
            conversationsElement.appendChild(conversationElement);
        }

        if (!conversationsElement.childNodes.length) {
            conversationsElement.textContent = "You are not a member of any conversation"
        }

        document.getElementsByClassName('conversations')[0].appendChild(conversationsElement)
    }).catch(function(error) {
        console.error(error);
    });

}
```

### 2.4 - Implement the handler for `selectConversation`

Create a new method `selectConversation` which will handle the user clicking on a Conversation name. When the user selects a Conversation, ensure the `conversation` variable is updated to correctly reference that conversation object and we're going to hide the Conversations list and display the Messages box.  We then want to listen for `text` event on the `conversation` and show them in the UI. We'll also listen for the `member:joined` event on the `conversation` and then show a message in the UI about a member joining the conversation.

```js
function selectConversation() {
    conversation = conversationList[this];

    document.getElementsByClassName('conversations')[0].style.display = 'none';
    document.getElementsByClassName('messages')[0].style.display = 'block';

    console.log('*** Conversation Member', conversation.me);

    // Bind to events on the conversation
    conversation.on('text', function(sender, message) {
        console.log('*** Message received', sender, message);
        var messagesEl = document.getElementById('messages');
        var text = sender.name + ': ' + message.text + '\n' +
            messagesEl.innerText;
        messagesEl.innerText = text;
    });

    conversation.on("member:joined",
        function(data, info) {
            console.log("*** " + info.user.name + " joined the conversation");
            var messagesEl = document.getElementById('messages');
            var text = info.user.name + ' joined the conversation\n' +
                messagesEl.innerText;
            messagesEl.innerText = text;
        });
}
```

### 2.5 - Listening for Conversation invites and accepting them

The next step is to update the `login` method to listen on the `application` object for the `member:invited` event. Once we receive an invite, we're going to automatically join the user to that Conversation.

```js
    ...
    rtc.login(userToken).then(function(app) {
        console.log('*** Logged into app', app);

        app.on("member:invited",
            function(data, invitation) {
                //identify the sender.
                console.log("*** Invitation received:", invitation);

                //accept an invitation.
                app.getConversation(invitation.cid || invitation.body.cname)
                    .then(function(conversation_to_join) {
                        conversation_to_join.join(app.me, invitation.body.user.member_id);
                        var username = document.getElementById('username').value;
                        var userToken = authenticate(username);
                        login(userToken);
                    });
            });

        // Get a list of conversations within the application
        return app.getConversations();
    }).then(function(conversations) {
        ...

}
```

Now run `index.html` in two side-by-side browser windows, making sure to login with the user name `jamie` in one and with `alice` in the other. Focus the browser window where you're logged in with `alice`.

### 2.6 - Invite the second user to the conversations

Finally, let's invite the user to the conversation that we created. In your terminal, run the following command and remember to replace `CONVERSATION_ID` in the URL with the ID of the Conversation you created in the first guide and the `USER_ID` with the one you got when creating the User for `alice`.

```bash
$ curl -X POST https://api.nexmo.com/beta/conversations/CONVERSATION_ID/members\
 -H 'Authorization: Bearer '$APP_JWT -H 'Content-Type:application/json' -d '{"action":"invite", "user_id":"USER_ID", "channel":{"type":"app"}}'
```

The response to this request will confirm that the user has been `INVITED` the "Nexmo Chat" conversation.

```json
{"id":"MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d","user_id":"USR-f4a27041-744d-46e0-a75d-186ad6cfcfae","state":"INVITED","timestamp":{"invited":"2017-06-17T22:23:41.072Z"},"channel":{"type":"app"},"href":"http://conversation.local/v1/conversations/CON-8cda4c2d-9a7d-42ff-b695-ec4124dfcc38/members/MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d"}
```

You can also check that `alice` was invited by running the following request, replacing `CONVERSATION_ID`:

```bash
$ curl https://api.nexmo.com/beta/conversations/CONVERSATION_ID/members\
 -H 'Authorization: Bearer '$APP_JWT
```

Where you should see a response similar to the following:

```json
[{"user_id":"USR-f4a27041-744d-46e0-a75d-186ad6cfcfae","name":"MEM-fe168bd2-de89-4056-ae9c-ca3d19f9184d","user_name":"alice","state":"INVITED"}]
```

Return to the previously opened browser windows so you can see `alice` has a conversation listed now. You can click the conversation name and proceed to chat between `alice` and `jamie`.

## Where next?

* Have a look at the [Nexmo Conversation JS SDK API Reference](https://conversation-js-docs.herokuapp.com/)
