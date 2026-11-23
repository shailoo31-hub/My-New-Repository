# LWC Component Communication in Salesforce: Bubbling, Pub-Sub and Lightning Message Service (LMS)

## 1. Introduction

In Salesforce Lightning Web Components (LWC), multiple components often
need to communicate with each other.

Examples: - A child component sends selected Account information to its
parent. - A parent passes an Account ID to its child. - Two independent
components on the same Salesforce record page share information.

The main communication approaches are:

1.  **Parent to Child:** `@api`
2.  **Child to Parent:** Custom Events
3.  **Event propagation:** Bubbling
4.  **Unrelated components:** Pub-Sub (legacy approach) or Lightning
    Message Service (LMS)

------------------------------------------------------------------------

## 2. Parent-to-Child Communication Using `@api`

Suppose we have: - Parent component: `parentComponent` - Child
component: `childComponent`

The parent passes an Account name to the child.

### Child JavaScript --- `childComponent.js`

``` javascript
import { LightningElement, api } from 'lwc';

export default class ChildComponent extends LightningElement {
    @api accountName;
}
```

The `@api` decorator makes `accountName` a public property that the
parent can set.

### Child HTML --- `childComponent.html`

``` html
<template>
    <h2>Child Component</h2>
    <p>Account Name: {accountName}</p>
</template>
```

### Parent JavaScript --- `parentComponent.js`

``` javascript
import { LightningElement } from 'lwc';

export default class ParentComponent extends LightningElement {
    accountName = 'Salesforce';
}
```

### Parent HTML --- `parentComponent.html`

``` html
<template>
    <h2>Parent Component</h2>

    <c-child-component
        account-name={accountName}>
    </c-child-component>
</template>
```

### Execution flow

1.  The parent has the `accountName` property.
2.  The parent passes it to the child through an HTML attribute.
3.  The child receives it through its `@api` property.
4.  The child displays the value.

**Remember:** `@api` is commonly used for Parent-to-Child communication.

------------------------------------------------------------------------

## 3. Custom Events and Event Bubbling (Child-to-Parent)

Suppose a child component contains a button. When the user clicks it,
the child sends a message to its parent.

We use: - `CustomEvent()` to create the event. - `dispatchEvent()` to
fire the event. - `bubbles: true` to enable upward propagation. -
`composed: true` to allow the event to cross shadow DOM boundaries.

### Child JavaScript --- `childComponent.js`

``` javascript
import { LightningElement } from 'lwc';

export default class ChildComponent extends LightningElement {
    handleClick() {
        const event = new CustomEvent('message', {
            detail: 'Hello Parent!',
            bubbles: true,
            composed: true
        });

        this.dispatchEvent(event);
    }
}
```

### Understanding the code

  Code                          Responsibility
  ----------------------------- ---------------------------------------
  `new CustomEvent()`           Creates a custom event
  `detail`                      Carries the data
  `bubbles: true`               Enables upward event propagation
  `composed: true`              Allows crossing shadow DOM boundaries
  `this.dispatchEvent(event)`   Fires the event

**Important:** `bubbles` and `composed` are both `false` by default. An
event can still reach an immediate parent that directly listens on the
child component, even when bubbling is not enabled.

### Child HTML --- `childComponent.html`

``` html
<template>
    <lightning-button
        label="Send Message"
        onclick={handleClick}>
    </lightning-button>
</template>
```

### Parent JavaScript --- `parentComponent.js`

``` javascript
import { LightningElement } from 'lwc';

export default class ParentComponent extends LightningElement {
    receivedMessage;

    handleMessage(event) {
        this.receivedMessage = event.detail;
    }
}
```

### Parent HTML --- `parentComponent.html`

``` html
<template>
    <h2>Parent Component</h2>

    <c-child-component
        onmessage={handleMessage}>
    </c-child-component>

    <p>Received Message: {receivedMessage}</p>
</template>
```

### Execution flow

1.  The user clicks the button in the child.
2.  The child executes `handleClick()`.
3.  The child creates a custom event named `message`.
4.  The child fires it using `dispatchEvent()`.
5.  The parent receives the event through `onmessage`.
6.  The parent reads the data using `event.detail`.

**Output:** `Received Message: Hello Parent!`

**Remember:** Custom Events are used for Child-to-Parent communication.
Bubbling controls whether the event propagates upward through the DOM.

