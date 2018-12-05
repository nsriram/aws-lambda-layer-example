# Lambda Layers for NodeJS - An Example

This article outlines the steps involved in building a node js lambda using lambda layers for library dependencies, using [AWS CLI](https://github.com/aws/aws-cli).

The example will build a lambda function that will return current time using [momentjs](https://github.com/moment/moment/) library. The lambda will not bundle [momentjs](https://github.com/moment/moment/) via `package.json`, `node_modules`, but will use [momentjs](https://github.com/moment/moment/) via lambda layers.

Following are assumed to be available on your computer.
1. [AWS](https://aws.amazon.com) Account
2. [IAM Role](https://docs.aws.amazon.com/lambda/latest/dg/intro-permission-model.html) to manage and execute lambda functions
3. [AWS CLI](https://github.com/aws/aws-cli) version **1.16.69**

## 1 : Create and publish momentjs lambda layer.

What is a lambda layer ?
**(Source: AWS Docs)** : A layer is a ZIP archive that contains libraries, a custom runtime, or other dependencies. With layers, you can use libraries in your function without needing to include them in your deployment package. 

### 1.1 Create an empty nodejs project.
```
> cd ~
> mkdir momentjs-lambda-layer
> cd momentjs-lambda-layer
> npm init -y
```

### 1.2 Add momentjs package dependency
```
> yarn add moment
```

### 1.3 Package momentjs and its node_module dependencies. 

* The packaging should use the folder structure shown below for nodejs lambda layers. This folder structure is a requirement from lambda layers.

```
momentjs-lambda-layer.zip
└ nodejs/node_modules/moment
```

* Move the `node_modules` into nodejs subdirectory and package them as a zip archive file.
**_Note_**: The packaged zip file is moved to `/tmp` folder.

```
> mkdir -p dist/nodejs
> cp package.json dist/nodejs
> cd dist/nodejs
> yarn --production
> cd ..
> zip -r /tmp/momentjs-lambda-layer.zip nodejs
```
### 1.4 Publish the momentjs as a lambda layer.
**_Note_** : Your [AWS CLI](https://github.com/aws/aws-cli) is assumed to be setup. You can execute `aws lambda list-layers` to get a response listing the layers.
e.g., response
```
{ "Layers": [] }
```

Publish the lambda layer under MIT license. We are not using S3, but directly pushing the zip file that was created earlier.

```
> aws lambda publish-layer-version --layer-name "momentjs-lambda-layer" \
  --description "NodeJS module of Moment JS library" \
  --license-info "MIT" \
  --compatible-runtimes "nodejs8.10" \
  --zip-file "fileb:////tmp/momentjs-lambda-layer.zip"
```

If the publishing was successful, you should see a response similar to the one below, returning the `LayerVersionArn`.
**_Note_** :
- You can publish as many times a layer and they will be automatically published to the next version.

```
{
    "LayerVersionArn": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:layer:momentjs-lambda-layer:1",
    "Description": "NodeJS module of Moment JS library",
    "CreatedDate": "2018-12-05T10:31:36.211+0000",
    "LayerArn": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:layer:momentjs-lambda-layer",
    "Content": {
        "CodeSize": 743941,
        "CodeSha256": "nakCb8UDFbyzPrbcCde4xRK9K5EZ5MiPZxW49kl3AGI=",
        "Location": "https://awslambda-ap-se-1-layers.s3.<AWS_REGION>.amazonaws.com/snapshots/<AWS_ACCOUNT_ID>/momentjs-lambda-layer-..."
    },
    "Version": 1,
    "CompatibleRuntimes": [
        "nodejs8.10"
    ],
    "LicenseInfo": "MIT"
}
```

## 2 : Create the 'CurrentTime' lambda.

### 2.1 Create an empty nodejs project.
**_Note_**: This example intentionally avoids ES6 `import`.

```
> cd ~
> mkdir current-time-lambda
> cd current-time-lambda
> npm init -y
```

Copy the simple lambda code, that returns the current time using momentjs library, into `currentTimeLambda.js` file.

```
> echo "const moment = require('moment');

exports.handler = (event, context, callback) => {
  const time = moment().format('MMMM Do YYYY, h:mm:ss a');
  callback(null, { time });
};" > currentTimeLambda.js
```

**_Note_** : 
- The lambda source has reference to moment.js, but moment.js is not included in package.json. 
- We will attach the lambda layer that was created earlier to this lambda in following steps.

### 2.2 Package the lambda source

Here, we only need the single js file containing the lambda function. The essential purpose of lambda layers - simplify the bundling of lambda and separate the node_modules (dependencies) concern.

```
> zip -r /tmp/currentTimeLambda.js.zip currentTimeLambda.js
```

### 2.3 Deploy the lambda
**_Note_** :
- Region, IAM role and AWS profile are specific to the user.
- `AWS_ROLE_FOR_LAMBDA` is used as the IAM role that has lambda management permissions.
- `AWS_PROFILE` used here is the value configured in `~/.aws/config` file, while setting up AWS CLI.
- The packaged file `currentTimeLambda.js.zip` is directly uploaded and S3 is not used.

```
aws lambda create-function \
    --region <AWS_REGION> \
    --function-name currentTimeLambda \
    --handler 'currentTimeLambda.handler' \
    --runtime nodejs8.10 \
    --role "arn:aws:iam::<AWS_ACCOUNT_ID>:role/AWS_ROLE_FOR_LAMBDA" \
    --zip-file 'fileb:///tmp/currentTimeLambda.js.zip' \
    --profile <AWS_PROFILE>
```

On successful creation of lambda, you should see a response similar to the one below.

```
{
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "CodeSha256": "9Bi4aWpD8XRFI7udqYo1NTy0WlYAkUVXw6iQKK0NYBo=",
    "FunctionName": "currentTimeLambda",
    "CodeSize": 14236,
    "RevisionId": "3e64f4fb-c146-47fd-bfb0-d3a14486b001",
    "MemorySize": 128,
    "FunctionArn": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:function:currentTimeLambda",
    "Version": "$LATEST",
    "Role": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/AWS_ROLE_FOR_LAMBDA",
    "Timeout": 3,
    "LastModified": "2018-12-05T10:48:10.171+0000",
    "Handler": "currentTimeLambda.handler",
    "Runtime": "nodejs8.10",
    "Description": ""
}
```
_Note_ : If you try executing the lambda function, it will throw error mentioning `momentjs` library missing.

## 3 : Update lambda to refer to lambda layer.

### 3.1. Attach the lambda layer to the lambda function. 

* For this we need the `LayerVersionArn` that was returned by AWS when the lambda layer was created. Refer to step 1.4 above.
* Layers and lambda runtimes compatibility should be checked when this association is done.

```
> aws lambda update-function-configuration \
  --function-name currentTimeLambda \
  --layers arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:layer:momentjs-lambda-layer:1
```

If the above command of lambda layer association was successful, you should see the layer listed in the lambda details as below.

```
{
    "Layers": [
        {
            "CodeSize": 743941,
            "Arn": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:layer:momentjs-lambda-layer:1"
        }
    ],
    "FunctionName": "currentTimeLambda",
    "LastModified": "2018-12-05T10:51:28.445+0000",
    "RevisionId": "2ca09f6f-e85a-4cbd-a5ed-4f52db28bb07",
    "MemorySize": 128,
    "Version": "$LATEST",
    "Role": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/AWS_ROLE_FOR_LAMBDA",
    "Timeout": 3,
    "Runtime": "nodejs8.10",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "CodeSha256": "9Bi4aWpD8XRFI7udqYo1NTy0WlYAkUVXw6iQKK0NYBo=",
    "Description": "",
    "CodeSize": 14236,
    "FunctionArn": "arn:aws:lambda:<AWS_REGION>:<AWS_ACCOUNT_ID>:function:currentTimeLambda",
    "Handler": "currentTimeLambda.handler"
}
```

## 4 : Invoke lambda to validate the usage of layer

### 4.1 Invoke the lambda
* Now, we will invoke the lambda (that was deployed without momentjs node_module) to verify that it uses lambda layer for momentjs dependency.

```
>  aws lambda invoke --function-name currentTimeLambda --log-type Tail --payload '{}' outputfile.txt
```

Successful invocation should list a response similar to the following.

```
{
    "LogResult": "U1RBUlQgUmVxdWVzdElkOiAwZ...HVyYXRpb246IDEwMCBtcyAJTWVtb3J5IFNpemU6IDEyOCBNQglNYXggTWVtb3J...E1CCQo=",
    "ExecutedVersion": "$LATEST",
    "StatusCode": 200
}
```
### 4.1 View the output
- To ensure the current time is responded by lambda, view the contents of `output.txt` file. You should see the time.

```
> cat outputfile.txt
{"time":"December 5th 2018, 10:53:55 am"}%
```

🏁 If you have seen the timestamp, **Congrats !** You got your first  lambda layer working  🏁



