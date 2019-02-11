<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

# Apache OpenWhisk Composer

[![Travis](https://travis-ci.org/apache/incubator-openwhisk-composer.svg?branch=master)](https://travis-ci.org/apache/incubator-openwhisk-composer)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Join
Slack](https://img.shields.io/badge/join-slack-9B69A0.svg)](http://slack.openwhisk.org/)

Composer is a new programming model for composing cloud functions built on
[Apache OpenWhisk](https://github.com/apache/incubator-openwhisk). With
Composer, developers can build even more serverless applications including using
it for IoT, with workflow orchestration, conversation services, and devops
automation, to name a few examples.

Composer synthesizes OpenWhisk [conductor
actions](https://github.com/apache/incubator-openwhisk/blob/master/docs/conductors.md)
to implement compositions. Compositions have all the attributes and capabilities
of an action, e.g., default parameters, limits, blocking invocation, web export.

This repository includes:
* the [composer](composer.js) Node.js module for authoring compositions using
  JavaScript,
* the [compose](bin/compose.js) and [deploy](bin/deploy.js)
  [commands](docs/COMMANDS.md) for compiling and deploying compositions,
* [documentation](docs), [examples](samples), and [tests](test).

## Installation

Composer is distributed as Node.js package. To install this package, use the
Node Package Manager:
```
npm install -g openwhisk-composer
```
We recommend to install the package globally (with `-g` option) if you intend to
use the `compose` and `deploy` commands to compile and deploy compositions.

## Defining a composition

A composition is typically defined by means of a Javascript expression as
illustrated in [samples/demo.js](samples/demo.js):
```javascript
const composer = require('openwhisk-composer')

module.exports = composer.if(
    composer.action('authenticate', { action: function ({ password }) { return { value: password === 'abc123' } } }),
    composer.action('success', { action: function () { return { message: 'success' } } }),
    composer.action('failure', { action: function () { return { message: 'failure' } } }))
```
Compositions compose actions using [combinator](docs/COMBINATORS.md) methods.
These methods implement the typical control-flow constructs of an imperative
programming language. This example composition composes three actions named
`authenticate`, `success`, and `failure` using the `composer.if` combinator,
which implements the usual conditional construct. It takes three actions (or
compositions) as parameters. It invokes the first one and, depending on the
result of this invocation, invokes either the second or third action.

 This composition includes the definitions of the three composed actions. If the
 actions are defined and deployed elsewhere, the composition code can be shorten
 to:
```javascript
composer.if('authenticate', 'success', 'failure')
```

## Deploying a composition

One way to deploy a composition is to use the `compose` and `deploy` commands:
```
compose demo.js > demo.json
deploy demo demo.json -w
```
```
ok: created /_/authenticate,/_/success,/_/failure,/_/demo
```
The `compose` command compiles the composition code to a portable JSON format.
The `deploy` command deploys the JSON-encoded composition creating an action
with the given name. It also deploys the composed actions if definitions are
provided for them. The `-w` option authorizes the `deploy` command to overwrite
existing definitions.

## Running a composition

The `demo` composition may be invoked like any action, for instance using the
OpenWhisk CLI:
```
wsk action invoke demo -p password passw0rd
```
```
ok: invoked /_/demo with id 4f91f9ed0d874aaa91f9ed0d87baaa07
```
The result of this invocation is the result of the last action in the
composition, in this case the `failure` action since the password in incorrect:
```
wsk activation result 4f91f9ed0d874aaa91f9ed0d87baaa07
```
```json
{
    "message": "failure"
}
```
### Execution traces

This invocation creates a trace, i.e., a series of activation records:
```
wsk activation list
```
```
activations
fd89b99a90a1462a89b99a90a1d62a8e demo
eaec119273d94087ac119273d90087d0 failure
3624ad829d4044afa4ad829d40e4af60 demo
a1f58ade9b1e4c26b58ade9b1e4c2614 authenticate
3624ad829d4044afa4ad829d40e4af60 demo
4f91f9ed0d874aaa91f9ed0d87baaa07 demo
```
The entry with the earliest start time (`4f91f9ed0d874aaa91f9ed0d87baaa07`)
summarizes the invocation of the composition while other entries record later
activations caused by the composition invocation. There is one entry for each
invocation of a composed action (`a1f58ade9b1e4c26b58ade9b1e4c2614` and
`eaec119273d94087ac119273d90087d0`). The remaining entries record the beginning
and end of the composition as well as the transitions between the composed
actions.

Compositions are implemented by means of OpenWhisk conductor actions. The
[documentation of conductor
actions](https://github.com/apache/incubator-openwhisk/blob/master/docs/conductors.md)
explains execution traces in greater details.

While composer does not limit in principle the length of a composition,
OpenWhisk deployments typically enforce a limit on the number of action
invocations in a composition as well as an upper bound on the rate of
invocation. These limits may result in compositions failing to execute to
completion.

## Parallel compositions with Redis

Composer offers parallel combinators that make it possible to run actions or
compositions in parallel, for example:
```javascript
composer.parallel('checkInventory', 'detectFraud')
```

The width of parallel compositions is not in principle limited by composer, but
issuing many concurrent invocations may hit OpenWhisk limits leading to
failures: failure to execute a branch of a parallel composition or failure to
complete the parallel composition.

These combinators require access to a Redis instance to hold intermediate
results of parallel compositions. The Redis credentials may be specified at
invocation time or earlier by means of default parameters or package bindings.
The required parameter is named `$composer`. It is a dictionary with a `redis`
field of type dictionary. The `redis` dictionary specifies the `uri` for the
Redis instance and optionally a certificate as a base64-encoded string to enable
TLS connections. Hence, the input parameter object for our order-processing
example should be:
```json
{
    "$composer": {
        "redis": {
            "uri": "redis://...",
            "ca": "optional base64 encoded tls certificate"
        }
    },
    "order": { ... }
}
```

The intent is to store intermediate results in Redis as the parallel composition
is progressing. Redis entries are deleted after completion and, as an added
safety, expire after twenty-four hours.

# Disclaimer

Apache OpenWhisk Composer is an effort undergoing incubation at The Apache Software Foundation (ASF), sponsored by the Apache Incubator. Incubation is required of all newly accepted projects until a further review indicates that the infrastructure, communications, and decision making process have stabilized in a manner consistent with other successful ASF projects. While incubation status is not necessarily a reflection of the completeness or stability of the code, it does indicate that the project has yet to be fully endorsed by the ASF.
