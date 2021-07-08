# paywalld

JustinChen@TVU

------

Change logs:

1. **20210424** the first version that have been used in product of TVUSearch
2. **20210525** add a new API `mutation checkoutNewSubscriptionInvoice` for Partyline
3. **20210526** enhance the feature of `query plan` for Partyline
   added a new API `mutation cancelSubscription`
4. **20210527** added a new API `query customerPortalStatus`
5. **20210531**  `query subscription` and  `query plan` support checking `remaining credit`
6. **20210607** added a new API `mutation cancelInvoice`
   enhance the feature of `query customer` to support getting the unpaid invoices.

------

Some Terms:

* frontend --- the web page of the client of paywalld, such as the webpage of TVUSearch
* backend -- the web service of the client of paywalld, such as the web service backend of TVUSearch.
* customer -- who uses TVU service. such as `NY Times`.
* client/product/service  --- TVU service, such as Mediamind/Producer. It is represented by the constantly string, which saves in the custom field of `TVU Service Type`.
* operation/billing type --- a kind of operation that is not free. Such as Downloading marked file in TVUSearch. It is represented by the constantly string, which saves in the custom field of `Billing Type`. It's an attribute of `addon`.
* chargeItem --- applicable `addon`
* SUB -- subscription
* PPU -- pay per usage
* PK -- credit

Notes:

* The API of Chargebee have two big versions. V1 and V2.
  For V2, there are two small versions, prod_cat_ver=1 and prod_cat_ver=2.  we are using is `prod_cat_ver=1`, which was the stable version when we start developing paywalld.  https://apidocs.chargebee.com/docs/api/

* router: `api/paywall/v2/`

* If customer doesn't exist when client invokes paywalld's API, paywalld needs to create a new one.

### for PM/Solution:

If PPU is  $0, SUB+PPU+PK = SUB; If PK is null, SUB+PPU+PK = SUB+PPU
All operations could be finished in the console of CB.

1. If there is a new service that begins to use paywall, need to add a new option to the custom field of `TVU Service Type`. Its value could be `TVUSearch`/`Producer`/`Partyline`.  

2. If there is a new billing type for this service, it should be added to the list of the custom field of `Billing Type` of `addon`. For `TVUSearch`, its value could be `mm_download`/`mm_live`.

3. create non-recurring  `addon` and select a `Billing Type` for it .  `addon` is still required even if it's free. Its price could be $0.
   e.g  If you created an addon for `TVUSearch`'s `download` , this billing type of the addon is `mm_download`.

4. [SUB] create `plan` ; set `Restricted addons`; attach those related `addon ` to it in the way of `On Demand`. 

5. [PK] if needing PK, please fill in the custom field `billing credit` of `plan`. Its format follows

   - in JSON

   - `string` is  the same value of  `Billing Type`

     e.g. 
     For `TVU Search` ,  the customer who subscribes the plan will have 50 seconds credits for downloading.

     ```JavaScript
     {
     	"mm_download": 50,
     	"mm_live": 0
     }
     ```

6. share the new added `TVU Service Type` and `Billing Type` with its engineer

7. done

Notes:

* In one TVU service, a customer is just allowed to subscript one plan.

### query customer

Retrieve a customer's information of payment method and those exceptional invoices. All exceptional invoices and its content can be achieved by this API. 

```JavaScript
query{
  customer(customerId:"justinchen@tvunetworks.com",
           product: "TVUSearch"
           ) {
    cardStatus				# valid / expiring / expired / no card
    unbilledCharges			# Total unbilled charges for this customer. in cents, min=0
    exceptionalInvoices		# status == payment_due / not_paid
    {
        id
        customer_id
        status				# https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status
        total
        due_date
        ...
    }
  }
}
```

`card_status`  == `valid` means the customer filed payment way.

ref: https://apidocs.chargebee.com/docs/api/customers#retrieve_a_customer

e.g. curl -L -g -X GET 'https://tvunetworks-test.chargebee.com/api/v2/invoices?customer_id[is]=emmasun4@admin.com&status[is_not]=not_paid' 

### query plan

Retrieve all information of `plan`.  This API will be frequently invoked, so need to pay attention to its performance.
paywalld's client can use this API to 

