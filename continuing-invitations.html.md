---
title: Continuing Invitation Pattern
layout: default
---

### Notes on Agoric Continuing invitation Pattern.

### Continuing Invitation pattern

**Brief introduction to invitation patterns**

To interact with an Agoric smart contract, an actor needs an invitation. With this invitation, they are assigned a [seat](https://docs.agoric.com/glossary/#seat) at the table to make an [offer](https://docs.agoric.com/glossary/#offer) for a contract. This process is facilitated by [Zoe](https://docs.agoric.com/guides/zoe/offer-safety)

Offers are routed to the contract via Agoric Smart Wallet. Agoric provides an interface to the smart wallet via [Agoric Ui Kit](https://docs.agoric.com/guides/UIComponentLibrary/#connect-wallet)

Here's a template for making an offer to a contract using the UI Kit library.

```javascript
wallet.makeOffer(
  invitationSpec,
  proposal,
  offerArgs, //optional
  callbackFunction,
  offer-id,
);
```

<blockquote>
  
  TODO create Diagram
  
  1. Actor gets an invitation they redeem the invitation to get a
  
  2. seat. They then use this seat to make an
  
  3. offer

</blockquote>
---

**InvitationSpec Usage**

From [specifying offers documentation](https://docs.agoric.com/guides/getting-started/contract-rpc.html#specifying-offers)

<blockquote>
    
    InvitationSpec Usage
    Supposing ```spec``` is an InvitationSpec, its .source is one of:

    - purse - to make an offer with an invitation that is already in the Invitation purse of the smart wallet and agrees with spec on .instance and .description properties. For example, in dapp-econ-gov, committee members use invitations sent to them when the committee was created.

    - contract - the smart wallet makes an invitation by calling a method on the public facet of a specified instance: E(E(zoe).getPublicFacet(spec.instance)[spec.publicInvitationMaker](...spec.invitationArgs)

    - agoricContract

    - continuing

</blockquote>

Our focus is on the `source` property. For this tutorial, we're exploring the `continuing` variant.
First, however, let's consider the `contract` variant, which will provide a foundation for exploring the `continuing` invitation in detail.

A basic contract might take the following form:

```javascript
//simpleTrade.contract.js
const start = (zcf) => {
  const publicFacet = Far('Public facet', {
    singleTradeInvitation: () =>
      zcf.makeInvitation(
        // invitation Handler
        (seat, offerArgs) => {
          const { give, want } = seat.getProposal();
          //reallocation happens here
          //zoe makes sure that you will get what you want if you give
          //what you claimed
          seat.exit(); //you exit the seat after reallocaton.
          return `trade complete. assets reallocated`;
        },
        'Single trade invitation',
      ),
  });
};
```

We can use the UI Kit provided by Agoric to send offers to the smart contract. First, we create an `InvitationSpec` and `proposal` as follows:

```javascript
const initialInvitationSpec = {
  source: 'contract',
  instance:
    simpleTrade /* Typically, you'll use vstorage to fetch the contract instance*/,
  publicInvitationMaker: singleTradeInvitation,
};

const proposal = {
  want: harden({
    /** some amount  wanted */
  }), // AssetA
  give: harden({
    /** some amount to give in return */
  }), // AssetB
};

let offer-id = offer-123;
```

We then use the wallet connection object to send our offer:

```javascript
wallet.makeOffer(
  initialInvitationSpec,
  proposal,
  undefined, //no offerArgs specified
  () => {},
  offer-id,
);
```

For more details see [makeOffer](https://docs.agoric.com/guides/getting-started/ui-tutorial/making-an-offer.html#submitting-the-offer) documentation.

---

**_Actor Interactions_**

Let's consider an actor named Alice interacting with our `simpleTrade` contract. To do this, Alice specifies `InvitationSpec` with `source,` `instance,` and `publicInvitationMaker,`. The `publicInvitationMaker` in this case is the name of the remote method from which she can redeem the invitation. In her proposal, she specifies the value of the payment she offers to give and the value of the payment she wants in return.

See more on [Zoe Offer Safety](https://docs.agoric.com/guides/zoe/offer-safety.html#offer-safety)
More detailed documentation on Zoe can be found here [zoe](https://docs.agoric.com/reference/zoe-api/)

Notice that in our `simpleTrade` contract, Alice interacts with the contract in a single sequence.

### Continuing Invitation Pattern

**Example: Event Tickets Minting Contract**

Let's imagine an event tickets minting contract. In our hypothetical contract, attendees MUST obtain an entry ticket before accessing other services at the event, such as buying drinks or upgrading their seat.

Here's how we could model our contract:
Alice intends to attend the event. First, she must secure an entry ticket. O ly after she obtains this ticket can she access other services, including any upgraded.
To purchase the entry ticket, `Alice` makes an offer via the publicly accessible `entryInvitation`. I her offer, she'll specify a proposal with the amount needed for the entry ticket. [dapp interface](https://docs.agoric.com/guides/getting-started/ui-tutorial/making-an-offer.html#submitting-the-offer).
In her proposal, `Alice` will specify `{want, give}` properties, where `give` is the amount(for the entry ticket), and `want` is the ticket payment the contract should mint at the end of this interaction. In our case, it's just the entry ticket.
Since she needs the entry ticket to access other services, our contract will also return an invitation, from which Alice can make future offers to the contract, like buying drinks or upgrading to a VIP section, as per the contract specification.

Here's a simplified look at the ticketing contract:

```javascript
  //ticketing.contract.js
  const start = zcf => {

    const publicFacet = Far("Ticketing publicFacet", {
        entryInvitation: zcf.makeInvitation(
          //invitation Handler
          (seat, offerArgs) => {
            const {give, want} =  seat.getProposal();
            // some reallocation

            seat.exit(); // they first need to exit the seat we
            return {InvitationMakers: /** A remote object that has some invitation */}
          },
          'Initial event entry invitation'
          ),
    })
  }
```

Notice that this time, our offer handler did not return a string; instead, it returns an object with the `InvitationMakers` property. The UI Kit requires this for continuing invitations. It specifically looks for that object in the return value from the offer handler.

Now, let's implement our `InvitationMakers` object.

```javascript
const upSellInvitationMaker = Far('Event Tickets Upsell Invitation', {
  buyDrinksInvitation: () =>
    zcf.makeInvitation(
      // Invitation handler
      (seat, offerArgs) => {
        const { give, want } = seat.getProposal();
        //Some processing if needed
        //Reallocate drinks if the actor gives the payment for them

        seat.exit();
        return `trade complete. d ink(s) sold`;
      },
      'Buy drinks invitation',
    ),

  upgradeSeatInvitation: () =>
    zcf.makeInvitation(
      //invitation Handler
      (seat, offerArgs) => {
        //validation and reallocation as before

        seat.exit();
        return `trade complete. Ticket upgraded`;
      },
      'Upgrade ticket invitation',
    ),
  /*
        Other invitations 
      */
});
```

`Alice` can now use `entryInvitation` to buy drinks, upgradeSeat, and any other invitation from our `upsellInvitation`.

In summary, in the continuing invitation pattern, the result of the offer from one invitation is an invitation to make another offer.

Now, putting it all together:

```javascript
//ticketing.contract.js
const start = (zcf) => {
  const upSellInvitationMaker = Far('Tickets Upsell Invitation', {
    buyDrinksInvitation: () =>
      zcf.makeInvitation((seat, offerArgs) => {
        const { give, want } = seat.getProposal();
        // some processing, if needed
        // reAllocation. buyer gets their drink

        seat.exit();
        return `trade complete. drink(s) sold`;
      }, 'Buy drinks invitation'),
    upgradeSeatInvitation: () =>
      zcf.makeInvitation((seat, offerArgs) => {
        //validation and reallocation

        seat.exit();
        return `trade complete. Ticket upgraded`;
      }, 'Upgrade seat invitation'),
    /*
          Other invitations 
        */
  });

  const publicFacet = Far('Ticketing publicFacet', {
    entryInvitation: zcf.makeInvitation((seat, offerArgs) => {
      const { give, want } = seat.getProposal();
      // validation and reallocation

      seat.exit(); // they first need to exit the seat we
      return { InvitationMakers: upSellInvitationMaker };
    }, 'Initial event entry invitation'),
  });
};
```

---

**Frontend Code**

Lets see how we can implement interaction from the frontend

Step 1. Redeem `entryInvitation` and make an offer to purchase the entry ticket

```javascript
const proposal = {
  give: harden({
    /* purchase amount */
  }), // supposedly, USDT
  want: harden({
    /* want amount */
  }), // supposedly, ETicket
};

//InvitationSpec
const entryInvitationSpec = {
  source: 'contract',
  instance: ticketingInstance,
  publicInvitationMaker: 'entryInvitation',
};
// making the offer
wallet.makeOffer(
  entryInvitationSpec,
  proposal,
  undefined,
  (/* callBack function to catch updates*/) => {},
  offer-id, // eg offer-124
);
```

Alice now has an entry `Eticket.` She also has a `continuing invitation` to enable her to access upsell services in the future.

Step 2. Redeem upsell services

Suppose she wants to purchase a drink. Alice redeems a `buyDrinksInvitation` and uses it to make an offer to buy a drink

```javascript
const proposal = {
  /* give, want as per buyDrinks terms */
};

const buyDrinksInvitationSpec = {
  source: 'continuing',
  previousOffer: 'offer-124', //the offer Id we used in our entry ticket offer
  invitationMakerName: 'buyDrinksInvitation', //
};
```

Then call makeOffer as before, with a new `offerId` for this offer.

Repeat the same process for `upgradeSeatInvitation`.


Continuing Invitation Use cases:

More resources:

- [Office Hours Discussion on continuing invitation pattern](https://github.com/Agoric/agoric-sdk/discussions/9648)
- [more complete implementaion of this pattern](https://github.com/Agoric/agoric-sdk/blob/7a3969fe649a85b404dc971732759c7357dbfc50/packages/orchestration/src/examples/staking-combinations.contract.js)
- [Official documentation](https://docs.agoric.com/glossary/#continuing-invitation-pattern)
- [source code](https://github.com/Agoric/agoric-sdk/blob/7a3969fe649a85b404dc971732759c7357dbfc50/packages/smart-wallet/src/invitations.js#L75)
- [Continuing invitation spec](https://github.com/Agoric/agoric-sdk/blob/7a3969fe649a85b404dc971732759c7357dbfc50/packages/smart-wallet/src/invitations.js#L146)
- [https://github.com/Agoric/agoric-sdk/discussions/10819](https://github.com/Agoric/agoric-sdk/discussions/10819)
-