------------------------------------------------------------------------

## 4. Pub-Sub (Publish-Subscribe)

Now imagine two independent components: - Component A: Search
Component - Component B: Display Component

Neither component is the parent or child of the other. When Component A
selects a product, Component B should receive its name.

### What is Pub-Sub?

Pub-Sub stands for Publish-Subscribe: - **Publisher:** Publishes a
message. - **Subscriber:** Registers to receive a message.

For example, Component A publishes `Laptop`, and Component B receives
`Laptop`.

### Step 1: Create a Pub-Sub JavaScript module

Location:

``` text
force-app/main/default/lwc/pubsub/
```

File: `pubsub.js`

``` javascript
const events = {};

const registerListener = (eventName, callback) => {
    if (!events[eventName]) {
        events[eventName] = [];
    }

    events[eventName].push(callback);
};

const unregisterListener = (eventName, callback) => {
    if (!events[eventName]) {
        return;
    }

    events[eventName] = events[eventName].filter(
        listener => listener !== callback
    );
};

const fireEvent = (eventName, payload) => {
    if (!events[eventName]) {
        return;
    }

    events[eventName].forEach(callback => {
        callback(payload);
    });
};

export {
    registerListener,
    unregisterListener,
    fireEvent
};
```

This is a simplified educational Pub-Sub implementation. It maintains
listeners and distributes messages.

### Step 2: Publisher JavaScript

File: `searchComponent.js`

``` javascript
import { LightningElement } from 'lwc';
import { fireEvent } from 'c/pubsub';

export default class SearchComponent extends LightningElement {
    handleSearch() {
        const productName = 'Laptop';

        fireEvent('Product_Selected', productName);
    }
}
```

### Publisher HTML

File: `searchComponent.html`

``` html
<template>
    <lightning-button
        label="Select Product"
        onclick={handleSearch}>
    </lightning-button>
</template>
```

### Step 3: Subscriber JavaScript

File: `displayComponent.js`

``` javascript
import { LightningElement } from 'lwc';

import {
    registerListener,
    unregisterListener
} from 'c/pubsub';

export default class DisplayComponent extends LightningElement {
    productName;

    connectedCallback() {
        this.handleProductSelected =
            this.handleProductSelected.bind(this);

        registerListener(
            'Product_Selected',
            this.handleProductSelected
        );
    }

    handleProductSelected(data) {
        this.productName = data;
    }

    disconnectedCallback() {
        unregisterListener(
            'Product_Selected',
            this.handleProductSelected
        );
    }
}
```

### Subscriber HTML

File: `displayComponent.html`

``` html
<template>
    <h2>Display Component</h2>
    <p>Selected Product: {productName}</p>
</template>
```

### Execution flow

1.  Component A publishes the product name.
2.  The Pub-Sub module identifies registered listeners.
3.  Component B receives the message.
4.  Component B updates its property.
5.  Its HTML displays the selected product.

### Important Salesforce note

Pub-Sub is a legacy/community approach. Salesforce recommends Lightning
Message Service (LMS) for new development where LMS supports the
communication scenario.

The example above is a simplified teaching implementation, not
Salesforce's complete historical Pub-Sub utility. It assumes both
components share the same JavaScript module instance.

------------------------------------------------------------------------

## 5. Lightning Message Service (LMS)

Lightning Message Service is Salesforce's standard mechanism for
communication between independent components.

It supports communication between: - Independent LWC components. - LWC
and Aura components. - LWC and Visualforce pages, where supported.

### Scenario

We have two independent components on a Salesforce record page:

1.  Account Publisher Component
2.  Contact Subscriber Component

When the user clicks a button in the Account Publisher, the Account ID
is sent to the Contact Subscriber.

### Step 1: Create a Lightning Message Channel XML file

Folder:

``` text
force-app/main/default/messageChannels/
```

File:

`AccountMessageChannel.messageChannel-meta.xml`

``` xml
<?xml version="1.0" encoding="UTF-8"?>

<LightningMessageChannel
    xmlns="http://soap.sforce.com/2006/04/metadata">

    <masterLabel>Account Message Channel</masterLabel>

    <isExposed>true</isExposed>

    <lightningMessageFields>
        <fieldName>recordId</fieldName>
    </lightningMessageFields>

</LightningMessageChannel>
```

