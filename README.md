# Custom Authorizers

## Prerequisites

* An AWS Account

On the developer machine:
* [AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html)
* [ASW CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

For the client:
* Node.js 10 or later
* jq (`sudo apt-get install jq`)

> **DISCLAIMER**: This solution is intended for demo purposes only and should not be used as is in a production environment without further works on the code quality and security.


## Deploy the backend via CDK

The custom authorizer logic is deployed via the CDK.
You can examine the definition of the resources that are going to be created in the `lib/jwt-iot-custom-authorizer-stack.ts` file.

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

The above commands will print out few output lines, with the 2 custom authorizer lamnda arn. one called `lambdaArn` and the other `lambdaArnMqtt`. Please note these down as they will be needed later.


### Create the signing key pair

The custom authorizer validated that the token that is provided is signed with a known key. This prevents malicious users to trigger you custom authorizer lambda function as AWS IoT Core will deny access if the token and the token signature do not match.

The token signature is generated using an RSA key. The private key is used by the client to sign the authorization token while the the public key will be associated with the custom authorizer.
This signature algorithm is equivalent to the RSA256 algorithm adopted by the JWT token [RFC 7518](https://tools.ietf.org/html/rfc7518#section-3). We are going to use this property to simplify the signing process.

To create the key pair follow these steps:

```bash
openssl genrsa -out myPrivateKey.pem 2048
openssl rsa -in myPrivateKey.pem -pubout > mykey.pub
```

The file `mykey.pub` will contain the public key in PEM format that you will need to configure for the authorizer in the next step.

##  Custom authorizer configuration (non MQTT)

In this step we are going to configure the custom authorizer in AWS IoT Core. You can find more information about custom authorizers in the [documentation](https://docs.aws.amazon.com/iot/latest/developerguide/custom-authorizer.html).

### CLI

We first create the authorizer, giving it a name and associating it with the lambda function that performs the authorization. This lambda fucntion has been created in the previous step. You can examine the code in `lambda/iot-custom-auth/lambda.js`.

```bash
arn=<lambdaArn from CDK>

resp=$(aws iot create-authorizer \
  --authorizer-name "TokenAuthorizer" \
  --authorizer-function-arn $arn \
  --status ACTIVE \
  --token-key-name token \
  --token-signing-public-keys KEY1="$key")
auth_arn=$(echo $resp | jq -r .authorizerArn -)
```

Take note of the arn of the token authorizer, we need it to add give the iot service the permission to invoke this lambda function on your behalf when a new connection request is made.

```bash
aws lambda add-permission \
  --function-name  $arn \
  --principal iot.amazonaws.com \
  --statement-id Id-1234 \
  --action "lambda:InvokeFunction" \
  --source-arn $auth_arn
```



### Test the authorizer

To test that the authorizer works fine, you can also use the `test/authTest.js` client. 

This code creates a JWT token as the following and signs it with RSA256 using the private key:

```json
{
  "sub": "id1234",
  "exp": 1593699087
}
```


```
node test/authTest.js --key_path <key path> --endpoint <endpoint> --id <id> [--verbose] [--authorizer]
```

where:
* **key_path** is the path to the private key PEM encoded file
* **endpoint** is the FQDN of your AWS IoT endpoint (get it via `aws iot describe-endpoint --endpoint-type iot:Data-ATS` on from the console)
* **id** is the client id, thingName
* **verbose** prints out the encoded JWT token and signature
* **authorizer** in case you need to specify another authorizer than TokenAuthorizer


The test app will publish message to a topic `d/<id>` every 5 sec. Use the iot console.

#### If you get an error

To test if the authorizer is setup correctly you can also use the aws cli.

```bash
aws iot test-invoke-authorizer \
  --authorizer-name TokenAuthorizer \
  --token <token> --token-signature <signature>
```

Use the `--verbose` mode in the authTest.js call to get the token and singature and pass those to the above command.

## Testing with [aws-iot-device-sdk-cpp-v2](https://github.com/aws/aws-iot-device-sdk-cpp-v2)

To test the custom authorizer with the CPP device SDK v2 proceed as follow:

* Clone the github repo
* Compile the code following the instructions
* execute the `samples/mqtt/raw-pub-sub` sample with the following args:
```
  --endpoint <iot endpoint> 
  --use_websocket --auth_params token=<token>,x-amz-customauthorizer-name=TokenAuthorizer,x-amz-customauthorizer-signature=<signature> --topic d/<id>
```

You can get the `token` and `signature` values from the authTest.js code running with --verbose. For `id` use the same value specified when running authTest.js.

## About the tokens and security

In this example the client is responsible of signing the token which is obviously not secure, as the client could craft his own priviledges or impersonate another device.

The token and its signature should therefore be generated in the backend, and possibly also encrypted. 

Rotations of the token can be implemented via the MQTT protocol, and the only issue to solve would be how to obtain the initial token to the device. This could be done via an external API, a companion app, a registration step, etc. and is out of the scope of this demo.


##  MQTT Custom authorizer configuration

In this step we are going to configure the custom authorizer in AWS IoT Core. You can find more information about custom authorizers in the [documentation](https://docs.aws.amazon.com/iot/latest/developerguide/custom-authorizer.html).

### CLI

We first create the authorizer, giving it a name and associating it with the lambda function that performs the authorization. This lambda fucntion has been created in the previous step. You can examine the code in `lambda/iot-custom-auth/lambda.js`.

```bash
arn=<lambdaArnMqtt arn from CDK>

resp=$(aws iot create-authorizer \
  --authorizer-name "MQTTTokenAuthorizer" \
  --authorizer-function-arn $arn \
  --status ACTIVE \

auth_arn=$(echo $resp | jq -r .authorizerArn -)
```

Take note of the arn of the token authorizer, we need it to add give the iot service the permission to invoke this lambda function on your behalf when a new connection request is made.

```bash
aws lambda add-permission \
  --function-name  $arn \
  --principal iot.amazonaws.com \
  --statement-id Id-1234 \
  --action "lambda:InvokeFunction" \
  --source-arn $auth_arn
```



### Test the authorizer

For this test we provide a Python client using the Python Aws Crt libraries.

```
pip install -r requirements.txt
python --endpoint <endpoint> --topic test/mqtt
```

Where
* **endpoint** is the FQDN of your AWS IoT endpoint (get it via `aws iot describe-endpoint --endpoint-type iot:Data-ATS` on from the console)

