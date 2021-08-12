# Developer Guide for Partyline

## 1. for the client of Billing Service:

### essential APIs

[query cpermission](https://justinchen00.github.io/page/billingservice/API%20definition.html#query-permission)
[query previewCustomSubscription](https://justinchen00.github.io/page/billingservice/API%20definition.html#query-previewcustomsubscription)
[mutation checkoutNewSubscription](https://justinchen00.github.io/page/billingservice/API%20definition.html#mutation-checkoutnewsubscription)

### Common Rate Card

workflow and related APIs

1. query plans
   fetch the existing plans with `type`="custom" for given product
2. query plan
   fetch the details of each plan for display.
3. mutation checkoutNewSubscription
   provides `a hosted page` for customers to finish checkout/create a new subscription for a given valid plan ID. 
   [TUTORIALS](https://www.chargebee.com/checkout-portal-docs/checkout-new-tutorial.html) or consult my team for how to use `the hosted page`

### Custom Rate Card

workflow and related APIs

1. query plans
    fetch the existing plans with `type`="custom" for given product
2. query plan
    fetch the details of each plan, 
3. query previewCustomSubscription
    return the price
4. mutation checkoutNewSubscription
    Based on the provided number of charge items and formula, generates a new subscription. Then provides `hosted page` for customers to finish checkout/create new subscriptions for a given valid plan ID. 
    [TUTORIALS](https://www.chargebee.com/checkout-portal-docs/checkout-new-tutorial.html) or consult my team for how to use `the hosted page`

## 2. for PM:

* need to  preset a plan with price $0 for custom plan.  add all optional add-on to it. providing readable `Description`

* need a custom field to tag the custom plan. I created a CF named  `Custom`

* select `custom` for custom plan.  add `formula` in `JSON meta data` . 
  e.g. 
  `JSON meta data`  of `partyline-common-custom`

  ```json
  {"formula": "$partyline-hour.quantity*0.25*(2*$partyline-participant.quantity*(1+225/100)+$partyline-output.quantity*(1+225/100))"}
  ```

   `$partyline-output.quantity`  means the quantity of the addon `partyline-output` in this subscription.
the attribute  `mandatory`/`Recommended` of addon could be utilized in the business layer later.
  
* Partyline needs to save `chargeItem` and create the mapping relationship between the business features with them. 
  Such as,  `partyline-output`=2 means the user allows to have 2 outputs at most.

* ~~when there are several addons with the same `billing type`, addon's name will be parsed for some purposes.~~ 

  It's not feasible to get the value of the addon attribute CF("cf_billing_type") via one API invoking. Using a loop to fetch attributes of each addon is time-consuming. It's fine for `TVUSearch`. Because there are just two addons. However, For Partyline, a plan will be attached with lots of addons, and those addons remain unchanged after the billing model is nailed down. So let's use the addon.id as the feature name and no need to change CF("cf_billing_type").  
  Please make sure the addon.id of Partyline is short and straightforward, such as `pl-output` or `partyline-output`.
  
* For each addon mosaic*, its attributes will be a part of the formula.
  e.g.  `mosaic7x7.weight` means the `weight` value saved in JSON metadata of addon `mosaic7x7`, which is 49.
  ![image](https://user-images.githubusercontent.com/18137639/128147207-8e954061-79dc-4eb2-8fb8-cf599d3652ca.png)

  
  e.g. Selling Price for Gallery:
  
  ```python
  {"formula": "$partyline-hour.quantity*(0.25*($partyline-output.quantity*(1+2.5) + $partyline-participant.quantity*(1+2.5) + sum(mosaics)*(1+2.5) + 0.085*max(mosaics)*(1+2.5)}"}
  
  sum(mosaics)= $mosaic3x3.weight*$mosaic3x3.quantity + $mosaic4x4.weight*$mosaic4x4.quantity + 
  $mosaic5x5.weight*$mosaic5x5.quantity + ...
  
  max(mosaics)= max(mosaic3x3.weight, mosaic4x4.weight, mosaic5x5.weight, ...)

  The final formula is:
  {"formula": "$partyline-hour.quantity*(0.25*($partyline-output.quantity*(1+2.5) + $partyline-participant.quantity*(1+2.5) + ($mosaic3x3.weight*$mosaic3x3.quantity + mosaic4x4.weight*$mosaic4x4.quantity + $mosaic5x5.weight*$mosaic5x5.quantity + ...)*(1+2.5) + 0.085*max(mosaic3x3.weight, mosaic4x4.weight, mosaic5x5.weight, ...)*(1+2.5)}"}
  ```

  e.g. 
  mosaics: `3x3, 4x4, 4x4, 6x6, 7x7, 0, 0, 0`
  The addons for this are:
  
   - `Mosaic3x3`
   - `Mosaic4x4`*2
   - `Mosaic6x6`
   - `Mosaic7x7`

  In the price calculator for mosaic, the formula is:
  
  **Selling Price for Gallery** =

  ```
  Selling Price for Gallery = [#OfHours]*{$0.25*(#OfOutputs*(1+[OutputMarkup%]/100) + #OfParticipants*(1+[ParticipantMarkup%]/100) + sum(mosaics)*(1+[MosaicMarkup%]/100)) + 0.085*max(mosaics)*(1+[MosaicMarkup%]/100)}
  ```
  
  The `sum(mosaics)` variable is `3x3 + 2*(4x4) + 6x6 + 7x7 = 126`
  The `max(mosaics)` variable is `max(3x3, 4x4, 6x6, 7x7) = 7x7 = 49`

## 3. More documents

1. Miro
   https://miro.com/app/board/o9J_lbF7xGI=/?moveToWidget=3074457361905206409&cot=14
2. 

