# The Actor Programming Model Whitepaper [DRAFT]

**The whitepaper describes new concept for building serverless microapps called _actors_,
which are easy to develop, share, integrate, and build upon.
Actors are a reincarnation of the UNIX philosophy
for programs running in the cloud.**

Written by [Jan Čurn](https://apify.com/jancurn),
[Marek Trunkát](https://apify.com/mtrunkat),
[Ondra Urban](https://apify.com/mnmkng), and the [Apify](https://apify.com/store) team
in June 2024.


## Contents

<!-- toc -->

- [Introduction](#introduction)
  * [Overview](#overview)
- [Basic concepts](#basic-concepts)
  * [Input](#input)
  * [Run environment](#run-environment)
  * [Output](#output)
  * [Storage](#storage)
  * [Integrations](#integrations)
  * [Publishing & Monetization](#publishing--monetization)
  * [What Actors are not](#what-Actors-are-not)
- [Philosophy](#philosophy)
  * [UNIX programs vs. Actors](#unix-programs-vs-Actors)
  * [Design principles](#design-principles)
  * [Relation to the Actor model](#relation-to-the-actor-model)
  * [Why the name "Actor" ?](#why-the-name-actor-)
- [Installation and setup](#installation-and-setup)
  * [Apify platform](#apify-platform)
  * [Node.js](#nodejs)
  * [Python](#python)
  * [Command-line interface (CLI)](#command-line-interface-cli)
- [Programming interface](#programming-interface)
  * [Actor initialization](#actor-initialization)
  * [Get input](#get-input)
  * [Key-value store access](#key-value-store-access)
  * [Push results to dataset](#push-results-to-dataset)
  * [Exit Actor](#exit-actor)
  * [Environment variables](#environment-variables)
  * [Actor status](#actor-status)
  * [System events](#system-events)
  * [Get memory information](#get-memory-information)
  * [Start another Actor](#start-another-actor)
  * [Metamorph 🪄](#metamorph-%F0%9F%AA%84)
  * [Attach webhook to an Actor run](#attach-webhook-to-an-actor-run)
  * [Abort another Actor](#abort-another-actor)
  * [Live view web server](#live-view-web-server)
  * [Migration to another server](#migration-to-another-server)
- [Actor definition files](#actor-definition-files)
  * [Actor file](#actor-file)
  * [Dockerfile](#dockerfile)
  * [README](#readme)
  * [Schema files](#schema-files)
  * [Backward compatibility](#backward-compatibility)
- [Development](#development)
  * [Local development](#local-development)
  * [Deployment to Apify platform](#deployment-to-apify-platform)
  * [Repackaging existing software as Actors](#repackaging-existing-software-as-Actors)
  * [Continuous integration and delivery](#continuous-integration-and-delivery)
- [Sharing & Community](#sharing--community)
  * [Shared Actors](#shared-Actors)
- [TODOs (@jancurn)](#todos-jancurn)

<!-- tocstop -->

## Introduction

This document explains how to develop _actors_,
a new kind of serverless microapps (or agents, programs, ...) for general-purpose language-agnostic computing and automation jobs.
The main design goal for Actors is to make it easy for developers build and ship reusable
cloud software tools, which are also easy to run
and integrate by potentially not-too-technical users.

The Actors were first introduced by [Apify](https://apify.com/) in late 2017,
as a way to easily build, package, and ship web scraping and web automation tools to customers.
Over the next years, Apify kept developing the concept and applied
it successfully to thousands of real-world use cases in many business areas,
well beyond the domain of web scraping.

Drawing on this experience,
we're releasing this specification of the Actor programming model to the public,
in a hope to make it a new open standard, and to help community to more effectively
build and ship software automation tools,
as well as encourage new implementations of the model in other programming languages.

The goal of this document is to be the north star showing what the
Actor programming model is and what its implementations should support.
Currently, the most complete implementation of Actor model is provided
by the Apify platform, with SDKs for 
[Node.js](https://sdk.apify.com/) and
[Python](https://pypi.org/project/apify/),
and a [command-line interface](https://docs.apify.com/cli).
Beware that these implementations do not support all features yet. This is work in progress. 


### Overview

Actors are serverless programs or agents running in the cloud.
They can perform anything from simple actions such as
filling out a web form or sending an email,
to complex operations such as crawling an entire website,
or removing duplicates from a large dataset.
Actors can run as short or as long as necessary, from seconds to hours, even infinitely.

Basically, Actors are programs packaged as Docker images,
which accept a well-defined JSON input, perform
an action, and optionally produce a well-defined JSON output.

Actors have the following elements:
- **Dockerfile** which specifies where is the Actor's source code,
  how to build it, and run it
- **Documentation** in a form of README.md file
- **Input and output schemas** that describe what input the Actor requires,
  and what results it produces
- Access to an out-of-box **storage system** for Actor data, results, and files
- **Metadata** such as the Actor name, description, author, and version

The documentation and the input/output schemas make it possible for people to easily understand what the Actor does,
enter the required inputs both in user interface or API,
and integrate the results of the Actor into their other workflows.
Actors can easily call and interact with each other, enabling building more complex
systems on top of simple ones.

The Actors can be published
on the [Apify platform](https://apify.com/store),
which automatically generates a rich website with documentation
and a practical user interface, in order to encourage people to try the Actor right away.
The Apify platform takes care of securely hosting the Actors' Docker containers
and scaling the computing, storage and network resources as needed,
so neither Actor developers nor the users need to deal with the infrastructure.
It just works.

The Apify platform provides an open API, cron-style scheduler, webhooks
and [integrations](https://apify.com/integrations)
to services such as Zapier or Make, which make it easy for users
to integrate Actors into their existing workflows. Additionally, the Actor developers
can set a price tag for the usage of their Actors, and thus make
[passive income](https://blog.apify.com/make-regular-passive-income-developing-web-automation-Actors-b0392278d085/)
to have an incentive to keep developing and improving the Actor for the users.

Currently, Actors can run locally or on the Apify platform. However, one of the goals of this open 
specification is to motivate creation of new runtime environments outside of Apify.

The ultimate goal of the Actor programming model is to make it as simple as possible
for people to develop, run, and integrate software automation tools.


## Basic concepts

This section describes core features of Actors, what they are good for,
and how Actors differ from other serverless computing platforms. 

### Input

Each Actor accepts an **input object**, which tells it what it should do.
The object is passed in JSON format, and its properties have
a similar role as command-line arguments when running a program in a UNIX-like operating system.

For example, an input object for an Actor `bob/screenshot-taker` can look like this:

```json
{
  "url": "http://www.example.com",
  "width": 800,
  "height": 600
}
```

The input object represents a standardized way for the caller to control the Actor's activity,
whether starting it using API, in user interface, CLI, or scheduler.
The Actor can access the value of the input object using the [Get input](#get-input) function.

In order to specify what kind of input object an Actor expects,
the Actor developer can define an [Input schema file](./pages/INPUT_SCHEMA.md).
For example, the input schema for Actor `bob/screenshot-taker` will look like this:

```json
TODO: Actor example
```

The input schema is used by the system to:

- Validate the passed input JSON object on Actor run,
  so that Actors don't need to perform input validation and error handling in their code.
- Render user interface for Actors to make it easy for users to run and test them manually
- Generate Actor API documentation and integration code examples on the web or in CLI, 
  making Actors easy for users to integrate the Actors.
- Simplify integration of Actors into automation workflows such as Zapier or Make, by providing smart connectors
  that smartly pre-populate and link Actor input properties

**TODO: Actor example - Show screenshots with manual input in Console, generated API docs, and code examples for that Actor **

### Run environment

The Actors run within an isolated Docker container with access to local file system and network,
and they can perform an arbitrary computing activity or call external APIs.
The **standard output** of the Actor's program (stdout and stderr) is printed out and logged,
which is useful for development and debugging.

To inform the users about the progress, the Actors might set a [**status message**](#actor-status),
which is then displayed in the user interface and also available via API.

Running Actors can also launch a [**live-view web server**](#live-view-web-server),
which is assigned a unique local or public URL to receive HTTP requests. For example,
this is useful for messaging and interaction between Actors, for running request-response REST APIs, or 
providing a full-featured website.

The Actors can store their working data or results into specialized **storages**
called [Key-value store](#key-value-store) and [Dataset](#dataset) storages,
from which they can be easily exported using API or integrated in other Actors.

### Output

While the input object provides a standardized way to invoke Actors,
the Actors can also generate an **output object**, which is a standardized way to display, consume, and integrate
Actors' results.

The Actor results are typically fully available only after the Actor run finishes,
but the consumers of the results might want to access partial results during the run.
Therefore, the Actors don't generate the output object directly, but rather
define an [Output schema file](./pages/OUTPUT_SCHEMA.md), which contains
instruction how to generate the output object. The output object is stored
to the Actor run object under the `output` property, and returned via API immediately after
the Actor is started, without the need to wait for it to finish or generate the actual results.

The output object is similar to input object, as it contains properties and values.
For example, for the `bob/screenshot-taker` Actor the output object can look like this:

```json
{
  "fileUrl": ""
}
```

The output object is generated automatically by the system based on the output schema file,
which looks as follows:

```json
TODO: Actor example
```

The output schema and output object can then be used by callers of Actors to figure where to find
Actor results, how to display them to users, or simplify plugging of Actors in workflow automation pipelines.

### Storage

The Actor system provides two specialized storages that can be used by Actors for storing files and results:
**Key-value store** and **Dataset**, respectively. For each Actor run,
the system automatically creates both these storages 
in empty state, and makes them readily available for the Actor.

Besides these default storages, Actors are free to create new or
access other existing key-value stores and datasets, either by ID or a name that can be set to them.
The storages are accessible through an API and SDK also externally, for example,
to download results when the Actor finishes.

Note that the Actors are free to access any other external storage through any third-party API.

#### Key-value store

The key-value store is a simple data storage that is used for saving and reading
files or data records. The records are represented by a unique text key and the data associated with a MIME content type.
Key-value stores are ideal for saving things like screenshots, web pages, PDFs, or to persist the state of Actors.

Each Actor run is associated with a default empty key-value store, which is created exclusively for the run.
The [Actor input](#input) is stored as JSON file into the default key-value store under the key defined by
the `ACTOR_INPUT_KEY` environment variable (usually `INPUT`).
The Actor can read this object using the [Get input](#get-input) function.

The Actor can read and write records to key-value stores using the API. For details,
see [Key-value store access](#key-value-store-access).


Output + schema...


#### Dataset

Dataset storage allows you to store a series of data objects such as results from web scraping, crawling or data processing jobs. You can export your datasets in JSON, CSV, XML, RSS, Excel or HTML formats.

The Dataset represents a store for structured data where each object stored has the same attributes,
such as online store products or real estate offers. You can imagine it as a table, where each object is
a row and its attributes are columns. Dataset is an append-only storage - you can only add new records to
it but you cannot modify or remove existing records. Typically it is used to store crawling results.

Larger results can be saved to append-only object storage called [Dataset](https://sdk.apify.com/docs/api/dataset).
When an Actor starts, by default it is associated with a newly-created empty default dataset.
The Actor can create additional datasets or access existing datasets created by other Actors,
and use those as needed.

### Integrations

Describe chaining, webhooks, running another, metamorph etc.


### Publishing & Monetization

....Charging money - basic info?

### What Actors are not

Actors are best suited for batch operations that take an input, perform an isolated job for a user,
and potentially produce some output.
However, Actors are currently not best suited for continuous computing or storage workloads, such
as running a live website, API backend, or database.

As Actors are based on Docker, it takes certain amount of time to spin up the container
and launch its main process. Doing this for every small HTTP transaction (e.g. API call) is not efficient,
even for highly-optimized Docker images. For long-running jobs, Actor execution might be migrated
to another machine, making it unsuitable for running databases.

## Philosophy

Actors are inspired by the **[UNIX philosophy](https://en.wikipedia.org/wiki/Unix_philosophy)** from the 1970s, adapted to the age of the cloud:

1. **Make each program do one thing well**. To do a new job, build afresh rather than complicate old programs by adding new “features”.
2. Expect the **output of every program to become the input to another, as yet unknown, program**. Don’t clutter output with extraneous information. Avoid stringently columnar or binary input formats. Don’t insist on interactive input.
3. Design and build software, even operating systems, to be **tried early**, ideally within weeks. Don’t hesitate to throw away the clumsy parts and rebuild them.
4. **Use tools in preference to unskilled help** to lighten a programming task, even if you have to detour to build the tools and expect to throw some of them out after you’ve finished using them.

The UNIX philosophy is arguably one of the most important software engineering paradigms
which, together with other favorable design choices of UNIX operating systems,
ushered the computer and internet revolution.
By combining smaller parts
that can be developed and used independently (programs),
it suddenly became possible to build, manage and gradually evolve ever more complex computing systems.
Even today's modern mobile devices are effectively UNIX-based machines that run a lot of programs
interacting with each other, and provide a terminal
which looks very much like early UNIX terminals. In fact, terminal is just another program.

The UNIX-style programs represent a great way to package software for usage
on a local computer. The programs can be easily used stand-alone,
but also in combination and in scripts
in order to perform much more complex tasks than an individual program ever could,
which in turn can be packaged as new programs.

The idea of Actors is to bring benefits of UNIX-style programs
from a local computer into an environment of cloud
where programs run on multiple computers
communicating over a network that is subject to latency and partitioning,
there is no global atomic filesystem,
and where programs are invoked via API calls rather than system calls.

Each Actor should do just one thing and do it well.
Actors can be used stand-alone, as well as combined or scripted into more complex
systems, which in turn can become new Actors.
Actors provide a simple user interface and documentation to help users interact with them.

### UNIX programs vs. Actors

The following table shows equivalents of key concepts of UNIX programs and Actors.

| UNIX programs            | Actors                                                                                                                             |
|--------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Command-line options     | [Input object](#get-input)                                                                                                         |
| Read stdin               | No direct equivalent, you can [read from a dataset](#dataset) specified in the input.                                              |
| Write to stdout	       | [Push results to dataset](#push-results-to-dataset), set [Actor status](#actor-status)                                             |
| Write to stderr	       | No direct equivalent, you can write errors to log, set error status message, or push failed dataset items into an "error" dataset. |
| File system	           | [Key-value store](#key-value-store-access)                                                                                         |
| Process identifier (PID) | Actor run ID                                                                                                                       |
| Process exit code        | [Actor exit code](#exit-actor)                                                                                                     |

### Design principles

- Each Actor should do just one thing, and do it well.
- Optimize for the users of the Actors, help them understand what the Actor does, easily run it, and integrate.
- Keep the system as simple as possible, so that Actors can be built and used by the top 90% of developers.

### Relation to the Actor model

Note that Actors are only loosely related to
the **Actor model** known from computer science.
According to [Wikipedia](https://en.wikipedia.org/wiki/Actor_model):

> The Actor model in computer science is a mathematical model of concurrent computation
> that treats Actor as the universal primitive of concurrent computation.
> In response to a message it receives, an Actor can: make local decisions,
> create more Actors, send more messages, and determine how to respond to the
> next message received. Actors may modify their own private state,
> but can only affect each other indirectly through messaging
> (removing the need for lock-based synchronization).

While the theoretical Actor model is conceptually very similar to "our" Actor programming model,
this similarity is rather coincidental.
Our primary focus was always on practical software engineering utility, not an
implementation of a formal mathematical model.

For example, our Actors
do not provide any standard message passing mechanism. The Actors might communicate together
directly via HTTP requests (see [Live view](#live-view)),
manipulate each other's operation using the Apify platform API (e.g. abort another Actor),
or affect each other by sharing some internal state or storage.
The Actors simply do not have any formal restrictions,
and they can access whichever external systems they want.


### Why the name "Actor" ?

In movies and theater, an _actor_ is someone who gets a script
and plays a role according to that script.
Our Actors also perform an act on someone's behalf, using a provided script,
and thus we considered the name "Actor" as a good description.
Also, an "Actor" evokes an idea of a person, which is a helpful way to think of and talk about
Actors as independent entities.

Coincidentally, in the web automation world it became common to call libraries
using names related to theater, such as Puppeteer or Playwright,
confirming "Actor" was a good choice.
Last but no least, our model of Actors is similar
to the Actor model known from the computer science.


## Installation and setup

Below are steps to start building Actors in various languages and environments.

### Apify platform

You can develop and run Actors in [Apify Console](https://console.apify.com/Actors) without
installing any software locally. Just create a free account, and start building Actors
in an online IDE.

### Node.js

The most complete implementation of Actor system is provided by the Apify SDK for Node.js,
via the [apify](https://www.npmjs.com/package/apify) NPM package. The package contains everything
that you need to start building Actors locally.
You can install it to your Node.js project by running: 

```bash
$ npm install apify
```

### Python

To build Actors in Python, simply install the Apify SDK for Python,
via the [apify](https://pypi.org/project/apify/) PyPi package
into your project:

```bash
$ pip3 install apify
```

### Command-line interface (CLI)

For local development of Actors and management of the Apify platform,
it is handy to install Apify CLI.
You can install it:

```bash
$ brew install apify-cli
```

or via the [apify-cli](https://www.npmjs.com/package/apify-cli) Node.js package:

```bash
$ npm install -g apify-cli
```

You can confirm the installation succeeded and login to the Apify platform by running:

```bash
$ apify login
```

The Apify CLI provides two commands: `apify` and `actor`.

`apify` command lets you interact with the Apify platform, for example run an Actor,
push deployment of an Actor to cloud, or access storages. For details, see [Local development](#local-development).

`actor` command is to be used from within an Actor in the runtime, to implement the Actors functionality in a shell script.
   For details, see [Actor-izing existing software](#actorizing-existing-software).
   
To get a help for a specific command, run:

 ```bash
$ apify help <command>
$ actor help <command>
```

## Actor programming interface

The commands described in this section are expected to be called from within a context
of a running Actor, both in local environment or on the Apify platform.

The Actor runtime system passes the context via [environment variables](#environment-variables),
such as `APIFY_TOKEN` or `ACTOR_RUN_ID`, which is used by the SDK or CLI to interact with the runtime.

### Initialization

The SDKs provide convenience methods to initialize the Actor and handle its result.
During initialization, the SDK loads environment variables, checks configuration, prepares to receive system events,
and optionally purges previous state from local storage.

#### Node.js
 
In Node.js the Actor is initialized by calling the `init()` method. It should be paired with an `exit()` method
which terminates the Actor. Use of `exit()` is not required, but recommended. For more information go to [Exit Actor](#exit-actor).

```js
import { Actor } from 'apify';

await Actor.init();

const input = await Actor.getInput();
console.log(input);

await Actor.exit();
```

An alternative way of initializing the Actor is with a `main()` function. This is useful in environments where the latest JavaScript
syntax and top level awaits are not supported. The main function is only syntax-sugar for `init()` and `exit()`. It will call `init()` before it executes its callback and `exit()` after the callback resolves. 

```js
import { Actor } from 'apify';

Actor.main(async () => {
  const input = await Actor.getInput();
  // ...
});
```

#### Python

TODO: Add Python examples

#### CLI

No initialization needed, the process exit terminates the Actor, with the process status code
determining if it succeeded or failed.

```bash
$ actor set-status-message "My work is done, friend"
$ exit 0
```

#### UNIX equivalent

```c
int main (int argc, char *argv[]) {
  ...
}
```

### Get input

Get access to the Actor input object passed by the user.
It is parsed from a JSON file, which is stored by the system in the Actor's default key-value store,
Usually the file is called `INPUT`, but the exact key is defined in the `ACTOR_INPUT_KEY` [environment variable](#environment-variables).

The input is an object with properties.
If the Actor defines the input schema, the input object is guaranteed to conform to it.
For details, see [Input and output](#input-and-output).

#### Node.js

```js
const input = await Actor.getInput();
console.log(input);

// prints: { "option1": "aaa", "option2": 456 }
```

#### Python

```python
input = actor.get_input()
print(input)
```

#### CLI

```bash
# Emits a JSON object, which can be parsed e.g. using the "jq" tool
$ actor get-input | jq

> { "option1": "aaa", "option2": 456 }
```

#### UNIX equivalent

```bash
$ command --option1=aaa --option2=bbb
```

```c
int main (int argc, char *argv[]) {}
```

### Key-value store access

Write and read arbitrary files using a storage
called [Key-value store](https://sdk.apify.com/docs/api/key-value-store).
When an Actor starts, by default it is associated with a newly-created key-value store,
which only contains one file with input of the Actor (see [Get input](#get-input)).

The user can override this behavior and specify another key-value store or input key
when running the Actor.

#### Node.js

```js
// Save objects to the default key-value store
await Actor.setValue('my_state', { something: 123 }); //  (stringified to JSON)
await Actor.setValue('screenshot.png', buffer, { contentType: 'image/png' });

// Get record from the default key-value store, automatically parsed from JSON
const value = await Actor.getValue('my_state');

// Access another key-value store by its name
const store = await Actor.openKeyValueStore('screenshots-store');
const imageBuffer = await store.getValue('screenshot.png');
```

#### Python

```python
# Save object to store (stringified to JSON)
await actor.set_value('my-state', { something=123 })

# Save binary file to store with content type
await actor.set_value('screenshot', buffer, { contentType='image/png' })

# Get object from store (automatically parsed from JSON)
dataset = await actor.get_value('my-state')
```

#### UNIX

```bash
$ echo "hello world" > file.txt
$ cat file.txt
```

### Push results to dataset

Larger results can be saved to append-only object storage called [Dataset](https://sdk.apify.com/docs/api/dataset).
When an Actor starts, by default it is associated with a newly-created empty default dataset.
The Actor can create additional datasets or access existing datasets created by other Actors,
and use those as needed.

Note that Datasets can optionally be equipped with schema that ensures only certain kinds
of objects are stored in them. See [Dataset schema file](./pages/DATASET_SCHEMA.md) for more details.

#### Node.js

```js
// Append result object to the default dataset associated with the run
await Actor.pushData({
    someResult: 123,
});

// Append result object to a specific named dataset
const dataset = await Actor.openDataset('bob/poll-results-2019');
await dataset.pushData({ someResult: 123 });
```

#### Python

```python
# Append result object to the default dataset associated with the run
await actor.push_data({ some_result=123 })

# Append result object to a specific named dataset
dataset = await actor.open_dataset('bob/poll-results-2019')
await dataset.push_data({ some_result=123 })
```

#### CLI

```bash
# Push data to default dataset, in JSON format
$ echo '{ "someResult": 123 }' | actor push-data --json
$ actor push-data --json='{ "someResult": 123 }'
$ actor push-data --json=@result.json

# Push data to default dataset, in text format
$ echo "someResult=123" | actor push-data
$ actor push-data someResult=123

# Push to a specific dataset in the cloud
$ actor push-data --dataset=bob/election-data someResult=123

# Push to dataset on local system
$ actor push-data --dataset=./my_dataset someResult=123
```

#### UNIX equivalent

```c
printf("Hello world\tColum 2\tColumn 3");
```

### Exit Actor

When the main Actor process exits (i.e. the Docker container stops running),
the Actor run is considered finished and the process exit code is used to determine
whether the Actor has succeeded (exit code `0` leads to status `SUCCEEDED`)
or failed (exit code not equal to `0` leads to status `SUCCEEDED`).
In this case, the platforms set a status message to a default value like `Actor exit with exit code 0`,
which is not very descriptive for the users.

An alternative and preferred way to exit an Actor is using the `exit` function in SDK, as
shown below. This has two advantages:

- You can provide a custom status message for users to tell them what the Actor achieved
  On error, try to explain to users
  what happened and most importantly, how they can fix the error.
  This greatly improves user experience.
- The system emits the `exit` event, which can be listened to and used by various
  components of the Actor to perform a cleanup, persist state, etc.
  Note that the caller of exit can specify how long should the system wait for all `exit`
  event handlers to complete before closing the process, using the `timeoutSecs` option.
  For details, see [System Events](#system-events).

#### Node.js

```js
// Actor will finish with 'SUCCEEDED' status
await Actor.exit('Succeeded, crawled 50 pages');

// Exit right away without calling `exit` handlers at all
await Actor.exit('Done right now', { timeoutSecs: 0 });

// Actor will finish with 'FAILED' status 
await Actor.exit('Could not finish the crawl, try increasing memory', { exitCode: 1 });

// ... or nicer way using this syntactic sugar:
await Actor.fail('Could not finish the crawl, try increasing memory');

// Register a handler to be called on exit.
// Note that the handler has `timeoutSecs` to finish its job
Actor.on('exit', ({ statusMessage, exitCode, timeoutSecs }) => {
    // Perform cleanup...
})
```

#### Python

```python
# Actor will finish in 'SUCCEEDED' state
await actor.exit('Generated 14 screenshots')

# Actor will finish in 'FAILED' state
await actor.exit('Could not finish the crawl, try increasing memory', { exitCode: 1 })
# ... or nicer way using this syntactic sugar:
await Actor.fail('Could not finish the crawl, try increasing memory');
```

#### CLI

```bash
# Actor will finish in 'SUCCEEDED' state
$ actor exit
$ actor exit --message "Email sent"

# Actor will finish in 'FAILED' state
$ actor exit --code=1 --message "Couldn't fetch the URL"
```

#### UNIX equivalent

```c
exit(1);
```


### Environment variables

Actors have access to standard process environment variables. 
The Apify platform uses environment variables prefixed with `ACTOR_` to pass the Actors information
about the execution context.

| Environment variable               | Description                                                                                                                                                                                                                |
|------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `ACTOR_ID`                         | ID of the Actor. |
| `ACTOR_RUN_ID`                     | ID of the Actor run.                                                                                                                                                                                                       |
| `ACTOR_BUILD_ID`                   | ID of the Actor build. |
| `ACTOR_BUILD_NUMBER`               | A string representing the version of the current Actor build. |
| `ACTOR_TASK_ID`                    | ID of the saved Actor task. |
| `ACTOR_DEFAULT_KEY_VALUE_STORE_ID` | ID of the key-value store where the Actor's input and output data are stored.                                                                                                                                    |
| `ACTOR_DEFAULT_DATASET_ID`         | ID of the dataset where you can push the data.                                                                                                                                                                        |
| `ACTOR_DEFAULT_REQUEST_QUEUE_ID`   | ID of the request queue that stores and handles requests that you enqueue.                                                                                                                                            |
| `ACTOR_INPUT_KEY`                  | The key of the record in the default key-value store that holds the Actor input. Typically it's `INPUT`, but it might be something else.                                                             |
| `ACTOR_MEMORY_MBYTES`              | Indicates the size of memory allocated for the Actor run, in megabytes (1,000,000 bytes). It can be used by Actors to optimize their memory usage.                                                                       |
| `ACTOR_STARTED_AT`                 | Date when the Actor was started, in ISO 8601 format. For example, `2022-01-02T03:04:05.678`.                                                                                                                                                                                          |
| `ACTOR_TIMEOUT_AT`                 | Date when the Actor will time out, in ISO 8601 format.                                                                                                                                                                          |
| `ACTOR_EVENTS_WEBSOCKET_URL`       | Websocket URL where Actor may listen for events from Actor platform. See [System events](#system-events) for more details.                                       |
| `ACTOR_WEB_SERVER_PORT`            | TCP port on which the Actor can start a HTTP server to receive messages from the outside world. See [Live view web server](#live-view-web-server) section for more details. |
| `ACTOR_WEB_SERVER_URL`             | A unique public URL under which the Actor run web server is accessible from the outside world. See [Live view web server](#live-view-web-server) section for more details.  |
| `ACTOR_MAX_PAID_DATASET_ITEMS`     | A maximum number of results that will be charged to the user using a pay-per-result Actor. |

The Actor developer can also define custom environment variables
that are then passed to the Actor process both in local development environment or on the Apify platform.
These variables are defined in the [.actor/actor.json](/pages/ACTOR.md) file using the `environmentVariables` directive,
or manually in the user interface in Apify Console.

The environment variables can be set as secure in order to protect sensitive data such as API keys or passwords.
The value of a secure environment variable is encrypted and can only be retrieved by the Actors during their run,
but not outside the runs. Furthermore, values of secure environment variables are omitted from the log.

#### Node.js

For convenience, rather than using environment vars directly, we provide a `Configuration` class
that allows reading and updating the Actor configuration.

```javascript
const token = Actor.config.get('token');

// use different token
Actor.config.set('token', 's0m3n3wt0k3n')
```

#### CLI

```bash
$ echo "$ACTOR_RUN_ID started at $ACTOR_STARTED_AT"
```


#### UNIX equivalent

```bash
$ echo $ACTOR_RUN_ID
```

### Actor status

Each Actor run has a status (the `status` field), which can be one of the following values:

|Status|Type|Description|
|--- |--- |--- |
|`READY`|initial|Started but not allocated to any worker yet|
|`RUNNING`|transitional|Executing on a worker|
|`SUCCEEDED`|terminal|Finished successfully|
|`FAILED`|terminal|Run failed|
|`TIMING-OUT`|transitional|Timing out now|
|`TIMED-OUT`|terminal|Timed out|
|`ABORTING`|transitional|Being aborted by user|
|`ABORTED`|terminal|Aborted by user|

Additionally, the Actor run has a status message (the `statusMessage` field),
which contains a text for users informing them what the Actor is currently doing,
and thus greatly improve their user experience.

When an Actor exits, the status message is either automatically set to some default text
(e.g. "Actor finished with exit code 1"), or to a custom message - see [Exit Actor](#exit-actor) for details.

When the Actor is running, it should periodically update the status message as follows,
to keep users informed and happy. The function can be called as often as necessary,
the SDK only invokes API if status changed. This is to simplify the usage.

#### Node.js

```js
await Actor.setStatusMessage('Crawled 45 of 100 pages');

// Setting status message to other Actor externally is also possible
await Actor.setStatusMessage('Everyone is well', { actorRunId: 123 });
```

#### Python

```python
await actor.set_status_message('Crawled 45 of 100 pages')
```

#### CLI

```bash
$ actor set-status-message "Crawled 45 of 100 pages"
$ actor set-status-message --run=[RUN_ID] --token=X "Crawled 45 of 100 pages"
```


### System events

Actors are notified by the system about various events such as a migration to another server,
[abort operation](#abort-another-actor) triggered by another Actor, or the CPU being overloaded.

Currently, the system sends the following events:

| Event name     | Payload | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------- | ------- |------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `cpuInfo`      | `{ isCpuOverloaded: Boolean }` | The event is emitted approximately every second and it indicates whether the Actor is using the maximum of available CPU resources. If that’s the case, the Actor should not add more workload. For example, this event is used by the AutoscaledPool class.                                                                                                                                                                                                                                                                             | 
| `migrating`    | N/A | Emitted when the Actor running on the Apify platform is going to be migrated to another worker server soon. You can use it to persist the state of the Actor and abort the run, to speed up migration. See [Migration to another server](#migration-to-another-server).                                                                                                                                                                                                                                                                  |
| `aborting`     | N/A | When a user aborts an Actor run on the Apify platform, they can choose to abort gracefully to allow the Actor some time before getting killed. This graceful abort emits the `aborting` event which the SDK uses to gracefully stop running crawls and you can use it to do your own cleanup as well.                                                                                                                                                                                                                                    |
| `persistState` | `{ isMigrating: Boolean }` | Emitted in regular intervals (by default 60 seconds) to notify all components of Apify SDK that it is time to persist their state, in order to avoid repeating all work when the Actor restarts. This event is automatically emitted together with the migrating event, in which case the `isMigrating` flag is set to `true`. Otherwise the flag is `false`. Note that the `persistState` event is provided merely for user convenience, you can achieve the same effect using `setInterval()` and listening for the `migrating` event. |

In the future, the event mechanism might be extended to custom events and messages enabling communication between
Actors.

Under the hood, Actors receive the system events by connecting to a web socket address specified
by the `ACTOR_EVENTS_WEBSOCKET_URL` environment variable.
The system sends messages in JSON format in the following structure:

```js
{
    // Event name
    name: String,

    // Time when the event was created, in ISO format
    createdAt: String,
          
    // Optional object with payload      
    data: Object,
}
```

Note that some events (e.g. `persistState`) are not sent by the system via the web socket,
but generated virtually on the Actor SDK level.

#### Node.js

```js
// Add event handler
const handler = (data) => {
  if (data.isCpuOverloaded) console.log('Oh no, we need to slow down!');
}
Actor.on('cpuInfo', handler);

// Remove all handlers for a specific event
Actor.off('systemInfo');

// Remove a specific event handler
Actor.off('systemInfo', handler);
```

#### Python

TODO: Add Python example, here and elsewhere too


#### UNIX equivalent

```c
signal(SIGINT, handle_sigint);
```

### Get memory information

Get information about the total and available memory of the Actor’s container or local system.
For example, this is useful to auto-scale a pool
of workers used for crawling large websites.

#### Node.js

```
const memoryInfo = await Actor.getMemoryInfo();
```

#### UNIX equivalent

```bash
# Print memory usage of programs
$ ps -a
```


### Start another Actor

Actor can start other Actors, if they have a permission.

It can override the default dataset or key-value store,
and e.g. forwarding the data to another named dataset,
that will be consumed by the other Actor.

The `call` operation waits for the other Actor to finish, the `start` operation
returns immediately.

#### Node.js

```js
// Start Actor and return a Run object
const run = await Actor.start(
    'apify/google-search-scraper', // name of the Actor to start
    { queries: 'test' }, // input of the Actor
    { memory: 2048 }, // run configuration
);

// Start Actor and wait for it to finish
const run2 = await Actor.call(
  'apify/google-search-scraper', 
  { queries: 'test' },
  { memory: 2048 },
);
```


#### CLI

```bash
# On stdout, the commands emit Actor run object (in text or JSON format),
# we shouldn't wait for finish, for that it should be e.g. "execute"
$ apify call apify/google-search-scraper queries='test\ntest2' \
  countryCode='US'
$ apify call --json apify/google-search-scraper '{ "queries": }'
$ apify call --input=@data.json --json apify/google-search-scraper
$ apify call --memory=1024 --build=beta apify/google-search-scraper
$ apify call --output-record-key=SCREENSHOT apify/google-search-scraper

# Pass input from stdin
$ cat input.json | actor call apify/google-search-scraper --json

# Call local actor during development
$ apify call file:../some-dir someInput='xxx'
```

#### Slack

It will also be possible to run Actors from Slack app.
The following command starts the Actor, and then prints the messages to a Slack channel.

```
/apify start bob/google-search-scraper startUrl=afff
```

#### API

```
[POST] https://api.apify.com/v2/actors/apify~google-search-scraper/run

[POST|GET] https://api.apify.com/v2/actors/apify~google-search-scraper/run-sync?
  token=rWLaYmvZeK55uatRrZib4xbZs&
  outputRecordKey=OUTPUT
  returnDataset=true
```

#### UNIX equivalent

```bash
# Run a program in the background
$ command <arg1>, <arg2>, … &
```

```c
// Spawn another process
posix_spawn();
```

### Metamorph 🪄

This is the most magical Actor operation. It replaces running Actor’s Docker image with another Actor,
similarly to UNIX `exec` command.
It is used for building new Actors on top of existing ones.
You simply define input schema and write README for a specific use case,
and then delegate the work to another Actor.

The target Actor inherits the default storages used by the calling Actor.
The target Actor input is stored to the default key-value store,
under a key such as `INPUT-2` (the actual key is passed via the `ACTOR_INPUT_KEY` [environment variable](#environment-variables)).
Internally, the target Actor can recursively metamorph into another Actor.

**PROPOSAL:**

An Actor can metamorph only to Actors that have compatible output schema as the main Actor,
in order to ensure logical and consistent outcomes for users. 
If the output schema of the target Actor is not compatible, the system should throw an error.

#### Node.js

```js
await Actor.metamorph(
    'bob/web-scraper',
    { startUrls: [ "https://www.example.com" ] },
    { memoryMbytes: 4096 },
);
```

#### CLI

```bash
$ actor metamorph bob/web-scraper startUrls=http://example.com
$ actor metamorph --input=@input.json --json --memory=4096 \
  bob/web-scraper
```

#### UNIX equivalent

```bash
$ exec /bin/bash
```

### Attach webhook to an Actor run

Run another Actor or an external HTTP API endpoint after Actor run finishes or fails.


#### Node.js

```js
await Actor.addWebhook({
    eventType: ['ACTOR.RUN.SUCCEEDED', 'ACTOR.RUN.FAILED'],
    requestUrl: 'http://api.example.com?something=123',
    payloadTemplate: `{
        "userId": {{userId}},
        "createdAt": {{createdAt}},
        "eventType": {{eventType}},
        "eventData": {{eventData}},
        "resource": {{resource}}
    }`,
});
```

#### CLI

```bash
$ actor add-webhook \\
  --event-types=ACTOR.RUN.SUCCEEDED,ACTOR.RUN.FAILED \\
  --request-url=https://api.example.com \\
  --payload-template='{ "test": 123" }'

$ actor add-webhook --event-types=ACTOR.RUN.SUCCEEDED \\
  --request-actor=apify/send-mail \\
  --memory=4096 --build=beta \\
  --payload-template=@template.json

# Or maybe have a simpler API for self-actor?
$ actor add-webhook --event-types=ACTOR.RUN.SUCCEEDED --request-actor=apify/send-mail 
```

#### UNIX equivalent

```bash
# Execute commands sequentially, based on their status
$ command1; command2    # (command separator)
$ command1 && command2  # ("andf" symbol)
$ command1 || command2  # ("orf" symbol)
```


### Abort another Actor

Abort itself or another Actor running on the Apify platform.
Aborting an Actor changes its [status](#actor-status) to `ABORTED`.

#### Node.js

```js
await Actor.abort({ statusMessage: 'Your job is done, friend.', actorRunId: 'RUN_ID' });
```

#### CLI

```bash
$ actor abort --run-id RUN_ID 
```

#### UNIX equivalent

```bash
# Terminate a program
$ kill <PID>
```

<!-- TODO: Include Actor.boot() or not? -->

### Live view web server

An Actor can launch an HTTP web server that is exposed to the outer world.
This enables Actors to provide a custom HTTP API to integrate with other systems,
to provide a web application for human users, to show Actor run details, diagnostics, charts,
or to run an arbitrary web app.

On Apify platform, the port on which the Actor can launch the public web server,
is specified by the `ACTOR_WEB_SERVER_PORT` environment variable.
The web server is then exposed to the public internet on a URL identified 
by the `ACTOR_WEB_SERVER_URL`, for example `https://hard-to-guess-identifier.runs.apify.net`.

#### Node.js

```js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send('Hello World!')
})

app.listen(process.env.ACTOR_WEB_SERVER_PORT, () => {
  console.log(`Example live view web server running at ${process.env.ACTOR_WEB_SERVER_URL}`)
})
```

### Standby Mode

# TODO: Write this


### Migration to another server

The Actors can be migrated from another host server from time to time, especially the long-running ones.
When the migration is imminent, the system sends the Actor the `migrating` [system event](#system-events)
to inform the Actor, so that it can persist its state to storages.
All executed writes to the default Actor [storage](#storage) are guaranteed to be persisted before the migration.
After the migration, Actor is restarted on a new host. It can restore its customer state from the storages again.

## Actor definition files

The Actor system uses several special files that define Actor metadata, documentation,
instructions how to build and run it, input and output schema, etc.

**These files MUST be stored in the `.actor` directory placed in Actor's top-level directory.
The entire `.actor` directory should be added to the source control.**
The only required files are [Actor file](#actor-file) and [Dockerfile](#dockerfile),
all other files are optional.

The Actor definition files are used by the CLI (e.g. by `apify push` and `apify run` commands),
as well as when building Actors on the Apify platform.
The motivation to place the files into a separate directory
is to keep the source code repository tidy and to prevent interactions with other source files,
in particular when creating an Actor from pre-existing software repositories.


### Actor file

This is the main definition file of the Actor in JSON format,
and it always must be present at `.actor/actor.json`.
This file contains references to all other necessary files.

```json
// TODO: Show small example
```

For details, see the [Actor file](./pages/ACTOR.md) page.


### Dockerfile

This file contains instructions for the system how to build the Actor's
Docker image and how to run it.
Actors are started by running their Docker image,
both locally using the `apify run` command,
as well as on the Apify platform.

```dockerfile
// TODO: Show small example
```

The Dockerfile is referenced from the [Actor file](./pages/ACTOR.md) using the `dockerfile`
directive, and typically stored at `.actor/Dockerfile`.
Note that paths in Dockerfile are ALWAYS specified relative to the Dockerfile's location.
Learn more about Dockerfiles in the official [Docker reference](https://docs.docker.com/engine/reference/builder/).


### README

The README file contains Actor documentation written
in [Markdown](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
format.
It is used to generate Actor's public web page on Apify,
and it should contain great explanation what the Actor does and how to use it.

The README file is referenced from the [Actor file](./pages/ACTOR.md) using the `readme`
directive, and typically stored at `.actor/README.md`.

Good documentation makes good Actors!
[Learn more](https://docs.apify.com/actors/publishing/seo-and-promotion) how to write great SEO-optimized READMEs.


### Schema files

The structure of Actor's [input and output](#input-and-output) can be optionally
dictated by the input and output schema files.
These files list properties accepted by Actor on input, and properties that the Actor produces on output,
respectively.
The input and output schema files are used to render a user interface
to make it easy to run the Actor manually,
to generate API documentation, render the view of Actor's output,
validate the input,
and to enable connecting Actor outputs to input of another Actor for rich workflows. 

The input and output schema is defined by two JSON files that are linked 
from the [Actor file](#actor-file):

- [Input schema file](./pages/INPUT_SCHEMA.md)
- [Output schema file](./pages/OUTPUT_SCHEMA.md)

Both input and output schema files can additionally reference schema files 
for specific storages:

- [Dataset schema file](./pages/DATASET_SCHEMA.md)
- [Key-value store schema file](./pages/KEY_VALUE_STORE_SCHEMA.md)
- [Request queue schema file](./pages/REQUEST_QUEUE_SCHEMA.md)

These storage schemas are used to ensure that stored objects or files 
fulfil specific criteria, their fields have certain types, etc.
On Apify platform, the schemas can be applied to the storages directly,
without Actors.

Note that all the storage schemas are weak, in a sense that if the schema doesn't define a property,
such property can be added to the storage and have an arbitrary type.
Only properties explicitly mentioned by the schema
are validated. This is an important feature which allows extensibility.
For example, a data deduplication Actor might require on input datasets
that have an `uuid: String` field in objects, but it does not care about other fields.


### Backward compatibility

If the `.actor/actor.json` file is missing,
the system falls back to the legacy mode,
and looks for `apify.json`, `Dockerfile`, `README.md` and `INPUT_SCHEMA.json`
files in the Actor's top-level directory instead.
This behavior might be deprecated in the future.


## Development

Actors can be developed locally, using a git integration, or in a web IDE.
The SDK is currently available for Node.js, Python, and CLI.

### Local development

TODO: Explain basic workflow with `apify` - create, run, push etc. Move the full local support for Actors
 to ideas (see https://github.com/apify/actor-specs/pull/7/files#r794681016 )

`apify run` - starts the Actor using Dockerfile
referenced from `.actor/actor.json` or Dockerfile in the Actor top-level directory
(if the first is not present)


### Deployment to Apify platform

The `apify push` CLI command takes information from the `.actor` directory and builds an Actor on the Apify platform,
so that you can run it remotely.

```bash
TODO: Show code example
````


### Continuous integration and delivery

TODO: Mention CI/CD, e.g. how to integrate with GiHub etc.


## Actor-izing existing software

You can repackage an existing software repository
as an Actor by creating the `.actor/` directory with the [Actor definition files](#actor-definition-files)
to describe how to run the software and what is its input and output.

With the `actor` CLI command, it's easy to "Actorize" software tools written in any programming language.
Only provide a shell script that transforms the Actor input to command-line options needed by the software, launch it,
and then transform and store results as Actor output.

**PROPOSAL:**

You can automatically generate the `.actor/` definition files using the CLI:

```bash
$ actor ize
```

The command works on the best-effort basis to wrap the code as an Actor,
depending on its programming language, libraries, and setup.


## Sharing & Community

TODO: Motivation - why building on Apify is easier than building your own SaaS

TODO: Ground rules:
- User of Actor should never need to look into log to understand what happened, e.g. why the Actor failed.
  This info must be in the status message

Fees
