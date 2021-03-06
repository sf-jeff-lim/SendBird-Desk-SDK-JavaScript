# SendBird Desk SDK Integration Guide for JavaScript

SendBird Desk is a chat customer service platform built on SendBird SDK and API.

Desk JavaScript SDK provides customer-side integration on your own application, so you can easily implement **ticketing system with chat inquiry**. Desk JavaScript SDK requires **SendBird JavaScript SDK 3.0.55 or higher**.

## Table of Contents

- [SendBird Desk SDK Integration Guide for JavaScript](#sendbird-desk-sdk-integration-guide-for-javascript)
  - [Table of Contents](#table-of-contents)
  - [Installation](#installation)
  - [Authentication](#authentication)
  - [Setting customer customFields](#setting-customer-customfields)
  - [Creating a new ticket](#creating-a-new-ticket)
  - [Count of opened tickets](#count-of-opened-tickets)
  - [Loading ticket list](#loading-ticket-list)
  - [Handling ticket event](#handling-ticket-event)
  - [Rich messages](#rich-messages)
  - [Confirm end of chat](#confirm-end-of-chat)
  - [Reopen a closed ticket](#reopen-a-closed-ticket)
  - [Ticket Feedback](#ticket-feedback)
  - [Related channels](#related-channels)
  - [URL preview](#url-preview)

## Installation

Before setup, you need to get SendBird App ID at [SendBird Dashboard](https://dashboard.sendbird.com). For using Desk solution, you may upgrade the plan. For more information, please contact [sales team](https://sendbird.com/contact-sales).

Installing the Desk SDK is straightforward if you're familiar with npm. Run the following command at the project path.

```
~$ cd /path/to/your-project
~$ npm install --save sendbird-desk
```

> You can use SendBird SDK along with SendBird Desk SDK (e.g. you can build your own messaging service while you use Desk SDK as well) In that case, you must use **the same APP_ID** for both SDKs. And please note that SendBird SDK version must be 3.0.55 or higher to work with Desk SDK.

> Basically SendBird Desk SDK is a plugin to handle tickets. You have to handle chat messages through SendBird SDK. Ticket is newly introduced in SendBird Desk SDK to support customer service ticketing system. Every ticket will be assigned to the appropriate agents and it will be mapped to a SendBird SDK channel, so you can implement real-time messaging on tickets with SendBird SDK.

## Authentication

Connecting to SendBird platform via SendBird SDK is essential for real-time messaging. The official guide to SendBird SDK authentication is [here](https://docs.sendbird.com/android#authentication_2_authentication).
Authentication in SendBird Desk is done by calling `SendBirdDesk.authenticate()`. The following is an example for SendBird SDK connection and SendBird Desk SDK authentication.

```js
const sb = new SendBird({ appId: "YOUR_APP_ID" });
sb.connect(userId, accessToken, (res, err) => {
  if (err) throw err;
  SendBirdDesk.init(SendBird);
  SendBirdDesk.authenticate(userId, accessToken, (res, err) => {
    if (err) throw err;
    // Now you can use Desk SDK later on
  });
});
```

Now your customers are ready to create tickets and start inquiry with your agents!

## Setting customer customFields

Customer information could be kept in `customFields`. `setCustomerCustomFields()` in `SendBirdDesk` lets the SDK set the `customFields` of the current customer. The `customFields` columns should be defined in SendBird Dashboard beforehand. Otherwise, the setting would be ignored.

```js
SendBirdDesk.setCustomerCustomFields(
  {
    gender: "male",
    age: 20,
  },
  (err) => {
    if (!err) {
      // customer's customFields is rightly set
      // (or a certain key could get ignored if the key is not defined yet)
    }
  }
);
```

## Creating a new ticket

Creating a new ticket is as simple as calling `Ticket.create()`. Once you create the ticket, you can access to created ticket and channel in callback. Channel object can be found at `ticket.channel` so that you can send messages to the channel. For more information in sending messages to channel, see [SendBird SDK guide docs](https://docs.sendbird.com/android#group_channel_3_sending_messages).

> Note: Notice that the ticket could be assigned to the agents only when customer sends at least one message to the ticket. Otherwise, agent cannot see the ticket.

```js
Ticket.create(ticketTitle, userName, (ticket, err) =>
    // You can send messages to the ticket channel using SendBird SDK
    // ticket.channel is the channel object
});
```

`Ticket.create()` has 2 more parameters `groupKey` and `customFields`. The values could be evaluated when a ticket is created though it's used only in Dashboard currently. `groupKey` is the key of an agent group so that the ticket is assigned to the agents in that group. `customFields` holds customizable data for the individual ticket. The below is an example.

```js
Ticket.create(
  ticketTitle,
  userName,
  "cs-team-1", // groupKey
  {
    text: "hello",
    number: 14,
    select: "option2",
  }, // customFields
  (ticket, err) => {
    // Ticket is created with groupKey 'cs-team-1' and customFields.
  }
);
```

> Note: each key in `customFields` should be preregistered in Dashboard. Otherwise, the key would be ignored.

## Count of opened tickets

When you need to display opened ticket count in your application, use `Ticket.getOpenCount()`.

```js
Ticket.getOpenCount((res, err) => {
  const count = res;
  // do something with the value
});
```

## Loading ticket list

Retrieving ticket list is essential for inbox. SendBird Desk SDK provides `Ticket.getOpenedList()` and `Ticket.getClosedList()` to get the list of open/closed ticket. Open ticket list and closed ticket list can be loaded as below:

```js
Ticket.getOpenedList(offset, (res, err) => {
  const tickets = res;
  // offset += tickets.length; for the next tickets.
  // here is to display tickets on inbox.
});
```

```js
Ticket.getClosedList(offset, (res, err) => {
  const tickets = res;
  // offset += tickets.length; for the next tickets.
  // here is to display tickets on inbox.
});
```

> Note: Once you set `customFields` to tickets, you can put `customFieldFilter` to `getOpenedList()` and `getClosedList()` in order to filter the tickets by `customFields` values.

## Handling ticket event

SendBird Desk SDK uses predefined AdminMessage custom type which can be derived by calling `message.customType`. Custom type for Desk AdminMessage is set to `SENDBIRD_DESK_ADMIN_MESSAGE_CUSTOM_TYPE`. And there are sub-types which indicate ticket events: assign, transfer, and close. Each event type is located in `message.data` which looks like below.

```js
{
    "type": "TICKET_ASSIGN" // "TICKET_TRANSFER", "TICKET_CLOSE"
}
```

You can check out these messages in `ChannelHandler.onMessageReceived()`.

## Rich messages

Rich message is a special message that holds custom content. You can distinguish the rich message by checking `message.customType` which is set to `SENDBIRD_DESK_RICH_MESSAGE`. Currently Desk SDK provides the following types of rich message.

- **`Confirm end of chat`** message notifies that the agent sent a confirm-end-of-chat request. When the user agrees with the closure, the ticket would be closed.
- **`URL preview`** message contains web URL which is filled with image and description.

## Confirm end of chat

Confirm end of chat message is initiated from the agent to inquire closure of ticket. The message has 3 states which are `WAITING`, `CONFIRMED`, `DECLINED`. When agent sends confirm end of chat message, its state is set to `WAITING`. Customer can answer to the inquiry as `YES` or `NO` which leads to `CONFIRMED` state and `DECLINED` state accordingly. You can check the state by looking `message.data`. The format looks like below:

```js
{
    "type": "SENDBIRD_DESK_INQUIRE_TICKET_CLOSURE",
    "body": {
        "state": "WAITING" // also can have "CONFIRMED", "DECLINED"
    }
}
```

Once customer replies to the inquiry, the message would be updated. `SendBirdDesk.Ticket.confirmEndOfChat(message, yesOrNo, callback)` sends request to update the message. You can catch the change in `channelHandler.onMessageUpdate(channel, message)`.

```js
channelHandler.onMessageUpdated = (channel, message) => {
  SendBirdDesk.Ticket.getByChannelUrl(channel.url, (ticket, err) => {
    if (err) throw err;
    let data = JSON.parse(message.data);
    const isClosureInquired = data.type === SendBirdDesk.Message.DataType.TICKET_INQUIRE_CLOSURE;
    if (isClosureInquired) {
      const closureInquiry = data.body;
      switch (closureInquiry.state) {
        case SendBirdDesk.Message.ClosureState.WAITING:
          // do something on WAITING
          break;
        case SendBirdDesk.Message.ClosureState.CONFIRMED:
          // do something on CONFIRMED
          break;
        case SendBirdDesk.Message.ClosureState.DECLINED:
          // do something on DECLINED
          break;
      }
    }
  });
};
```

## Reopen a closed ticket

The closed ticket could be reopened with `reopen()` function in `Ticket`.

```js
closedTicket.reopen((openTicket, err) => {
  // update the ticket list here
});
```

## Ticket Feedback

If Desk satisfaction feature is on, a message would come after closing the ticket. The message is for getting customer feedback including score and comment. The data of satisfaction form message looks like below.

```js
{
    "type": "SENDBIRD_DESK_CUSTOMER_SATISFACTION",
    "body": {
        "state": "WAITING" // also can have "CONFIRMED",
        "customerSatisfactionScore": null, // or a number ranged in [1, 5]
        "customerSatisfactionComment": null // or a string (optional)
    }
}
```

Once the customer inputs the score and the comment, the data could be submitted by calling `SendBirdDesk.Ticket.submitFeedback(message, score, comment, callback)`. Then updated message is going to be sent in `channelHandler.onMessageUpdate(channel, message)`.

```js
channelHandler.onMessageUpdated = (channel, message) => {
  SendBirdDesk.Ticket.getByChannelUrl(channel.url, (ticket, err) => {
    if (err) throw err;
    let data = JSON.parse(message.data);
    const isFeedbackMessage = data.type === SendBirdDesk.Message.DataType.TICKET_FEEDBACK;
    if (isFeedbackMessage) {
      const feedback = data.body;
      switch (feedback.state) {
        case SendBirdDesk.Message.FeedbacState.WAITING:
          // do something on WAITING
          break;
        case SendBirdDesk.Message.FeedbacState.CONFIRMED:
          // do something on CONFIRMED
          break;
      }
    }
  });
};
```

## Related channels

You can link certain channels to a ticket. Then the ticket would have `relatedChannels` property which contains the `channelUrl` and its `name`. In order to relate the channels to a ticket, use `ticket.setRelatedChannelUrls()` for the update, or set the `relatedChannelUrls` parameter at the time of ticket creation.

```js
Ticket.create(title, name, groupKey, customFields, priority, relatedChannelUrls, (ticket, err) => {
  // the created ticket will have `relatedChannels` property with the channel information that you provide.
});
```

```js
ticket.setRelatedChannelUrls(relatedChannelUrls, (ticket, err) => {
  // the `relatedChannels` property has been updated.
});
```

## URL preview

To send URL preview message, you should send a text message with URL, extract preview data, and update it with the preview data. Use `channel.updateUserMessage(messageId, text, messageData, customType, callback)` for the update operation. The format of `messageData` looks like below:

```js
/// give stringified JSON object to channel.updateUserMessage()
{
    "type": "SENDBIRD_DESK_URL_PREVIEW",
    "body": {
        "url": "string",
        "site_name": "string",
        "title": "string",
        "description": "string",
        "image": "string (image url)"
    }
}
```

You may get the preview message in `ChannelHandler.onMessageUpdated()` or `channel.getPreviousMessagesByTimestamp()` for URL preview rendering.
