<head>
<style>
img {
  width: 300
}
</style>
</head>

# Custom Authorizers

## Prerequisites

* An AWS Account

On the developer machine:
* [AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html)
* [ASW CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

For the client:
* Node.js 10 or later
* jq (`sudo apt-get install jq`)

## Solution 
> **DISCLAIMER**: This solution is intended for demo purposes only and should not be used as is in a production environment without further works on the code quality and security.


## Deploy the backend via CDK

The custom authorizer logic is deployed via the CDK.
You can examine the definition of the resources that are going to be created in the `lib/ruuvi_poc-stack.ts` file.

Run the following commands to download all the project dependencies and compile the stack:

```
npm install
npm run build
```

Finally, you can deploy it with:

```
cdk deploy
```

**NOTE**: if this is the first time you use CDK on this AWS account and region the deploy will fail. Read the instructions printed on screen on how to bootstrap the account to be able to use the CDK tool.


The above commands will print out few output lines, with the custom authorizer lamnda arn, and the api gateway endpoint. Please note these down as they will be needed later.

![](images/README.md-2020-02-11-15-01-38.png)


 
### Create the signing key pair

The custom authorizer validated that the token that is provided is signed with a known key. This prevents malicious users to trigger you custom authorizer lambda function as AWS IoT Core will deny access if the token and the token signature do not match.

The token signature is generated using an RSA key. The private key is used by the client to sign the authorization token while the the public key will be associated with the custom authorizer.

To create the key pair follow these steps:

```bash
openssl genrsa -out myPrivateKey.pem 2048
openssl rsa -in myPrivateKey.pem -pubout > mykey.pub
```

The file `mykey.pub` will contain the public key in PEM format that you will need to configure for the authorizer in the next step.

##  Custom authorizer configuration

In this step we are going to configure the custom authorizer in AWS IoT Core. You can find more information about custom authorizers in the [documentation](https://docs.aws.amazon.com/iot/latest/developerguide/custom-authorizer.html).

### CLI

We first create the authorizer, giving it a name and associating it with the lambda function that performs the authorization. This lambda fucntion has been created in the previous step. You can examine the code in `lambda/iot-custom-auth/lambda.js`.

```bash
arn=<lambda arn from CDK>

resp=$(aws iot create-authorizer \
  --authorizer-name "TokenAuthorizer" \
  --authorizer-function-arn $arn \
  --status ACTIVE )
auth_arn=$(echo $resp | jq ".authorizerArn" -)
```

Take note of the arn of the token authorizer, we need it to add give the iot service the permission to invoke this lambda function on your behalf when a new connection request is made.

```bash
aws lambda add-permission \
  --function-name  $arn \
  --principal iot.amazonaws.com \
  --statement-id Id-123 \
  --action "lambda:InvokeFunction" \
  --source-arn $auth_arn \
```


#### Test the authorizer

To test that the authorizer works fine, you can also use the `test/authTest.js` client.

```
node test/authTest.js --key_path <key path> --endpoint <endpoint> --id <id> [--verbose]
```

The test app will publish message to a topic `d/<id>` every 5 sec. Use the iot console.

#### If you get an error

To test if the authorizer is setup correctly you can also use the aws cli.

```bash
aws iot test-invoke-authorizer \
  --authorizer-name TokenAuthorizer \
  --token <token> --token-signature <signature>
```

Use the `--verbose` mode in the authTest.js call to get the token and singature and pass those to the above command.

## How to use the tokens

In this example the client is responsible of signing the token which is obviously not secure, as the client could craft his own proviledges or impersonate another device.

The token and its signature should therefore be generated in the backend, and possibly also encrypted. 

Rotations of the token can be implemented via the MQTT protocol, and the only issue to solve would be how to obtain the initial token to the device. This could be done via an external API, a companion app, a registration step, etc. and is out of the scope of this demo.