### Understanding the XML

  XML element                 Purpose
  --------------------------- ---------------------------------------
  `LightningMessageChannel`   Defines the message channel
  `masterLabel`               Display name of the channel
  `isExposed`                 Makes the channel available for use
  `lightningMessageFields`    Defines fields in the message payload
  `fieldName`                 Name of a field in the payload

**Important:** The XML file is not part of the Publisher or Subscriber
component. It is a separate metadata component shared by both.

You can create multiple message channels in this folder:

``` text
messageChannels/
    AccountMessageChannel.messageChannel-meta.xml
    ContactMessageChannel.messageChannel-meta.xml
    ProductMessageChannel.messageChannel-meta.xml
    OrderMessageChannel.messageChannel-meta.xml
```

Each XML file defines a separate channel. The filename determines the
API name used in the JavaScript import.

### Step 2: Publisher JavaScript

File: `accountPublisher.js`

``` javascript
import { LightningElement, wire } from 'lwc';

import {
    publish,
    MessageContext
} from 'lightning/messageService';

import ACCOUNT_CHANNEL
    from '@salesforce/messageChannel/AccountMessageChannel__c';

export default class AccountPublisher extends LightningElement {
    @wire(MessageContext)
    messageContext;

    handleClick() {
        const payload = {
            recordId: '001XXXXXXXXXXXXXXX'
        };

        publish(
            this.messageContext,
            ACCOUNT_CHANNEL,
            payload
        );
    }
}
```

The sample record ID is illustrative; use a real Account ID in an actual
org.

### Understanding the important parts

**MessageContext**

``` javascript
@wire(MessageContext)
messageContext;
```

Provides the context required by LMS to publish and subscribe.

**Import the channel**

``` javascript
import ACCOUNT_CHANNEL
    from '@salesforce/messageChannel/AccountMessageChannel__c';
```

Imports the XML-defined channel.

**publish()**

``` javascript
publish(
    this.messageContext,
    ACCOUNT_CHANNEL,
    payload
);
```

Publishes the message. Its arguments are the message context, message
channel, and payload.

### Publisher HTML

File: `accountPublisher.html`

``` html
<template>
    <lightning-button
        label="Select Account"
        onclick={handleClick}>
    </lightning-button>
</template>
```

### Step 3: Subscriber JavaScript

File: `contactSubscriber.js`

``` javascript
import { LightningElement, wire } from 'lwc';

import {
    subscribe,
    MessageContext
} from 'lightning/messageService';

import ACCOUNT_CHANNEL
    from '@salesforce/messageChannel/AccountMessageChannel__c';

export default class ContactSubscriber extends LightningElement {
    subscription = null;
    selectedAccountId;

    @wire(MessageContext)
    messageContext;

    connectedCallback() {
        this.subscribeToMessage();
    }

    subscribeToMessage() {
        if (!this.subscription) {
            this.subscription = subscribe(
                this.messageContext,
                ACCOUNT_CHANNEL,
                (message) => {
                    this.selectedAccountId = message.recordId;
                }
            );
        }
    }
}
```

**subscribe()** registers the component to receive messages on the
specified channel.

The callback runs when a message is received:

``` javascript
(message) => {
    this.selectedAccountId = message.recordId;
}
```

### Subscriber HTML

File: `contactSubscriber.html`

``` html
<template>
    <h2>Contact Subscriber</h2>

    <p>
        Selected Account ID:
        {selectedAccountId}
    </p>
</template>
```

### Execution flow

1.  The user clicks the button in Account Publisher.
2.  Publisher creates a JavaScript payload.
3.  Publisher calls `publish()`.
4.  Salesforce sends the message through the Lightning Message Channel.
5.  Subscriber receives the message.
6.  Subscriber executes its callback.
7.  Subscriber reads the Account ID using `message.recordId`.

**Output:** The Contact Subscriber displays the received Account ID.

For production implementations, consider subscription lifecycle,
cleanup, and whether the component needs an active or application-scoped
subscription.

### Is XML data transferred?

No. The XML defines the channel. The actual JavaScript payload is
transferred.

For example:

``` javascript
const payload = {
    recordId: '001XXXXXXXXXXXXXXX'
};

publish(
    this.messageContext,
    ACCOUNT_CHANNEL,
    payload
);
```

