---
title:  "Power Automate: Handling JSON schema validation with Parse JSON"
date: 2021-11-01T18:09:34+01:00
draft: false
---


# Introduction

Had a scenario recently when working on a project where I needed to check some required columns from a staging excel file before it was inserted into a master excel file. Now in this case it only was a small subset of the overall columns in the excel were actually needed for the process. But I didn't want the whole process to fall over if only one row was missing so I used the some scopes, parse JSON and JSON schema magic to build the below solution to get around it.


# Walk-through


## Scenario

In the below fake scenario we receive car owner data in an excel table. We need to take this data and insert the rows in a master excel for analysis. We're really only looking for the owner personal data and the car vehicle identification
number (vin) as we can lookup the rest of the data from the vin. But if this vin is missing then this data isn't as useful to us as we can't guarantee they will also provide there car details.

## Sample Data

| id  | first_name | last_name | email                   | car_make   | car_model        | car_model_year | car_vin           |
| --- | ---------- | --------- | ----------------------- | ---------- | ---------------- | -------------- | ----------------- |
| 1   | Chrisse    | Marieton  | cmarieton0@umn.edu      |            |                  |                | 2G61L5S37E9694881 |
| 2   | Dion       | Frenchum  | dfrenchum1@nifty.com    |            |                  |                | NM0AE8F74E1334359 |
| 3   | Brockie    | Bushaway  | bbushaway2@google.co.uk |            |                  |                | WBAUN1C55DV305824 |
| 4   | Janie      | Iacomelli | jiacomelli3@uiuc.edu    |            |                  |                | WA1AGAFE7CD051379 |
| 5   | Cassie     | Housby    | chousby4@icq.com        | Mitsubishi | Lancer Evolution | 2004           |                   |

Sample Data Schema [Mockaroo Link](https://www.mockaroo.com/1591bbc0)

## Flow Structure Setup

To get started I just created a basic flow structure:

- Variable outputData is an array and we will use this later on as we append data to it
- Variable outputAction is going to be use to output the action that was taken ie. *"inserted into master table" or "error missing column data"*
- List rows present in the excel table (see above for schema and sample data)

I like to use scopes to help with error handling and logical design of the flows. I won't be discussing scope error handling in this blog but here is a great resource to read over to get you started [Advanced error handling in Power Automate](https://www.portiva.nl/portiblog/advanced-error-handling-in-power-automate)

![Power Automate setup flow](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/power-automate-handling-json-schema-validation-with-parse-json/pa-setup-01.png)

## Flow JSON Schema Validation Setup

In this step I just create a loop to run through each row returned from the *list rows present in a table* action from above. Now we need to generate the schema for the parse json easiest way to do this is to just run the flow once. Now what we need to do is take this schema and add some constraints too it. Now in the below example all the data types are strings but this can work with other data types. See the [JSON Schema Docs](https://json-schema.org/understanding-json-schema/index.html)

Below you can see I have added the [minLength](https://json-schema.org/understanding-json-schema/reference/string.html#id5) constraint to some of the objects. This means JSON is looking for each of the objects to have at least one string character present otherwise it will create a validation error. This will be the error we will read from later on. Now just create two more scopes one for the "Happy path" (Success) and the "Unhappy Path" (Error). Setup the "Unhappy path" to only run on **has failed**.

 ```json
  {
   "car_vin": {
            "type": "string",
            "minLength": 1
        }
  }
```

```json
{
    "type": "object",
    "properties": {
        "@@odata.etag": {
            "type": "string"
        },
        "ItemInternalId": {
            "type": "string"
        },
        "id": {
            "type": "string",
            "minLength": 1
        },
        "first_name": {
            "type": "string",
            "minLength": 1
        },
        "last_name": {
            "type": "string",
            "minLength": 1
        },
        "email": {
            "type": "string",
            "minLength": 1
        },
        "car_make": {
            "type": "string"
        },
        "car_model": {
            "type": "string"
        },
        "car_model_year": {
            "type": "string"
        },
        "car_vin": {
            "type": "string",
            "minLength": 1
        }
    }
}
```

![Power Automate main scope setup with parse json](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/power-automate-handling-json-schema-validation-with-parse-json/pa-main-scope-loop-parse.png)



## Unhappy Path (Important Step)

This is the important step and a key to the whole flow. Setup any action in this case im using a compose to make it clearer but setup an expression like the one below that will take the error output from the Parse JSON. 

```plain
  outputs('Parse_JSON')?['errors']
```
The error output will only appear when the validation has failed otherwise it just displays the body like normal. It outputs an array as seen in the below example. The key objects to look at the **Path**, **errorType** and **message**. If you have more complex schemas some more of this information will become useful to you when handling errors for me in this case I only need two.

```json
 [
    {
        "message": "String '' is less than minimum length of 1.",
        "lineNumber": 0,
        "linePosition": 0,
        "path": "car_vin",
        "value": "",
        "schemaId": "#/properties/car_vin",
        "errorType": "minLength",
        "childErrors": []
    }
]
```

Now that I have an array with the errors im going to create another loop, to go though these errors set my *outputAction* variable with a custom message. This will then be append to the *outputData* array which will be used later on for a emailed report.

```plain
  items('Apply_to_each_error')?['path']
```

![Power Automate Unhappy Path](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/power-automate-handling-json-schema-validation-with-parse-json/pa-unhappy-path.png)


## Happy Path

The happy path in this case is pretty simple and is where you would put the insert into excel table action. In this case I am just setting the *outputAction* variable again with a custom message to say the insert was a success. Then just appending this to the array which will be used next.


![Power Automate Happy Path](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/power-automate-handling-json-schema-validation-with-parse-json/pa-happy-path.png)



## Output

Last but not least the output of all that data we just logged into the *outputData* variable. Because the variable I had created at the start is an array this will allow us to use the *create html table* action easily, to create a html table we can use in Email, Teams etc...



![Power Automate Output](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/power-automate-handling-json-schema-validation-with-parse-json/pa-output.png)

![Power Automate Output](https://raw.githubusercontent.com/john-o-keeffe/BlogResources/main/power-automate-handling-json-schema-validation-with-parse-json/pa-email-report.png)


## Conclusion

This isn't the only approach to error handling in power automate. But I found in this case it was useful way of solving my issue and also learnt something new in the process.