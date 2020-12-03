# Serverless WarmUp Plugin ♨
[![Serverless][serverless-badge]](serverless-badge-url)
[![npm version][npm-version-badge]][npm-version-badge-url]
[![npm monthly downloads][npm-downloads-badge]][npm-version-badge-url]
[![Build Status][travis-badge]][travis-badge-url]
[![Coverage Status][coveralls-badge]][coveralls-badge-url]
[![Dependency Status][dev-badge]][dev-badge-url]
[![license](https://img.shields.io/npm/l/serverless-plugin-warmup.svg)](https://raw.githubusercontent.com/FidelLimited/serverless-plugin-warmup/master/LICENSE)

Keep your lambdas warm during winter.

**Requirements:**
* Serverless *v1.12.x* or higher (Recommended *v1.33.x* or higher because of [this](https://github.com/FidelLimited/serverless-plugin-warmup/pull/69)).
* AWS provider

## How it works

WarmUp solves *cold starts* by creating a scheduled lambda (the warmer) that invokes all the selected service's lambdas in a configured time interval (default: 5 minutes) and forcing your containers to stay warm.

## Installation

Install via npm in the root of your Serverless service:

```sh
npm install --save-dev serverless-plugin-warmup
```

Add the plugin to the `plugins` array in your Serverless `serverless.yaml`:

```yaml
plugins:
  - serverless-plugin-warmup
```

## Configuration

The warmup plugin supports creating one or more warmer functions. Warmers are defined under `custom.warmup` in the `serverless.yaml` file:

```yaml
custom:
  warmup:
    officeHoursWarmer:
      enabled: true
      events:
        - schedule: cron(0/5 8-17 ? * MON-FRI *)
      concurrency: 10
    outOfOfficeHoursWarmer:
      enabled: true
      events:
        - schedule: cron(0/5 0-7 ? * MON-FRI *)
        - schedule: cron(0/5 18-23 ? * MON-FRI *)
        - schedule: cron(0/5 * ? * SAT-SUN *)
      concurrency: 1
    testWarmer:
      enabled: false
```

The options are the same for all the warmers:

* **folderName** Folder to temporarily store the generated code (defaults to `.warmup`)
* **cleanFolder** Whether to automatically delete the generated code folder. You might want to keep it if you are doing some custom packaging (defaults to `true`)
* **name** Name of the generated warmer lambda (defaults to `${service}-${stage}-warmup-plugin-${warmerName}`)
* **role** Role to apply to the warmer lambda (defaults to the role in the provider)
* **tags** Tag to apply to the generated warmer lambda (defaults to the serverless default tags)
* **vpc** The VPC and subnets in which to deploy. Can be any [Serverless VPC configuration](https://serverless.com/framework/docs/providers/aws/guide/functions#vpc-configuration) or be set to `false` in order to deploy the warmup function outside of a VPC (defaults to the vpc in the provider)
* **memorySize** The memory to be assigned to the warmer lambda (defaults to `128`)
* **events** The event that triggers the warmer lambda. Can be any [Serverless event](https://serverless.com/framework/docs/providers/aws/events/) (defaults to `- schedule: rate(5 minutes)`)
* **package** The package configuration. Can be any [Serverless package configuration](https://serverless.com/framework/docs/providers/aws/guide/packaging#package-configuration) (defaults to `{ individually: true, exclude: ['**'], include: ['.warmup/${warmerName}/**'] }`)
* **timeout** How many seconds until the warmer lambda times out. (defaults to `10`)
* **environment** Can be used to set environment variables in the warmer lambda. You can also unset variables configured at the provider by setting them to undefined. However, you should almost never have to change the default. (defaults to unset all package level environment variables. )
* **prewarm** If set to true, it warms up your lambdas right after deploying (defaults to `false`)

There are also some options which can be set under `custom.warmup.<yourWarmer>` to be applied to all your lambdas or under `yourLambda.warmup.<yourWarmer>` to  overridde the global configuration for that particular lambda. Keep in mind that in order to configure a warmer at the function level, it needed to be previously configured at the `custom` section or the pluging will error.

* **enabled** Whether your lambda should be warmed up or not. Can be a boolean, a stage for which the lambda will be warmed up or a list of stages for which your lambda will be warmed up (defaults to `false`)
* **clientContext** Custom data to send as client context to the data. It should be an object where all the values are strings. (defaults to the payload. Set it to `false` to avoid sending any client context custom data)
* **payload** The payload to send to your lambda. This helps your lambda identify when the call comes from this plugin (defaults to `{ "source": "serverless-plugin-warmup" }`)
* **payloadRaw** Whether to leave the payload as-is. If false, the payload will be stringified into JSON. (defaults to `false`)
* **concurrency** The number of times that each of your lambda functions will be called in parallel. This can be used in a best-effort attempt to force AWS to spin up more parallel containers for your lambda. (defaults to `1`)

```yaml
custom:
  warmup:
    default:
      enabled: true # Whether to warm up functions by default or not
      folderName: '.warmup' # Name of the folder created for the generated warmup 
      cleanFolder: false
      memorySize: 256
      name: warmer-default
      role: WarmupRole
      tags:
        Project: foo
        Owner: bar 
      vpc: false
      events:
        - schedule: 'cron(0/5 8-17 ? * MON-FRI *)' # Run WarmUp every 5 minutes Mon-Fri between 8:00am and 5:55pm (UTC)
      package:
        individually: true
        exclude: # exclude additional binaries that are included at the serverless package level
          - ../**
          - ../../**
        include:
          - ./**
      timeout: 20
      prewarm: true # Run WarmUp immediately after a deploymentlambda
      clientContext:
        source: my-custom-source
        other: '20'
      payload: 
        source: my-custom-source
        other: 20
      payloadRaw: true # Won't JSON.stringify() the payload, may be necessary for Go/AppSync deployments
      concurrency: 5 # Warm up 5 concurrent instances
    
functions:
  myColdfunction:
    handler: 'myColdfunction.handler'
    events:
      - http:
          path: my-cold-function
          method: post
    warmup:
      default:
        enabled: false

  myLowConcurrencyFunction:
    handler: 'myLowConcurrencyFunction.handler'
    events:
      - http:
          path: my-low-concurrency-function
          method: post
    warmup:
      default:
        clientContext:
          source: different-source-only-for-this-lambda
        payload:
          source: different-source-only-for-this-lambda
        concurrency: 1
   
  myProductionOnlyFunction:
    handler: 'myProductionOnlyFunction.handler'
    events:
      - http:
          path: my-production-only-function
          method: post
    warmup:
      default:
        enabled: prod
      
   myDevAndStagingOnlyFunction:
    handler: 'myDevAndStagingOnlyFunction.handler'
    events:
      - http:
          path: my-dev-and-staging-only-function
          method: post
    warmup:
      default:
        enabled:
          - dev
          - staging
```

##### Options should be tweaked depending on:

* Number of lambdas to warm up
* Day cold periods
* Desire to avoid cold lambdas after a deployment

#### Runtime Configuration

Concurrency can be modified post-deployment at runtime by setting the warmer lambda environment variables.  
Two configuration options exist:
* Globally set the concurrency for all lambdas on the stack (overriding the deployment-time configuration):  
  Set the environment variable `WARMUP_CONCURRENCY`
* Individually set the concurrency per lambda  
  Set the environment variable `WARMUP_CONCURRENCY_YOUR_FUNCTION_NAME`. Must be all uppercase and hyphens (-) must be replaced with underscores (_). If present for one of your lambdas, it overrides the global concurrency setting.

### Permissions

WarmUp requires permission to be able to `invoke` your lambdas.

If no role is provided at the `custom.warmup` level, each warmer function gets a default role with minimal permissions allowing the warmer function to:
* Create its log stream and write logs to it
* Invoke the functions that it should warm (and only those)
* Create and attach elastic network interfaces (ENIs) which is necessary if deploying to a VPC

The default role looks like:

```yaml
resources:
  Resources:
    WarmupRole:
      Type: AWS::IAM::Role
      Properties:
        RoleName: WarmupRole
        AssumeRolePolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - lambda.amazonaws.com
              Action: sts:AssumeRole
        Policies:
          - PolicyName: WarmUpLambdaPolicy
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
               # Warmer lambda to send logs to CloudWatch
                - Effect: Allow
                  Action:
                    - logs:CreateLogGroup
                    - logs:CreateLogStream
                  Resource: 
                    - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/${self:service}-${opt:stage, self:provider.stage}/*:*
                - Effect: Allow
                  Action:
                    - logs:PutLogEvents
                  Resource: 
                    - !Sub arn:aws:logs:${AWS::Region}:${AWS::AccountId}:log-group:/aws/${self:service}-${opt:stage, self:provider.stage}/*:*:*
                # Warmer lambda to invoke the functions to be warmed
                - Effect: 'Allow'
                  Action:
                    - lambda:InvokeFunction
                  Resource:
                    - !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${self:service}-${opt:stage, self:provider.stage}-*
                # Warmer lambda to manage ENIS (only needed if deploying to VPC, https://docs.aws.amazon.com/lambda/latest/dg/vpc.html)
                - Effect: Allow
                  Action:
                    - ec2:CreateNetworkInterface
                    - ec2:DescribeNetworkInterfaces
                    - ec2:DetachNetworkInterface
                    - ec2:DeleteNetworkInterface
                  Resource: "*"
```

The permissions can also be added to all lambdas using setting the role to `IamRoleLambdaExecution` and setting the permissions in `iamRoleStatements` under `provider` (see https://serverless.com/framework/docs/providers/aws/guide/functions/#permissions):

```yaml
provider:
  name: aws
  runtime: nodejs10.x
  iamRoleStatements:
    - Effect: 'Allow'
      Action:
        - 'lambda:InvokeFunction'
      Resource:
      - !Sub arn:aws:lambda:${AWS::Region}:${AWS::AccountId}:function:${self:service}-${opt:stage, self:provider.stage}-*
custom:
  warmup:
    default:
      enabled: true
      role: IamRoleLambdaExecution
```

If setting `prewarm` to `true`, the deployment user used by the AWS CLI and the Serverless framework also needs permissions to invoke the warmer.

## On the function side

When invoked by WarmUp, your lambdas will have the event source `serverless-plugin-warmup` (unless otherwise specified using the `payload` option):

```json
{
  "Event": {
    "source": "serverless-plugin-warmup"
  }
}
```

To minimize cost and avoid running your lambda unnecessarily, you should add an early return call before your lambda logic when that payload is received.

### Javascript

Using the Promise style:

```js
module.exports.lambdaToWarm = async function(event, context) {
  /** Immediate response for WarmUp plugin */
  if (event.source === 'serverless-plugin-warmup') {
    console.log('WarmUp - Lambda is warm!');
    return 'Lambda is warm!';
  }

  ... add lambda logic after
}
```

Using the Callback style:

```js
module.exports.lambdaToWarm = function(event, context, callback) {
  /** Immediate response for WarmUp plugin */
  if (event.source === 'serverless-plugin-warmup') {
    console.log('WarmUp - Lambda is warm!')
    return callback(null, 'Lambda is warm!')
  }

  ... add lambda logic after
}
```

Using the context. This could be useful if you are handling the raw input and output streams.

```js
module.exports.lambdaToWarm = async function(event, context) {
  /** Immediate response for WarmUp plugin */
  if (context.custom.source === 'serverless-plugin-warmup') {
    console.log('WarmUp - Lambda is warm!');
    return 'Lambda is warm!';
  }

  ... add lambda logic after
}
```

If you're using the `concurrency` option you might want to add a slight delay before returning on warmup calls to ensure that your function doesn't return before all concurrent requests have been started:

```jss
module.exports.lambdaToWarm = async (event, context) => {
  if (event.source === 'serverless-plugin-warmup') {
    console.log('WarmUp - Lambda is warm!');
    /** Slightly delayed (25ms) response 
    	to ensure concurrent invocation */
    await new Promise(r => setTimeout(r, 25));
    return 'Lambda is warm!';
    
  }

  ... add lambda logic after
}
```

### Python

You can handle it in your function:

```python
def lambda_handler(event, context):
    # early return call when the function is called by warmup plugin
    if event.get("source") in ["aws.events", "serverless-plugin-warmup"]:
        print('Lambda is warm!')
        return {}

    # function logic here
    ...
```

Or you could use a decorator to avoid the redundant logic in all your functions:

```python
def skip_execution_if.warmup_call(func):
    def warmup_wrapper(event, context):
      if event.get("source") in ["aws.events", "serverless-plugin-warmup"]:
        print("Lambda is warm!")
        return {}

      return func(event, context)

    return warmup_wrapper

# ...

@skip_execution_if.warmup_call
def lambda_handler(event, context):
    # function logic here
    ...
```

## Deployment

WarmUp supports `serverless deploy`.

## Packaging

WarmUp supports `serverless package`.

By default, the WarmUp function is packaged individually and it uses a folder named `.warmup` to store duiring the packaging process, which is deleted at the end of the process.

If you are doing your own [package artifact](https://serverless.com/framework/docs/providers/aws/guide/packaging#artifact) you can set the `cleanFolder` option to `false` and include the `.warmup` folder in your custom artifact.

## Gotchas

The WarmUp function use normal calls to the AWS SDK in order to keep your lambdas warm.
By deafult, the WarmUp function is deployed outside of any VPC so it can reach AWS API.
If you use the VPC option to deploy your WarmUp function to a VPC subnet it will need internet access. You can do it by using an [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html) or a [Network Address Translation (NAT) gateway](http://docs.aws.amazon.com/lambda/latest/dg/vpc.html). 

## Migrations

### v4.X to v5.X

#### Support multiple warmer

Previous versions of the plugin only support a single warmer which limited use cases like having different concurrentcies in different time periods. From v5, multiple warmers are supported. The `warmup` field in the `custom` section or the function section, takes an object where each key represent the name of the warmer and the value the configuration which is exactly as it used to be except for the changes listed below.

```yaml
custom:
  warmup:
    enabled: true
    events:
      - schedule: rate(5 minutes)
```

have to be named, for example, to `default`:

```yaml
custom:
  warmup:
    default:
      enabled: true
      events:
        - schedule: rate(5 minutes)
```

#### Change the default temporary folder to `.warmup`

Previous versions of the plugin named the temporary folder to create the warmer handler `_warmup`. It has been renamed to `.warmup` to better align with the serverless framework and other plugins' behaviours.

Remembe to add `.warmup` to your git ignore.

#### Default to Unqualified alias

Previous versions of the plugin used the `$LATEST` alias as default alias to warm up if no alias was provided. From v5, the unqualified alias is the default. You can still use the `$LATEST` alias by setting it using the `SERVERLESS_ALIAS` environment variable.

```yaml
custom:
  warmup:
    default:
      environment:
        SERVERLESS_ALIAS: $LATEST
```

#### Automatically exclude package level includes

Previous versions of the plugin exclude everything in the service folder and include the `.warmup` folder. This caused that any files that you include to the service level were also included in the plugin specially if you include ancestor folders (like `../**`)
From v5, all service level include are automatically excluded from the plugin. You still override this behaviour using the `package` option.

#### Removed shorthand

Previous versions of the plugin support replacing the configuration by a boolean, a string representing a stage or an array of strings representing a lsit of stages. From v5, this is not supported anymore. The `enabled` option is equivalent.

```yaml
custom:
  warmup: 'prod'
```

is the same as
```yaml
custom:
  warmup:
    default: # Name of the warmer, see above
      enabed: 'prod'
```

#### Removed legacy options

The following legacy options have been completely removed:

* **default** Has been renamed to `enabled`
* **schedule** `schedule: rate(5 minutes)` is equivalent to `events: - schedule: rate(5 minutes)`.
* **source** Has been renamed to `payload`
* **sourceRaw** Has been renamed to `payloadRaw`

#### Automatically creates a role for the lambda

If no role is provided in the `custom.warmup` config, a default role with minimal permissions is created for each warmer.

## Cost

You can check the Lambda [pricing](https://aws.amazon.com/lambda/pricing/) and CloudWatch [pricing](https://aws.amazon.com/cloudwatch/pricing/) or can use the [AWS Lambda Pricing Calculator](https://s3.amazonaws.com/lambda-tools/pricing-calculator.html) to estimate the monthly cost

#### Example

If you have a single warmer and want to warm 10 functions, each with `memorySize = 1024` and `duration = 10`, using the default settings (and we ignore the free tier):

* WarmUp: runs 8640 times per month = $0.18
* 10 warm lambdas: each invoked 8640 times per month = $14.4
* Total = $14.58

CloudWatch costs are not in this example because they are very low.

## Contribute

Help us making this plugin better and future-proof.

* Clone the code
* Install the dependencies with `npm install`
* Create a feature branch `git checkout -b new_feature`
* Add your code and add tests if you implement a new feature
* Validate your changes `npm run lint` and `npm test` (or `npm run test-with-coverage`)

## License

This software is released under the MIT license. See [the license file](LICENSE) for more details.

[serverless-badge]: http://public.serverless.com/badges/v3.svg
[serverless-badge-url]: http://www.serverless.com
[npm-version-badge]: https://badge.fury.io/js/serverless-plugin-warmup.svg
[npm-version-badge-url]: https://www.npmjs.com/package/serverless-plugin-warmup
[npm-downloads-badge]: https://img.shields.io/npm/dm/serverless-plugin-warmup.svg
[travis-badge]: https://travis-ci.org/FidelLimited/serverless-plugin-warmup.svg
[travis-badge-url]: https://travis-ci.org/FidelLimited/serverless-plugin-warmup
[coveralls-badge]: https://coveralls.io/repos/FidelLimited/serverless-plugin-warmup/badge.svg?branch=master
[coveralls-badge-url]: https://coveralls.io/r/FidelLimited/serverless-plugin-warmup?branch=master
[dev-badge]: https://david-dm.org/FidelLimited/serverless-plugin-warmup.svg
[dev-badge-url]: https://david-dm.org/FidelLimited/serverless-plugin-warmup
