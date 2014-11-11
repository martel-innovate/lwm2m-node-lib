# iotagent-lwm2m-lib

## Overview
The [Open Mobile Alliance Lightweight M2M protocol](http://openmobilealliance.org/about-oma/work-program/m2m-enablers/) is a machine to machine communication protocol built over [COAP](https://tools.ietf.org/html/draft-ietf-core-coap), and meant to
communicate resource constrained devices. The protocol defines two roles for the devices: a Lightweight M2M Client (the constrained device) and a Lightweight M2M Server (meant to consume the device data and control its execution).

This library aims to provide a simple way to build Lightweight M2M Servers and Clients with Node.js, giving an abstraction over the COAP Protocol based on function calls and handlers. The provided features are:
* Creation of a server listening to Client calls for the LWTM2M Interfaces, linked to handlers defined by the user.
* Registry of devices connected to the server (in-memory registry for the first version).
* Server calls to the registered devices in the registry (for Device Management Interface mostly) to retrieve information.

The library also provides command line clients to test both its client and server capabilities.

## Command line applications
The library provides two command line applications in order to help developing both Lightweight M2M clients and/or servers. This applications can be used to simulate the behavior of one of the peers of the communication. Both of them use the iotagent-lwm2m-lib library to serve all the LWTM2M requests. The following sections explain the basic features of each one.

To use any of the applications you need to clone the project in your computer, and execute `npm install` from the root of the project, in order to download the dependencies. You can start both applications from the same folder using different console windows.

Take into account that all the information loaded or registered by any of the applications is transient, so it will be lost once the processes have been stopped.

### Server Command Line Application
#### Description
This application simulates the use of a Lightweight M2M server. It provides commands to start and stop the server, manage and query the devices connected to the server, and perform read and write operations over the resources provided by each of the connected devices.

#### Usage
From the root of the project type (make sure the `npm install` command has been previously executed to download all the dependencies):
```
bin/iotagent-lwm2m-server.js
```
You can type `help` in the command line at any moment to get a full list of the available commands. 

All the server configuration is read from the `config.js` file in the root of the project. You can print the configuration that is actually being used using the `config` command.

To exit the command line client, use `CTRL-C`.

#### Command reference
```
start  

	Starts a new Lightweight M2M server listening in the prefconfigured port.

stop  

	Stops the current LWTM2M Server running.

list  

	List all the devices connected to the server.

write <deviceId> <resourceId> <resourceValue>  

	Writes the given value to the resource indicated by the URI (in LWTM2M format) in the givendevice.

read <deviceId> <resourceId>  

	Reads the value of the resource indicated by the URI (in LWTM2M format) in the given device.

discover <deviceId> <resourceId>  

	Sends a discover order for the given resource (defined with a LWTM2M URI) to the given device.

cancel <deviceId> <resourceId>  

	Cancel the discover order for the given resource (defined with a LWTM2M URI) to the given device.

config  

	Print the current config.
```
### Client Command Line Application
#### Description
This application simulates the use of a Lightweight M2M Client (typically a device or device hub). It provides the following features:
* Resource management: an internal resource registry lets the client create and update the objects and resources that will be exposed to the server. These resources will be affected by the `read` and `write` operations of the server. If there is an `Observe` command over any resource, it will be periodically reported to the observer. All this information is currently kept in memory, so it needs to be recreated if the client is restarted.
* Connection management: the client can connect, disconnect and update the connection to a remote Lightweight M2M Server. Each time the client is registered or its registration updated, the list of available objects is forwarded to the server.

#### Usage
From the root of the project type (make sure the `npm install` command has been previously executed to download all the dependencies):
```
bin/iotagent-lwm2m-client.js
```
You can type `help` in the command line at any moment to get a full list of the available commands. 

All the client configuration is read from the `config.js` file in the root of the project. You can print the configuration that is actually being used using the `config` command.

To exit the command line client, use `CTRL-C`.

#### Command reference
```
create <objectUri>  

	Create a new object. The object is specified using the /type/id OMA notation.

get <objectUri>  

	Get all the information on the selected object.

remove <objectUri>  

	Remove an object. The object is specified using the /type/id OMA notation.

set <objectUri> <resourceId> <resourceValue>  

	Set the value for a resource. If the resource does not exist, it is created.

unset <objectUri> <resourceId>  

	Removes a resource from the selected object.

list  

	List all the available objects along with its resource names and values.

connect <host> <port> <endpointName>  

	Connect to the server in the selected host and port, using the selected endpointName.

updateConnection  

	Updates the current connection to a server.

disconnect  

	Disconnect from the current server.

config  

	Print the current config.
```

## Usage
Note: as it is not yet published in npm repositories, this module has to be currently used as a github dependency in the package.json. To do so, add the following dependency to your package.json file, indicating the commit you want to use:

```
"iotagent-lwm2m-lib": "https://github.com/dmoranj/iotagent-lwm2m-lib/tarball/43664dd4b011673dd56d52b00d825cc3cf2ef679"
```

In order to use this library, first you must require it:
```
var lwtm2m = require('iotagent-lwm2m-lib');
```
As a Lightweight M2M Server, the library supports four groups of features, one for each direction of the communication: client-to-server and server-to-client (and each flow both for the client and the server). Each feature set is defined in the following sections.

### Server features (client -> server)

#### Starting and stopping the server
To start the LWTM2M Server execute the following command:
```
lwtm2m.start(config, function(error) {
  console.log('Listening');
});
```
The config object contains all the information required to start the server (see its structure in the Configuration section below).

Only one server can be listening at a time in the library (is treated as a singleton), so multiple calls to 'start()' without a previous call to 'stop()' will end up in an error.

To stop the server, execute the following method:
```
lwtm2m.stop(function(error) {
  console.log('Server stopped');
});
```
No information is needed to stop the server, as there is a single instance per module.

#### Handling incoming messages
The server listens to multiple kinds incoming messages from the devices, described in the different LWTM2M Interfaces. For each operation of an interface that needs to be captured by the server, this library provides a handler that will have the opportunity to manage the event.

The following table lists the current supported events along with the expected signature of the handlers. Be careful with the handler signatures, as an interruption in the callbacks pipeline may hang up your server.

| Interface        | Operation              | Code                  | Signature            |
|:---------------- |:---------------------- |:--------------------- |:-------------------- |
| Registration     | Register               | registration          | fn(endpoint, lifetime, version, binding, callback) |
| Registration     | Update                 | unregistration        | fn(device, callback) |
| Registration     | De-register            | updateRegistration    | function(object, callback) |

The meaning of each parameter should be clear reading the operation description in OMA's documentation.

### Configuration
The configuration object should contain the following fields:
* `server.port`: port where the server's COAP server will start listening.
* `client.port`: port where the client's COAP server will start listening.
* `client.lifetime`: lifetime in miliseconds of the client registration. After that lifetime, the registration will be dismissed.
* `client.version`: version of the Lightweight M2M protocol. Currently `1.0` is the only valid option.
* `client.observe`: default parameters for resource observation (they can be overwritten from the server). 

### Server features (server -> client)
Each writing feature is modelled as a function in the LWTM2M module. The following sections describe the implemented features, identified by its Interface and name of the operation.

#### Device Management Interface: Write
Signature:
```
function write(deviceId, objectType, objectId, resourceId, value, callback)
```
Execute a Write operation over the selected resource, identified following the LWTM2M conventions by its: deviceId, objectType, objectId and resourceId, changing its value to the value passed as a parameter. The device id can be found from the register, based on the name or listing all the available ones.

#### Device Management Interface: Read
Signature:
```
function read(deviceId, objectType, objectId, resourceId, callback)
```
Execute a read operation for the selected resource, identified following the LWTM2M conventions by its: deviceId, objectType, objectId and resourceId. The device id can be found from the register, based on the name or listing all the available ones.

## Development documentation
### Project build
The project is managed using Grunt Task Runner.

For a list of available task, type
```bash
grunt --help
```

The following sections show the available options in detail.


### Testing
[Mocha](http://visionmedia.github.io/mocha/) Test Runner + [Chai](http://chaijs.com/) Assertion Library + [Sinon](http://sinonjs.org/) Spies, stubs.

The test environment is preconfigured to run [BDD](http://chaijs.com/api/bdd/) testing style with
`chai.expect` and `chai.should()` available globally while executing tests, as well as the [Sinon-Chai](http://chaijs.com/plugins/sinon-chai) plugin.

Module mocking during testing can be done with [proxyquire](https://github.com/thlorenz/proxyquire)

To run tests, type
```bash
grunt test
```

Tests reports can be used together with Jenkins to monitor project quality metrics by means of TAP or XUnit plugins.
To generate TAP report in `report/test/unit_tests.tap`, type
```bash
grunt test-report
```


### Coding guidelines
jshint, gjslint

Uses provided .jshintrc and .gjslintrc flag files. The latter requires Python and its use can be disabled
while creating the project skeleton with grunt-init.
To check source code style, type
```bash
grunt lint
```

Checkstyle reports can be used together with Jenkins to monitor project quality metrics by means of Checkstyle
and Violations plugins.
To generate Checkstyle and JSLint reports under `report/lint/`, type
```bash
grunt lint-report
```


### Continuous testing

Support for continuous testing by modifying a src file or a test.
For continuous testing, type
```bash
grunt watch
```


### Source Code documentation
dox-foundation

Generates HTML documentation under `site/doc/`. It can be used together with jenkins by means of DocLinks plugin.
For compiling source code documentation, type
```bash
grunt doc
```


### Code Coverage
Istanbul

Analizes the code coverage of your tests.

To generate an HTML coverage report under `site/coverage/` and to print out a summary, type
```bash
# Use git-bash on Windows
grunt coverage
```

To generate a Cobertura report in `report/coverage/cobertura-coverage.xml` that can be used together with Jenkins to
monitor project quality metrics by means of Cobertura plugin, type
```bash
# Use git-bash on Windows
grunt coverage-report
```


### Code complexity
Plato

Analizes code complexity using Plato and stores the report under `site/report/`. It can be used together with jenkins
by means of DocLinks plugin.
For complexity report, type
```bash
grunt complexity
```

### PLC

Update the contributors for the project
```bash
grunt contributors
```


### Development environment

Initialize your environment with git hooks.
```bash
grunt init-dev-env 
```

We strongly suggest you to make an automatic execution of this task for every developer simply by adding the following
lines to your `package.json`
```
{
  "scripts": {
     "postinstall": "grunt init-dev-env"
  }
}
``` 


### Site generation

There is a grunt task to generate the GitHub pages of the project, publishing also coverage, complexity and JSDocs pages.
In order to initialize the GitHub pages, use:

```bash
grunt init-pages
```

This will also create a site folder under the root of your repository. This site folder is detached from your repository's
history, and associated to the gh-pages branch, created for publishing. This initialization action should be done only
once in the project history. Once the site has been initialized, publish with the following command:

```bash
grunt site
```

This command will only work after the developer has executed init-dev-env (that's the goal that will create the detached site).

This command will also launch the coverage, doc and complexity task (see in the above sections).

