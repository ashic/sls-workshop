# Serverless Sanity
-------------------------------

=========================
# Agenda
=========================

We will attempt to coder:

* Installing serverless framework.
* Creating a project. 
* Deploying it to AWS. 
* Adding GET / POST endpoints that use a library in the codebase. 
* Creating an AWS resource (dynamodb table) within serverless. 
* Creating necessary IAM permissions. 
* Assigning environment variables.
* Auth. 
* Testing
* More? 

=========================
# Up and Running
=========================

## 1. installation

* install node. 
* npm install -g serverless

## 2. aws credentials

* create an admin user. Note down AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY

```
sls config credentials --provider aws --key $AWS_ACCESS_KEY_ID --secret $AWS_SECRET_ACCESS_KEY --profile profile_name
```

## 3. look at sls create --help

## 4. create the project

```
mkdir slsdemo
cd slsdemo
sls create -t aws-nodejs
```

## 5. check the code

* change the service name
* add profile
* add aws region

```
provider:
  name: aws
  runtime: nodejs8.10
  profile: sls-admin
  region: eu-west-1
```

## 6. deploy!

```
sls deploy
```

## 7. Go to AWS console. Check CloudFormation. 

## 8. Go to Lambda. 

Don't change stuff here. Subsequent deploy will undo. 

## 9. Invoke

```
sls invoke -f hello
```

## 10. Invoke locally

```
sls invoke local -f hello
```

## 11. http endpoint

```
functions:
  hello:
    handler: handler.hello
    events:
      - http:
         path: say-hello
         method: GET
```

deploy. 

================================================
# A bit more
================================================

## 12. Add notes/index.js

```
let notes = [
    "Go to the office",
    "Pretend to work",
    "Have lunch"
]



module.exports.getAllNotes = () => Promise.resolve(notes)
```

## 13. Add views/notes.js


```
const notes = require('../notes') 


module.exports.index = async (event, context) => {
    return {
        statusCode: 200,
        body: JSON.stringify({
            notes: await notes.getAllNotes()
        }),
    };
}
```

## 14. Update serverless.yaml with a new endpoint

```
functions:
  hello:
    handler: handler.hello
    events:
      - http:
         path: say-hello
         method: GET
  
  notes_list:
    handler: views/notes.index
    events:
      - http:
          path: notes
          method: GET
```


## 15. Deploy and test.

## 16. Query string params

```
    if (event.queryStringParameters && event.queryStringParameters.q) {
        const res = await notes.searchNotes(event.queryStringParameters.q)
        return {
            statusCode: 200,
            body: JSON.stringify({
                notes: res
            })
        }
    }
```

## 17. POST

In notes/index.js

```
module.exports.addNote = (note) => Promise.resolve(notes.push(note))
```

In views/notes.js

```
module.exports.add = async (event, context) => {

    const data = JSON.parse(event.body)
    
    if (!data.text) {
        return {
            statusCode: 400,
            body: JSON.stringify({
                message: "Please provide 'text'",
                event
            })
        }
    }

    await notes.addNote(data.text)

    return {
        statusCode: 200,
        body: JSON.stringify({
            message: "Note added successfully."
        })
    }

}
```

In serverless.yaml

```
  notes_add:
    handler: views/notes.add
    events:
      - http:
          path: notes
          method: POST
```


Deploy. 

======================================
# Add a Database
======================================

## 18. Resources - Dynamodb

In serverless.yaml, add the resource:

```
resources:
  Resources:
    demoBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: ${self:service}-${self:provider.stage}-demo1

    notesTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-${self:provider.stage}-notes
        AttributeDefinitions:
          - AttributeName: note
            AttributeType: S
        KeySchema:
          - AttributeName: note
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
```

Add IAM permissions, and make the table available as an environment variable:

```
provider:
  name: aws
  runtime: nodejs8.10
  profile: sls-admin
  region: eu-west-1
  environment:
    DynamoTable: ${self:resources.Resources.notesTable.Properties.TableName}
    
  iamRoleStatements:
   - Effect: "Allow"
     Action:
       - dynamodb:DescribeTable
       - dynamodb:Query
       - dynamodb:Scan
       - dynamodb:GetItem
       - dynamodb:PutItem
       - dynamodb:UpdateItem
       - dynamodb:DeleteItem
     Resource: "arn:aws:dynamodb:${self:provider.region}:*:*" 
```