Here, `payload` is the data being sent.

------------------------------------------------------------------------

## 6. Multiple Publishers and Subscribers in LMS

Multiple Publisher and Subscriber components can use the same Lightning
Message Channel.

For example:

``` text
              AccountMessageChannel
                       |
          +------------+------------+
          |            |            |
          v            v            v
      Publisher 1  Publisher 2  Publisher 3
          |            |            |
          +------------+------------+
                       |
                       v
               Subscribers
```

You can also create separate channels for different business
requirements:

-   `AccountMessageChannel`
-   `ProductMessageChannel`
-   `OrderMessageChannel`

**Remember:** One XML file represents one Lightning Message Channel.

------------------------------------------------------------------------

## 7. Difference Between Bubbling, Pub-Sub and LMS

  -----------------------------------------------------------------------
  Feature           Custom Events /   Pub-Sub           LMS
                    Bubbling                            
  ----------------- ----------------- ----------------- -----------------
  Communication     Child to Parent   Unrelated         Unrelated
                                      components        components

  Mechanism         Custom Events     Publish /         Message Channel
                                      Subscribe         

  Parent-child      Normally yes      No                No
  relationship                                          

  Salesforce        Yes               Legacy approach   Yes
  standard                                              

  Recommended for   Yes               No                Yes
  new development                                       
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 8. When Should We Use What?

  Requirement                                    Recommended approach
  ---------------------------------------------- ----------------------
  Parent passes data to child                    `@api`
  Child sends data to parent                     Custom Event
  Event propagates through multiple DOM levels   Event Bubbling
  Independent LWCs communicate                   LMS
  LWC communicates with Aura                     LMS
  LWC communicates with Visualforce              LMS
  Existing legacy application uses Pub-Sub       Pub-Sub

------------------------------------------------------------------------

## 9. Important Salesforce Interview Questions

### Q1. What is the difference between @api and Custom Events?

-   `@api` is used for Parent-to-Child communication.
-   Custom Events are used for Child-to-Parent communication.

### Q2. What is dispatchEvent()?

It fires a custom event from a component.

### Q3. Does dispatchEvent() automatically enable bubbling?

No. Bubbling is disabled by default. Set `bubbles: true` to enable it.

### Q4. What is the purpose of composed: true?

It allows an event to cross shadow DOM boundaries.

### Q5. What is LMS?

Lightning Message Service is Salesforce's standard mechanism for
communication between independent components.

### Q6. Why do we create an XML file in LMS?

The XML file defines the Lightning Message Channel that Publisher and
Subscriber components share.

### Q7. Can multiple components use the same LMS channel?

Yes. Multiple Publishers and Subscribers can use the same channel.

### Q8. What is the difference between Pub-Sub and LMS?

Pub-Sub is a legacy/community publish-subscribe approach, whereas LMS is
Salesforce's standard message service.

### Q9. What is MessageContext?

It provides the context required by LMS to publish and subscribe to
messages.

### Q10. What is the purpose of publish() and subscribe()?

-   `publish()` sends a message.
-   `subscribe()` listens for messages.

------------------------------------------------------------------------

## 10. Final Summary

  Concept        Direction / Purpose      Main Function
  -------------- ------------------------ -----------------------------
  `@api`         Parent to Child          Public property
  Custom Event   Child to Parent          `dispatchEvent()`
  Bubbling       Upward DOM propagation   `bubbles: true`
  Pub-Sub        Independent components   Publish / Subscribe
  LMS            Independent components   `publish()` / `subscribe()`

### The easiest way to remember

``` text
Parent Component
       |
       | @api
       v
Child Component

Child Component
       |
       | Custom Event
       v
Parent Component

Independent Components
       |
       | Pub-Sub (Legacy)
       |
       | LMS (Recommended)
       v
Other Independent Components
```

### Final takeaway

-   `@api` → Parent to Child.
-   Custom Event → Child to Parent.
-   `bubbles: true` → Enables upward event propagation.
-   `composed: true` → Allows crossing shadow DOM boundaries.
-   Pub-Sub → Legacy approach for unrelated components.
-   LMS → Salesforce's standard approach for communication between
    independent components.

Learn `@api` and Custom Events first because they are the foundation of
LWC component communication. Then learn LMS for communication between
independent Salesforce components.
