# new API and its internal logic

JustinChen@TVU



## Change logs:

1. 20210915 the first version
2. 20210922 add a new API `mutation updateSubscription`;  
   modify `query permission` and add the recommended redirect URL
3. 20211018 rename `mutation checkoutNewSubscription` to `mutation checkoutSubscription`;  
   merge `mutation updateSubscription` into `mutation checkoutSubscription`;  
   `mutation checkoutSubscription` supports prorating credits and charges during upgrade subscription;  
   enrich `query permission`;  
   add `group` in `mutation checkoutSubscription` to support creating subscription for `group`;  
   add `mutation cancelSubscription`; 
4. 20211019 for the requirement: no Chargebee's checkout portal in our workflow
   add `query customerPaymentPortal`;  
   delete `query customerPortalStatus`;  
   update `mutation checkoutSubscription`
5. 20211020 change the return value of `query permission`

## Guideline 

[Concept](https://docs.google.com/document/d/1kbozAs6vIr7dGajmTbhbqR4O2u3WCQUzsZfP4NBSUnE/edit)

[Data Structure](https://docs.google.com/document/d/108J1mX22vblwRx7OgXGxaNQHYypfyKKP7kBFjGzusg0/edit)

[Slack](https://join.slack.com/share/zt-w9dqrpc4-BvDkRy4pzXumeIiDmWf_4A)

[Miro](https://miro.com/app/board/o9J_lbF7xGI=/?moveToWidget=3074457363982308077&cot=14)

[Engineering Document](https://docs.google.com/document/d/1LJjxVGk3X9yfuTIjHIq8q6FSnxWk298Fo4CQmwLiFRQ/edit#)

## Requirement

The new Billing Service should implement those features:

1. just rely on Chargebee to do two things:  (1) maintain payment information  (2) do transaction 
2. give up using the tools(plan/sub/addon) provided by Chargebee. Those logic should be implemented by ourselves.
3. Chargebee customer account should be saved in UserService's Group object. User has no its own CB customer account.

## Notes:

- router: `api/billing/v1/`

## Billing Service API

### query permission

> used by APP & HouseKeeper

For Partyline, its `partyline-hour` saves the static maximum credit. This user's real usage is obtained from Usage Service, which calculates from the moment that subscription was activated or updated to the moment when API is invoked. 
If all `permission.allow`=`true` , APP is allowed to proceed.
If the API returns the user has an unpaid invoice, App should guide user to open customer portal to maintain their payment information. `query customerPortal` can get the `customer portal`.

```
query{
  permission (
        customerId: "justinchen@tvunetworks.com",
        product: "Partyline",				// Partyline user just has one plan at a time. So no plan argument here.
        chargeItems: "string in JSON",		// sample listed below 		
        									{}  TVUChannel has no chargeItems. so it is empty here.
        									
        									OR:
                                            {
                                                "version":1,
                                                "partyline-participant":8,
                                                "partyline-output":4
                                            }  
                                            
                                            OR:
                                            {
                                            	"version":2,
                                            	"chargeItems": [
                                                {
                                                "id": "partyline-participant",
                                                "quantity": 8,
                                                },
                                                {
                                                "id": "partyline-output",
                                                "quantity": 4,
                                                }
                                            	]
                                            }
    ) {
    	encryptedStr
    	code 				// error code or 0x0(represents allow)
        description 		// readable description, such as "need to upgrade plan, or there is an unpaid invoice"
    	subscriptionId		// related subscription     			
	    remainHours			// The quantity of chargeItems - usage 
		permission {		// []
			id				// partyline-participant
			ceiling			// 8
			allow			// true/false
		},
		
		redirect {
			iframeSrc			//"the URL of the default plan list, or a self-service portal"
			iframeWidth
			iframeHeight
		}
    }
}
```

Usage Service API: 
https://showdoc.tvunetworks.com/web/#/111?page_id=3313
https://usageservice.tvunetworks.com/usageservice-backend/graphql

```
query{
    partylineJoinTimeSummary(
        startTime: "0",
        endTime: "1631571056",
        email: "justinchen@tvunetworks.com"
    )
}
```


The backend DB of Usage Service: https://usageservice.tvunetworks.com/#/infoView

### query customer

> used by embedded web

Retrieve a customer's information of payment method and those exceptional invoices. All exceptional invoices and its content can be assembled by this API.

```
query{
  customer(customerId:"justinchen@tvunetworks.com",
           product: "TVUSearch"
           ) {
    cardStatus              # valid / expiring / expired / no card
    unbilledCharges         # Total unbilled charges for this customer. in cents, min=0
    exceptionalInvoices     # TODO: [], status == payment_due / not_paid
    {
        id
        customer_id
        status              # https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status
        total
        due_date
        ...
    }    
  }
}
```

`card_status`  == `valid` means the customer filed payment way.

ref: [https://apidocs.chargebee.com/docs/api/customers#retrieve_a_customer](https://apidocs.chargebee.com/docs/api/customers#retrieve_a_customer)

### query customerPaymentPortal

The end-user can use the self-service portal to manage their payment method. 

```
query{
  customerPortal (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        redirectUrl: "https://search.tvunetworks.com"	# optional. URL to redirect when the user logs out from the portal.
    ) {
        id          # "portal_AzZlxDSWcJOT1FvC",
        token       # aRNcIRmnenKuV67D07JlT8tC07caVpPw",
        access_url  # https://tvunetworks-test.chargebee.com/pages/v3/6gAvxPV9cMFNcZVk8gehmThfW1mvJflX/
    }
}
```

### query customerPortal

The end-user can use the self-service portal to maintain their billing information / invoice / subscription. 

```
query{
  customerPortal (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        redirectUrl: "https://search.tvunetworks.com"	# optional. URL to redirect when the user logs out from the portal.
    ) {
        id          # "portal_AzZlxDSWcJOT1FvC",
        token       # aRNcIRmnenKuV67D07JlT8tC07caVpPw",
        access_url  # https://tvunetworks-test.chargebee.com/portal/v2/authenticate?token=KzU6TF4tcd8laBs1RCzz2z8FTeamcubBu6
    }
}
```

### query plans

> used by embedded web

fetch the existing plans and their details for a given product.

```
query{
  plans(
        product: "TVUSearch"
        planType: "all"  // "all"/"common"/"custom", default is "all";
    ) {
        "number": 2,
        "planIds":[ "mm_fednet5_subppu_199", "mm_fednet_special_knbc_0" ]
    }
}
```

### query subscription

> used by embedded web

return the information of customer's subscription. 

`paywalld` ensures that a user just has one active `subscription` for one `plan` in TVUSearch; a user just has one active `subscription` in `Partyline`.

```
query {
    subscription (
        customerId: "justinchen@tvunetworks.com",
        product: "Partyline",
        planId: "partyline-common-custom",	# In `Partyline`, this option will be optional 
        status: 1
    ) {
         subscriptionId 		// subscription ID
         status		// active / future/in_trial/non_renewing/paused/cancelled
    	 deleted	// true / false,
         ...
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#list_subscriptions The explanation of `status`: https://apidocs.chargebee.com/docs/api/subscriptions#subscription_status

### mutation changeSubscriptionStatus

```
query {
    cancelSubscription (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "MM_Fednet_Sub_98",		
        subscriptionId: "id"，
        status: "cancel"					// active/non_renewing/paused/cancelled
    ) {
         subscriptionId 		// subscription ID
         status					// active/non_renewing/paused/cancelled
         ...
	}
}
```

### query previewCustomSubscription

calculate the price of the custom subscription. 

```
query {
    previewCustomSubscription (
        customerId: "justinchen@tvunetworks.com",	# reserved
		product: "Partyline",						# case insensitive
    	planId: "partyline-common-custom",
    	chargeItems: "string in JSON",				# sample listed below 				
                                            {
                                                "version":1,
                                                "partyline-hour":10,
                                                "partyline-participant":8,
                                                "partyline-output":4
                                            }  
                                            
                                            OR:
                                            {
                                            	"version":2,
                                            	"chargeItems": [
                                                {
                                                "id": "partyline-hour",
                                                "quantity": 10,
                                                },
                                                {
                                                "id": "partyline-participant",
                                                "quantity": 8,
                                                },
                                                {
                                                "id": "partyline-output",
                                                "quantity": 4,
                                                }
                                            	]
                                            }

    ) {
      	price	# $100
    }
}
```

`formula` presents in JSON and is saved in Billing Service.

```json
{"formula": "$partyline-hour*0.25*(2*$partyline-participant*(1+225/100)+$partyline-output*(1+225/100))"}
```

### mutation checkoutSubscription

> used by embedded web

providing a hosted page for customers to finish checkout/create/update subscriptions for given valid plan ID.  Return `success` when the customer has an active subscription of the plan already.

```
mutation {
    checkoutSubscription (
    	groupId: "In new UserService, this `group` will be a new concept and is not the same as `root group` or `sub group`"
        customerId: "justinchen@tvunetworks.com",
		product: "Partyline",
    	planId: "partyline-common-advance",
    	chargeItems: "same JSON as query previewCustomSubscription",	# optional. It's meaningful while `plan` is `custom`
    ) {
        invoice{
        	id: "xxxx",
        	status: "NOT_PAID",			//PAID/POSTED/PAYMENT_DUE/NOT_PAID/VOIDED/PENDING
        	statusDescription: "the payment is not made and all attempts to collect is failed",
        	amountPaid: 0
        	total: 84
        }
    }
}
```

Internal logic:

1. need to create a Chargebee customer account if no account of this user's group exists. And pass its customer account ID to User Service;
2. checkout one time invoice(https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create_an_invoice);
3. save the string of chargeItems to the database table `billing_record` to log the event
4. Chargebee will pass the invoice status to `mutation eventNotify` via web hook. The relationship between `subscriptionID` and `invalid_invoice_id` will be created there and saved in table `billing_record`.
5. Based on the record saved in table `billing_record` , insert a new record to `subscription` table and set the status of subscription to `valid` , and table `subaddon` should be updated too. 
6. Since this introduces a change in the price of the subscription, **prorated credits and charges** can be raised to ensure proper billing. The accountable duration is a **day**. 
7. no refund for downgrade yet.

ref:

Invoice status: https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1&lang=java#invoice_status

~~Internal logic:~~

1. ~~need to create a Chargebee customer account if no account of this user's group exists. And pass its customer account ID to User Service;~~
2. ~~checkout OneTime Invoice(https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutOneTime-usecases), return a hosted page;~~
3. ~~save hostedPageID & the string of chargeItems to the database table `billing_record` to log the event~~
4. ~~our embedded web uses `query customerPortalStatus` to get the payment status. The embedded web should inform App of the status.~~ 
5. ~~At the same time, the backend should wait for the callback from Chargebee. We can check the payment status in its handler.~~
6. ~~Chargebee will pass the invoice status to `mutation eventNotify` via web hook.~~
7. ~~The payment status and invoiceID can be obtained from the previous three steps.  They have the same logic. The relationship between `subscriptionID` and `invalid_invoice_id` will be created there and saved in table `billing_record`.~~
8. ~~Based on the record saved in table `billing_record` , insert a new record to `subscription` table and set the status of subscription to `valid` , and table `subaddon` should be updated too.~~ 
9.  ~~Since this introduces a change in the price of the subscription, **prorated credits and charges** can be raised to ensure proper billing. The accountable duration is a **day**.~~ 
10. ~~no refund and downgrading yet.~~

#### `cron` task

cron tasks are created to scan the valid subscription and abnormal invoices each day and generate the financial statement.

### mutation cancelSubscription

> used by embedded web

Normally, one `plan` just has one `subscription` for one user. Return the same structure as `query subscription`'s.

```
query {
    cancelSubscription (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "MM_Fednet_Sub_98"
    ) {
         subscriptionId 		// subscription ID
         status		// active / future/in_trial/non_renewing/paused/cancelled
    	 deleted	// true / false,
         ...
	}
}
```

ref:  [https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#cancel_a_subscription](

### mutation eventNotify

> used by CB's webhook

use it to get the status of the invoice. The format is defined by Chargebee.

## HouseKeeper API

1. an internal Timer. It is created when an event is created in APP; It is destroyed when an event is stopped in APP. And notify App when credit runs out or is about to run out via `callback_URL` listed below.
2. API: `startEvent` { session, user， `a new encrypted string got from query permission` ,   `callback_URL`}
   `callback_URL`:  hosted by APP.
3. API: `stopEvent` { session, user,  `a new encrypted string got from query permission` }