Add a package.json to the root:

```
{
    "name": "my-project"
}
```
```
yarn add aws-sdk
```

Add notes/dynamo-notes.js:

```
const aws = require('aws-sdk')
const db = new aws.DynamoDB.DocumentClient()
const dynamoTable = process.env.DynamoTable


module.exports.addNote = (note) => {

    console.log(`writing to ${dynamoTable}`)

    const created = new Date().getTime()
    const p = {
        "TableName": dynamoTable,
        "Item": {
            note,
            created
        }
    }

    return db.put(p).promise()
}


module.exports.getAllNotes = () => db.scan({TableName: dynamoTable}).promise()


module.exports.searchNotes = (note) => db.get({
    TableName: dynamoTable,
    Key: {
        note
    }
}).promise()
```

Flip to the dynamo one in views/notes.js:

```
// const notes = require('../notes')
const notes = require('../notes/dynamo-notes')
```

Deploy

===============================================
## Auth
===============================================


## 19. API Keys

In serverless.yaml, add an API key, and make an endpoint private:

```
provider:
  name: aws
  runtime: nodejs8.10
  profile: sls-admin
  region: eu-west-1
  environment:
    DynamoTable: ${self:resources.Resources.notesTable.Properties.TableName}
  apiKeys:
    - my-secret-api-key
```


```
  notes_list:
    handler: views/notes.index
    events:
      - http:
          path: notes
          method: GET
          private: true
```

Deploy. Make note of the api key.
Make request with x-api-key header.

## 20. Quoatas and Throttling

```
provider:
  name: aws
  runtime: nodejs8.10
  profile: sls-admin
  region: eu-west-1
  environment:
    DynamoTable: ${self:resources.Resources.notesTable.Properties.TableName}
  apiKeys:
    - my-secret-api-key
  usagePlan:
    quota:
      limit: 5
      offset: 0
      period: DAY
    throttle:
      burstLimit: 10
      rateLimit: 10
```



======================================
## Custom Authoriser
======================================

* yarn add jsonwebtoken

* Add views/tokens.js

```
const jwt = require('jsonwebtoken')
const SECRET = "abracadabra"


module.exports.generate = async (event, context) => {

    const token = jwt.sign(event.body, SECRET)

    return {
        statusCode: 200, 
        body: JSON.stringify({
            token
        })
    }
}
```

Note: We're not validating, and key is right there in code. Obviously, this is for simplicity. 

* Add the endpoint to serverless.yaml

```
  tokens_generate:
    handler: views/tokens.generate
    events:
      - http:
          path: generate-token
          method: POST
```

* Deploy, and test token generation.

* Add another endpoint to authorise:

```
const generatePolicy = (token, methodArn) => {

    var statement = {
        "Action": "execute-api:Invoke",
        "Effect": "Deny",
        "Resource": methodArn,
    }

    try {
        console.log(`token is ${token}`)
        const decoded = jwt.verify(token, SECRET)

        if (!decoded) {
            throw new Error("GANDALF!!!!")
        }

        console.log(decoded)

        if(decoded.place != "london") {
            throw new Exception("Only in London!!!!")
        }

        statement.Effect = "Allow"
    }
    catch (err) {
    }

    //do some more useful validation here.

    const policyDocument = {
        "Version": "2012-10-17",
        "Statement": [statement]
    }

    const authResponse = {
        "principalId": "user",
        "policyDocument": policyDocument
    }


    return authResponse
}


module.exports.authorise = async (event, context) => {
    console.log(event)
    return generatePolicy(event.authorizationToken, event.methodArn)
}

```

* Add it to serverless.yaml, and mark it as the authorizer for nodes_list:


```
  tokens_generate:
    handler: views/tokens.generate
    events:
      - http:
          path: generate-token
          method: POST
  
  tokens_authorise:
    handler: views/tokens.authorise
```

```
  notes_list:
    handler: views/notes.index
    events:
      - http:
          path: notes
          method: GET
          authorizer: tokens_authorise
```


==========================
# Blow Up
==========================

sls remove
