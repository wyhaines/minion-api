# Minion API Spec

The Minion architecture will consist of several services and parts, but one of
the most critical is the HTTP API. Laid out below are the main functions that
API needs to have, along with a description of each, example input and example
expected output.

I'm trying to put down "Authentication Required" where it's necessary, but if
I've missed it and it seems like it's something that's needed, use your best
judgement and add it. That goes for all of this - use your best judgement and
if anything's unclear, **feel free to get in touch with me!** Questions and
comments are welcome, and I'd rather hear from you if you have concerns than
see it implemented incorrectly, so when in doubt, get in touch!

- [J. Austin Hughey](mailto:j.austin.hughey@joshsoftware.com)

## API FUNCTIONS

We'll put all API paths under `/api/v1/` to start with rather than making it
a necessary header. This is just easier to do.

## Authentication Overview

Authentication for this app will be via [JWT](http://jwt.io). A user presents
their email and password, and if valid, they'll be issued a JSON Web Token,
cryptographically signed by this app, that will then be saved by the front-end
and presented in an HTTP header on each subsequent request where authentication
is required.

In short, conduct the login as explained below and you'll get a response back
with a header of `Authorization` and a token value for the value of that header.
Every request from then on needs to include that token in its own
`Authorization` header as follows:

```
Authorization: bearer <token goes here>
```

### /users

+ Create a user
+ Login as that user
+ Get information for yourself (current user bearing JWT)
+ Logout (blacklist the token)

#### POST /users

Given a `POST` to `/users`, create a new user. Sample input POST body:

```json
{
  "name":"J. Austin Hughey",
  "email":"jah@jah.io",
  "password":"A really secure password",
  "phone":"+1 (512) 123-4567"
}
```

Sample expected returned user information:

```json
{
  "user_id":"<some uuid here>",
  "email":"jah@jah.io",
  "name":"J. Austin Hughey",
  "phone":"+1 (512) 123-4567"
}
```

When creating a new user, create an organization as well that's named after
their name, with a blank website and that new user specified as the admin for
that organization. This is all placeholder stuff for future iterations but for
now, that's good enough. Just make sure **every user belongs to their own
organization** when they've signed up.

#### GET /users/me

**AUTHENTICATION REQUIRED**

Input: just the JWT token that was issued in authentication

Output (example):

```json
{
  "name":"J. Austin Hughey",
  "email":"jah@jah.io",
  "password":"A really secure password",
  "phone":"+1 (512) 123-4567",
  "organization": {
    "name":"Org Name",
    "website":"http://www.example.com/",
    "admin_id":"<some uuid representing an admin user for this org>"
  }
}
```

### /logout

#### GET /logout

**AUTHENTICATION REQUIRED**

Given an existing token (`Authorization` header), log the person out by adding
that token to a blacklisted tokens table. Then redirect them to the login page.

### /login

#### POST /login

No auth required, only a valid user. Given a POST body of JSON as follows,
ensure the user isn't deactivated (`WHERE deactivated_at IS NULL`) and if found,
authenticate the user with the password provided.

Example request body:

```json
{
  "email":"me@example.com",
  "password":"abc123"
}
```

If the login is valid, return basic user info as seen in `GET /users/me` along
with an Authorization header as follows:

```
Authorization: bearer <JWT goes here>
```

JWT stands for JSON Web Token. It needs to be generated on each valid login, and
then verified each time the user presents it with each request. If the signature
turns out to be valid, then we issued it and the user's request is valid.

Read more at jwt.io.

### /servers

The point of this section of the app is to manage and retrieve information
for each server managed by Minion. We start by registering a given server, which
will then give that server a name in the dashboard and allow us to send new
configuration data out to it via the stream server (different part of the
architecture). Once we know about it, we can then send it commands and review
logs for that server via our logging interface.

A server belongs not to a user, but to an _organization_. So when a minion agent
registers a new server, it'll do so by reporting an organization ID (which will
be our database UUID representing an organization). When the API receives that
registration, it'll generate a name for the server at random (duplication is
OK here, for now anyway) and assign it to that server in our database/dashboard.

#### GET /servers

**AUTHENTICATION REQUIRED**