1. get its quoted price,  
2. the ceiling of feature, which is presented in attachedItems.quantity.
3. get its remaining credit

```JavaScript
query{
  plan(
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "mm_exsource_subpkppu_50"
    ) {
        planId
        price	# 19900 cents
        status  # "active"...
        ...
        chargeItems 		# [] ="applicable_addons". Just show those addons whose "status" is "active"
        {
            id				# "mm_fednet4_ppu_addon_300_nr_live",		
            product 		# "TVUSearch",		== cf_tvu_service_type
            billingType 	# e.g. "mm_live",	== cf_billing_type
            pricing_model	# "per_unit",
            price	
            ...
        }
        attachedItems		# [] =attached_addons, it is used for Feature Control for now.
        {
            id 		
            quantity		# user can use it to decide the ceiling of the charge item.
            billingType		# = cf_billing_type. User can use it map to the feature name of business logic.
            ...
        }
		credit				# = cf_billing_credit. e.g. "{\"pl_hours\": 5}"	
		remainingCredit		# remaining credit. this valus is from subscription.	 e.g. "{\"pl_hours\": 5}"
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/plans?prod_cat_ver=1#retrieve_a_plan
https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#retrieve_an_addon
https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#list_addons
For fetching the value of `billingType` of addon, please use the filter, such as cf_billing_type[is]=pl_participant&cf_tvu_service_type[is]=Partyline&status[is]=active

### query plans

paywalld's client can use this API to get the existing plan.

```JavaScript
query{
  plans(
        product: "TVUSearch"
    ) {
        "number": 2,
		"planIds":[ "mm_fednet5_subppu_199", "mm_fednet_special_knbc_0" ]        
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/plans?prod_cat_ver=1&lang=curl#retrieve_a_plan

### query subscription

return the information of customer's subscription.  paywalld's client can use this API to check whether the customer has an active subscription of the plan and its remaining credit. 

 `paywalld` ensures that one `plan` just has one `subscription` for one user. 

```JavaScript
query {
    subscription (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "jj_plan_pk"
    ) {
         subscriptionId 	
         status				# active/future/in_trial/non_renewing/paused/cancelled  
    	 deleted			# true / false         
         remainingCredit	# remaining credit. = cf_billing_credit. e.g. "{\"pl_hours\": 5}"					
         ...
	}
}
```

ref:  https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#retrieve_a_subscription
The explanation of `status`: https://apidocs.chargebee.com/docs/api/subscriptions#subscription_status

### query chargeItem 

Retrieve all information of `addon`.  paywalld's client can use this API to get its quoted price.

```JavaScript
query{
  chargeItem (
        customerId: "justinchen@tvunetworks.com",	//which is useless for paywalld now
        product: "TVUSearch",
        chargeItem: "mm_fednet4_ppu_addon_300_nr_live"
    ) {
        # ="applicable_addons". Just show those addons whose "status" is "active"
		id				# "mm_fednet4_ppu_addon_300_nr_live",		
        product			# "TVUSearch",	== cf_tvu_service_type
        billingType 	# "mm_live",	== cf_billing_type
        pricingModel	# "per_unit",
        price			# 300,
        ...
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/addons?prod_cat_ver=1#retrieve_an_addon

### query customerPortal

The end-user can use the self-service portal to maintain their billing information / invoice / subscription. 

```JavaScript
query{
  customerPortal (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        redirectUrl: "https://search.tvunetworks.com"
    ) {
        id			# "portal_AzZlxDSWcJOT1FvC",
        token		# aRNcIRmnenKuV67D07JlT8tC07caVpPw",
        access_url	# https://tvunetworks-test.chargebee.com/portal/v2/authenticate?token=aRNcIRmnenKuV67D07JlT8tC07caVpPw"
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/portal_sessions?prod_cat_ver=1#create_a_portal_session

### query customerPortalStatus

client can use it to check the final status of checkout.

```JavaScript
query{
  customerPortalStatus (
        hostedPageID: "b9Z5Sce6AxFBNBMLu4qcuJHFthSF4VI0I"
    ) {
        type		# "checkout_one_time"
        url
        created_at	# 1622021085
        invoice {	# type invoice
            id
            customer_id
            status	# "payment_due", check below
            total	
            amount_paid
			amount_due
            amount_to_collect
        }
    }
}
```

`invoice.status`: 
If the payment succeeds, it is marked as **paid**. If the payment fails, the invoice is marked as **payment_due** and retry settings are taken into account. If no retry attempts are configured, the invoice is marked as **not_paid**. If the amount due is zero or negative, the invoice is immediately marked as **paid** and the balance if any is carried forward to the next term of the invoice.

ref: https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page
https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#invoice_status

### mutation createCustomer

```JavaScript
mutation{
  createCustomer(
    customerId:"justinchen@tvunetworks.com",	//using TVU's account(email address) as Chargebee(CB)'s ID
    email:"justinchen@tvunetworks.com",	// normally same as the value of `id`
    firstName:"Justin",
    lastName:"Chen"
  ) {
      customer{
          id
      }
  }
}
```

ref: https://apidocs.chargebee.com/docs/api/customers?prod_cat_ver=1#create_a_customer

### mutation updatePayment

return hosted page. paywalld's client can use this API to allow customer to update his payment way.

```
mutation{
  updatePayment(customerId:"justinchen@tvunetworks.com") 
  {
  	hostedPage {
  		...
	}
  }
}
```

ref: https://apidocs.chargebee.com/docs/api/hosted_pages#manage_payment_sources

### mutation checkoutOneTimePageAmount

for PPU. return hosted page. no need add-on.

```JavaScript
mutation {    
    checkoutOneTimePageAmount (
        customerId:"justinchen@tvunetworks.com",
        product: "TVUSearch",
        description: "FedNet01 April 29 12:33:01 7 minutes download",
        amount: 84	//$ cent
        )
    {
      hostedPage {
          ...
	  }
    }
}
```

> How to check the status of payment?

Two ways:

* The frontend webpage can handle it. The prerequisite is paywalld needs to enable `iframe_messaging`  while get the hosted page.

* The backend service uses `hosted_page.id` to query the payment status. If paid, the backend can save the state for future reference. 
  ref: https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#retrieve_a_hosted_page

### mutation checkoutOneTimePageQuantity

for PPU. return hosted page. The add-on should be created in advance, and its `Charge Type` must be `One-time`.  And share it with the client of paywalld, such as MediaMind. 

  ```JavaScript
mutation {    
    checkoutOneTimePageQuantity (
    customerId:"justinchen@tvunetworks.com",
     product: "TVUSearch",
     description: "FedNet01 April 29 12:33:01 7 minutes download",

     chargeItem: "addon_name",  // create it in advance,
     quantity: 15
    )
    {
        hostedPage {
            ...
        }
	}
}
  ```

`hosted page`:
e.g.

```JavaScript
{
    "hosted_page": {
        "id": "b3coJuK1RJkcoRCP69d9gdqCKMcOcwxy",
        "type": "checkout_one_time",
        "url": "https://tvunetworks-test.chargebee.com/pages/v3/b3coJuK1RJkcoRCP69d9gdqCKMcOcwxy/",
        "state": "created",
        "embed": false,
        "createdAt": 1618720670,
        "expiresAt": 1618731470,
        "object": "hosted_page",
        "updatedAt": 1618720670,
        "resourceVersion": 161872060898
  }
}
```

ref: https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutOneTime-usecases

### mutation checkoutOneTimeInvoiceAmount

for PPU. API returns the status of invoice. It applies to the case that no need the confirmation from customer, such as `mm_takelive` of `tvusearch`. no add-on. Need to check customer's billing information before using it in business logic. 
ref: https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create-usecases

```JavaScript
mutation {    
    checkoutOneTimeInvoiceAmount (
        customerId:"justinchen@tvunetworks.com",
        product: "TVUSearch",
        description: "FedNet01 April 29 12:33:01 7 minutes download",
        amount: 84	//$ cent
        )
    {
        invoiceId: "xxxx",
        invoiceStatus: "paid",
        amountPaid: 84
        total: 84
    }
}
```

### mutation checkoutOneTimeInvoiceQuantity

`add-on` can be used here. It should be created in advance, and its `Charge Type` must be `One-time`. The client of paywalld, such as MediaMind.
ref: https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create-usecases

  ```JavaScript
mutation {    
    checkoutOneTimeInvoiceQuantity (
    customerId:"justinchen@tvunetworks.com",
     product: "TVUSearch",
     description: "FedNet01 April 29 12:33:01 7 minutes download",       
     chargeItem: "addon_name",  // create it in advance,
     quantity: 15
    )
    {
        invoiceId: "xxxx",
        invoiceStatus: "paid",
        amountPaid: 84
        total: 84
    }
}
  ```



### mutation cancelSubscription

 `paywalld` ensures that one `plan` just has one `subscription` for one user.  Return the same structure as `query subscription` did.

```JavaScript
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

ref:  https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#cancel_a_subscription

### mutation cancelInvoice

 If a prepay invoice fails to charge, it should be canceled immediately. 

```JavaScript
query {
    cancelInvoice (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        invoiceID: "735"
    ) {
         invoiceID
         status
	}
}
```

ref:  

### mutation checkoutNewSubscription

return hosted page.  Return `success` when the customer has an existing plan already. 

```JavaScript
mutation {
    checkoutNewSubscription (
        customerId: "justinchen@tvunetworks.com",
		product: "TVUSearch",
    	planId: "mm_exsource_subpkppu_50",
    ) {
      hostedPage {
          ...
	  }
    }
}
```

[PK] copy the custom field `billing credit` of `plan` onto the custom field `billing credit` of `subcription`.  It follows JSON.
`iframe_messaging` should be `true`. `embed` should be `true`.
ref: https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkoutNew-usecases

[PK]  There is **a periodic task for PK mode: resetting the `billing credit` at the end of every billing period**.
paywalld should provide an API to receive the notification from Chargebee for event `subscription renewed`.  Its format is listed in Appendix 1.
ref: https://tvunetworks-test.chargebee.com/apikeys_and_webhooks/webhooks

### mutation checkoutNewSubscriptionInvoice

If no suitable payment method, this API returns `failure` .  If the customer has an existing and active plan, API returns `success`. 

```JavaScript
mutation {
    checkoutNewSubscriptionInvoice (
        customerId: "justinchen@tvunetworks.com",
		product: "TVUSearch",
    	planId: "mm_exsource_subpkppu_50",
    ) {
        invoice {
            invoiceId: "xxxx",
            invoiceStatus: "paid",
            amountPaid: 84
            total: 84  
            
            subscriptionId
        }
        subscription {		//subscription structure
            ...
        }
        customer {
            ...
        }
    }
}
```

[PK] copy the custom field `billing credit` of `plan` onto the custom field `billing credit` of `subcription`.  It follows JSON.
ref: https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#create_subscription_for_customer

### mutation checkoutExistingSubscriptionInvoice

checkout an existing subscription with billing_type(add_on). API returns the status of invoice. So it can use the same class as the return value of checkoutOnetimeInvoice.

paywalld's client can use this API to charge customer's add-on that attaches to an active subscription.

```JavaScript
mutation {
    checkoutExistingSubscriptionInvoice (
        customerId: "justinchen@tvunetworks.com",
        product: "TVUSearch",
        planId: "mm_exsource_subpkppu_50",
        billingType: "mm_download",
        description: "FedNet01 April 29 12:33:01 7 minutes download",    
        quantity: 120
    ) {
        invoice {
            invoiceId	
            invoiceStatus	# "paid"
            amountPaid
            total
        }
    }
}
```

ref: https://apidocs.chargebee.com/docs/api/invoices?prod_cat_ver=1#create_invoice_for_a_non-recurring_addon

Internal workflow:  

1. check whether the customer has an active subscription. if has no, return error.

2. check whether the customer has no unbilled charges. If has, return error. 

3. get `plan` 's `applicable_addons` list.
   ref: https://apidocs.chargebee.com/docs/api/plans#retrieve_a_plan

4. get the addon ID whose `cf_billing_type` has the same value as "billing_type" of the API input.

3. If  the custom field `billing credit` of `subscription` is empty, no PK is involved. 
   directly invoices `addon` with the `quantity` in API to the customer.
   Return the information of `invoice`. Most important value is "invoice"."amount_paid" and 
   
   "invoice"."[status](https://apidocs.chargebee.com/docs/api/invoices#invoice_status)" 
   
6. [PK] If  `billing credit` has value, parse it and get the remained `credit/quantity ` of the addon of the subscription.  

5. [PK] if (remained credit >= charging `quantity` in API)
   invoice addon with the `quantity` in API, but its price should be $0(addon_unit_price=$0).
   
6. [PK] if (remained credit < charging `quantity` in API) 
   invoice addon with those remained credit in $0.
   And invoice addon with the non-free quantity(charging quantity - remained credit) in the default price.
   
7. [PK] If invoice.status == paid, update the JSON of the custom field `billing credit` of `subscription`. 
   ref: https://apidocs.chargebee.com/docs/api/subscriptions?prod_cat_ver=1#update_a_subscription
   
8. [PK] Return the information of `invoice`.



> ~~mutation checkoutExistingSubscriptionPage~~

~~typically in the trial state~~

~~ref: https://apidocs.chargebee.com/docs/api/hosted_pages?prod_cat_ver=1#checkout_existing_subscription~~

### mutation createPlan

### mutation createChargeItem



# Billing models and their essential APIs

## PPU

### case1:  the business service knows the final price before carry out the action that is not free

mutation checkoutOneTimePageAmount

### case2: the business service knows the final price after carried out the action that is not free

query customer: use this API to check whether customer correctly filed his payment way.

mutation checkoutOneTimeInvoiceAmount: charge customer when we know the final amount of the bill.

### case3: the business service knows the count of charged item before carry out the action that is not free 

mutation checkoutOneTimePageQuantity

### case4: the business service knows the count of charged item  after carried out the action that is not free 

query customer: use this API to check whether customer correctly filed his payment way.

mutation checkoutOneTimeInvoiceQuantity: charge customer when we know the final amount of the bill.

## SUB

### case 5: charge customer a fixed amount each charging period

query subscription: use this API to check whether the customer has an active subscription of the plan. 

mutation checkoutNewSubscription

mutation checkoutExistingSubscriptionInvoice:  this is optional for this case.

## SUB+PPU

### case 6:  charge customer a fixed amount each charging period and extra charge customer when he carry out  the action that is not free 

query subscription: use this API to check whether the customer has an active subscription of the plan. 

mutation checkoutNewSubscription

mutation checkoutExistingSubscriptionInvoice

## SUB+PPU+PK

### case 7:  when customer subscribed a plan, he will get amount of free credits for the charge item

same as case 6. The details is hidden by paywalld.







### Appendix

1. message body of subscription_renewed

```JavaScript
 {
    "id": "ev_Azz5oUSRtOs3v1CkB",
    "occurred_at": 1615960809,
    "source": "scheduled_job",
    "object": "event",
    "api_version": "v2",
    "content": {
        "subscription": {
            "id": "jj_sub_pk",
            "plan_id": "jj_plan_pk",
            "plan_quantity": 1,
            "plan_unit_price": 95000,
            "billing_period": 1,
            "billing_period_unit": "week",
            "customerId": "AzZltkSPvMtaRKGr",
            "plan_amount": 95000,
            "plan_free_quantity": 0,
            "status": "active",
            "current_term_start": 1615960800,
            "current_term_end": 1616565599,
            "next_billing_at": 1616565600,
            "created_at": 1615445442,
            "started_at": 1615359600,
            "activated_at": 1615359600,
            "updated_at": 1615960808,
            "has_scheduled_changes": false,
            "resource_version": 1615960808948,
            "deleted": false,
            "object": "subscription",
            "currency_code": "USD",
            "addons": [{
                "id": "-justin_sub_pk_ppu_recurring_week",
                "quantity": 1,
                "unit_price": 20,
                "amount": 20,
                "object": "addon"
            }],
            "due_invoices_count": 0,
            "mrr": 412876,
            "exchange_rate": 1.0,
            "base_currency_code": "USD",
            "shipping_address": {
                "first_name": "JUSTIN",
                "last_name": "CHEN",
                "email": "JUSTINCHEN@TVUNETWORKS.COM",
                "phone": "6504395678",
                "line1": "857 Maude Avenue",
                "city": "Mountain View",
                "state_code": "CA",
                "state": "California",
                "country": "US",
                "zip": "94043",
                "validation_status": "not_validated",
                "object": "shipping_address"
            }
        },
        "customer": { },
        "card": { },
        "invoice": {}
    },
    "event_type": "subscription_renewed",
    "webhook_status": "not_configured"
}
```

