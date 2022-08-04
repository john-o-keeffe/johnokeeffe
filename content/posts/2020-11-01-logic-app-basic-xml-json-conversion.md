---
title:  "Logic App: Basic XML to JSON Expression Conversion"
date: 2020-11-01T18:09:34+01:00
draft: false
---

# Introduction

Working with JSON data in logic apps is native and really is a plus for the PaaS solution. Working with XML is also relatively easy, even with xpaths but it is not native like JSON. When I came across the problem myself I found it much faster and easier taking the approach of simply converting to JSON as I was much more familiar with it.

Below is an overview of the approach I take when working with XML data.

## Scenario

In the below scenario we receive online orders from our web store. We can receive order requests that could be sent anywhere in the world. When an order comes in, we use the country field under the shipping address as the conditional value. Based off of this value we then send the order request to the distributor supplying that region.

```xml
<?xml version="1.0"?>
<PurchaseOrder PurchaseOrderNumber="99503" OrderDate="1999-10-20">
  <Address Type="Shipping">
    <Name>Adam Murphy</Name>
    <Street>123 Maple Street</Street>
    <City>Mill Valley</City>
    <State>CA</State>
    <Zip>10999</Zip>
    <Country>Ireland</Country>
  </Address>
  <Address Type="Billing">
    <Name>Tai Yee</Name>
    <Street>8 Oak Avenue</Street>
    <City>Old Town</City>
    <State>PA</State>
    <Zip>95819</Zip>
    <Country>Ireland</Country>
  </Address>
  <DeliveryNotes />
  <Items>
    <Item PartNumber="872-AA">
      <ProductName>Lawnmower</ProductName>
      <Quantity>1</Quantity>
      <USPrice>148.95</USPrice>
      <Comment>Confirm this is electric</Comment>
    </Item>
  </Items>
</PurchaseOrder>
```


|Azure|Postman|GitHub Repo|
|:-:|:-:|:-:|
|[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjohn-o-keeffe%2FBlogResources%2Fmain%2Flogic-app-basic-xml-json-conversion-samples%2Fazuredeploy.json)|[![Run in Postman](https://run.pstmn.io/button.svg)](https://app.getpostman.com/run-collection/bda8ade15f3e3acddcf6)|[Github Link][Github-repo]|



## Walk-through


### Logic App

#### HTTP Trigger

First setup a HTTP trigger, this will be needed for testing and accepting the XML body content. We will be using [Postman][Postman] in this case to test the logic app URI.

#### Initialize Variable

Now using the initialize variable action we create a string. I use this as an easy way to test and develop expressions, as you can see the expression develop overtime. Troubleshooting using some of the control actions is not easy.

When creating expressions it is always easier to take it one function at a time.

Step 1: First we convert the HTTP trigger body to XML.

![Logic app http and variable](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/logic-app-basic-xml-json-conversion-samples/logicapp-http-variable-01.png)


- This then outputs the trigger body content in it's XML format. In this case it is always better to be explicit here.

```plain
xml(triggerBody())
```


![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-variable-02.png?raw=true)
   
Step 2: We convert the XML to JSON. This will then allow us to access these elements.


![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-variable-03.png?raw=true)

- This is easy to do so far due to the native XML and JSON expression functions.

```plain
json(xml(triggerBody()))
```

![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-variable-04.png?raw=true)

```json
{
    "?xml": {
        "@version": "1.0"
    },
    "PurchaseOrder": {
        "@PurchaseOrderNumber": "99503",
        "@OrderDate": "1999-10-20",
        "Address": [
            {
                "@Type": "Shipping",
                "Name": "Ellen Adams",
                "Street": "123 Maple Street",
                "City": "Mill Valley",
                "State": "CA",
                "Zip": "10999",
                "Country": "USA"
            },
            {
                "@Type": "Billing",
                "Name": "Tai Yee",
                "Street": "8 Oak Avenue",
                "City": "Old Town",
                "State": "PA",
                "Zip": "95819",
                "Country": "USA"
            }
        ],
        "DeliveryNotes": "Please leave packages in shed by driveway.",
        "Items": {
            "Item": [
                {
                    "@PartNumber": "872-AA",
                    "ProductName": "Lawnmower",
                    "Quantity": "1",
                    "USPrice": "148.95",
                    "Comment": "Confirm this is electric"
                },
                {
                    "@PartNumber": "926-AA",
                    "ProductName": "Baby Monitor",
                    "Quantity": "2",
                    "USPrice": "39.98",
                    "ShipDate": "1999-05-21"
                }
            ]
        }
    }
}
```

Step 3: We now start to go down the JSON path to get the country object.

- PurchaseOrder
  - Address[0]
    - Country

Due to the schema selected for this order request there were two address elements. Above shows, having converted it to JSON the address object is now in an array.
It is easy to overcome in this case, as the address we want is on the zero index of this array.


![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-variable-06.png?raw=true)

- We can see below the value output is the JSON country object.

![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-variable-05.png?raw=true)

```plain
json(xml(triggerBody()))['PurchaseOrder']['Address'][0]['Country']
```

We have a full expression that we can now use in a switch control. Using this in conjunction with HTTP request actions, we can start to send our orders to the distributors.
In this example we will create two [Request Bin][RequestBin] endpoints for testing.

- In the switch statement below we can use the variable we just created called 'country'. In addition to this we can add a 'tolower' function to the expression as the switch action values are case sensitive.


```plain
tolower(json(xml(triggerBody()))['PurchaseOrder']['Address'][0]['Country'])
```

![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-switch-http-07.png?raw=true)


![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/logicapp-switch-http-08.png?raw=true)

The request bin response:

![Logic app http and variable](https://github.com/john-o-keeffe/BlogResources/blob/main/logic-app-basic-xml-json-conversion-samples/requestbin-response-09.png?raw=true)


## Conclusion

This isn't the only approach to working with XML data in logic apps. I found if you are familiar with JSON in logic apps and XML is new to you, this is a quick and easy way to work with XML data.

## Software Used

- VS Code
- Postman
- Request Bin

[Github-repo]: https://github.com/john-o-keeffe/BlogResources/tree/main/logic-app-basic-xml-json-conversion-samples
[RequestBin]: https://requestbin.com/
[Postman]: https://www.postman.com/downloads/