Returns a list of all servers (for now; we'll "paginate" that later), ordered
by registration date (e.g. `created_at`) that belong to the user's organization.

#### POST /servers

(Authentication not required here since we want to allow easy server
registration. The user will be responsible for specifying their organization
id in a config file that the minion agent will read, and that agent will be
responsible for issuing this HTTP request.)

Server registration as described above. A Minion agent will register a server
by performing a POST request with a body similar to the following example:

```json
{
  "organization_id":"<some uuuid here>"
}
```

From our perspective, a server is a server is a server. We don't care what it's
doing under the hood, that's up to the operator. We'll add custom tagging
support in a future release.

When the server is registered (created in the database), the API should respond
with something like:

```json
{
  "server_id":"<server's uuid here>",
  "organization_id":"<same uuid as before for the org>",
  "name":"Dancing Catfish (a name made up by the app, see Server.name)",
  "config": {
    "todo":"For now return an empty object, but we'll define more formal configuration later"
  }
}
```

#### GET /servers/{UUID}

**AUTHENTICATION REQUIRED**

Given a UUID, and given that the user making the request is of the same
organization as the server that UUID represents, retrieve and render the
following information about that server:

```json
{
  "server_id":"<the uuid of the server>",
  "organization_id":"<the server's organization's uuid>",
  "name":"Dancing Catfish"
}
```

#### GET /servers/{UUID}/metrics

**AUTHENTICATION REQUIRED**

This path should get the CPU and memory usage statistics from the database for
that server and return them in a format that can be used by the front-end to
plot a graph.

**TODO** - We're still working on the details on how this should be implemented.
Stay tuned.

### Streaming / WebSockets

This section of the API deals with streaming information directly from
[RethinkDB](https://rethinkdb.com/), though the final data modeling isn't yet
quite ready as of this writing (it's being worked on). The API will need to
implement a WebSocket streaming connection from the front-end (built separately)
using the API as a bridge, to the RethinkDB server that will hold and notify of
updates when changes occur to a given record.

An example on how to subscribe to record changes with RethinkDB is on their
website: https://rethinkdb.com/docs/changefeeds/ruby/

#### GET /servers/{UUID}/logs/{UUID}

**AUTHENTICATION REQUIRED**

Given the UUID for a log, a client will open a WebSocket connection to the API -
which **you should open a new thread for each WebSocket connection using the
[async](https://github.com/socketry/async) library** - and subscribe to changes
for the log's data in that thread. Every time it's updated by the agent and
streamserver, RethinkDB will _automatically_ report that there's been a change
and return that data to you.

**TODO** - We're still working on the implementation details here. Stay tuned.

#### GET /servers/{UUID}/commands/<UUID>

**AUTHENTICATION REQUIRED**

Given a server UUID and a command for that server, open a WebSocket connection
with the client that will, just as in the logs portion above, stream out new
lines of data to the front-end when an update is received from RethinkDB.

**TODO** - We're still working on the implementation details here. Stay tuned.

### Commands

The bread and butter of Minion is to allow operators (our eventual customers)
to issue commands through the dashboard, and have those commands executed on the
server by minion itself. We need the ability to create a command for a given
server, and that command needs to be saved in RethinkDB so that the stream
server can then pick up that new command and instruct a minion agent to run it.

#### POST /servers/{UUID}/commands

**AUTHENTICATION REQUIRED**

Given a post body like the following:

```json
{
  "server_id":"<uuid for the server>",
  "command":"/bin/bash -c echo \"hello world!\""
}
```

You'll need to create an "external" (type) command in the PostgreSQL database in
the `commands` table. Then you'll need to link it to that server using the
servers_commands` table. (Leave `response_id`, `dispatched_at`, and
`response_at` blank when you insert.) Next, insert the `command_id` into the
`command_queues` table. Then run the SQL query:

```sql
NOTIFY agent_commands;
```

To run it.

### Logs

There will be two ways for users to view logs:

1. By searching them for a specific string or regex within a specific timeframe;
1. By tailing a specific log file in real time on a specific machine.

In the first case, the interface will provide controls for specifying the time
range (between X and Y), and the string or regex to search for. The API then
needs to query Elastic Search for the log information as specified.

In the second case, the API needs to hold open a WebSocket connection with the
end user and stream changes from RethinkDB where the log is specified by UUID.

#### Tailing Logs in Real Time

In RethinkDB, logs will look similar to this:

```json
{
  "uuid":"<uuid>",
  "server_id":"<server uuid>",
  "filename":"/var/log/syslog",
  "lines":[
    {"at":1592022382, "stderr":"Some error", "stdout":"output"},
    {"at":1592022383, "stderr":"Some error", "stdout":"output"},
    {"at":1592022386, "stderr":"Some error", "stdout":"output"},
    {...}
  ]
}
```

(Note that `at` is a Unix timestamp; seconds since the Unix epoch.)

So in RethinkDB, we'll have one document for all the lines of any log file, and
we'll be pushing to that `lines` array in it every time the agent sees a new
line output to either STDERR or STDOUT. The API then needs to subscribe to
changes on the UUID of the log document the user specifies, then over an active
WebSocket, report those changes as new JSON documents. You can just stream new
lines each time you get a new line in the `lines` array, such as:

```json
{"at":1592022383, "stderr":"Some error", "stdout":"output"}
```

#### Searching Logs

In Elastic Search, logs are going to be split up into different documents where
_each line is its own document_. This means that each line needs to hold
metadata about the log and server with it. An entire Elastic Search document
might look like this:

```json
{
  "stderr":"some error",
  "stdout":"some regular output",
  "filename":"/var/log/messages",
  "log_id":"<uuid of log object>",
  "at":1592022383,
  "id":"<uuid specific to elastic search>"
}
```

This is going to result in billions of entries in Elastic Search, _and that's
OK_. We can query against these fields later to aggregate and show log data
that's been searched for.

The API needs to provide a way to search for regex or string match on the
`stderr`, `stdout` fields (either or both) and in a range, specified by the user,
of time that's then converted into the Unix epoch and searched as values between
the lower and upper bounds of said range, inclusive. Return all records in the
above format to the front-end, while creating a way to paginate those results
and order them in chronological order (order of occurrence).

### Minion Agent

We'll need a route to tell the agent what the latest published version is, and
where to go to download it. This is so the agent can upgrade itself.

#### GET `/api/v1/minion`

Sample response:

```json
{
  "latest_version":"0.1.0",
  "sha256":"ad2d3c84228b0d2107173a7291a33931b25de95c",
  "download":"http://localhost:3000/agent"
}
```