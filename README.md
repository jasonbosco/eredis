# eredis

Non-blocking Redis client with a focus on performance and robustness.

It also supports authentication, choosing a specific database,
transactions and poolboy.

## Example

If you have Redis running on localhost, with default settings, you may
copy and paste the following into a shell to try out Eredis:

    git clone git://github.com/wooga/eredis.git
    cd eredis
    ./rebar compile
    erl -pa ebin/
    {ok, C} = eredis:start_link().
    {ok, <<"OK">>} = eredis:q(C, ["SET", "foo", "bar"]).
    {ok, <<"bar">>} = eredis:q(C, ["GET", "foo"]).
    eredis:close(C).

MSET and MGET:

    KeyValuePairs = ["key1", "value1", "key2", "value2", "key3", "value3"].
    {ok, <<"OK">>} = eredis:q(C, ["MSET" | KeyValuePairs]).
    {ok, Values} = eredis:q(C, ["MGET" | ["key1", "key2", "key3"]]).

EUnit tests:

    ./rebar eunit


## Commands

Eredis has only one function to interact with redis, which is
`eredis:q(Client::pid(), Command::iolist())`. The response will either
be `{ok, Value::binary() | [binary()]}` or `{error,
Message::binary()}`. The value is always the exact value returned by
Redis, without any type conversion. If Redis returns a list of values,
this list is returned in the exact same order without any type
conversion.

To start the client, use any of the `eredis:start_link/0,1,2,3,4,5`
functions. They all include sensible defaults. `start_link/5` takes
the following arguments:

* Host, dns name or ip adress as string
* Port, integer, default is 6379
* Database, integer or 0 for default database
* Password, string or empty string([]) for no password
* Reconnect sleep, integer of milliseconds to sleep between reconnect attempts

## Reconnecting on Redis down / network failure / timeout / etc

When Eredis for some reason looses the connection to Redis, Eredis
will keep trying to reconnect until a connection is successfully
established, which includes the `AUTH` and `SELECT` calls. The sleep
time between attempts to reconnect can be set in the
`eredis:start_link/5` call.

As long as the connection is down, Eredis will respond to any request
immediately with `{error, no_connection}` without actually trying to
connect. This serves as a kind of circuit breaker and prevents a
stampede of clients just waiting for a failed connection attempt or
`gen_server:call` timeout.

Note: If Eredis is starting up and cannot connect, it will fail
immediately with `{connection_error, Reason}`.

## AUTH and SELECT

Eredis also implements the AUTH and SELECT calls for you. When the
client is started with something else than default values for password
and database, it will issue the `AUTH` and `SELECT` commands
appropriately, even when reconnecting after a timeout.


## Benchmarking

Using basho_bench(https://github.com/basho/basho_bench/) you may
benchmark Eredis on your own hardware using the provided config and
driver. See `priv/basho_bench_driver_eredis.config` and
`src/basho_bench_driver_eredis.erl`.

## Queueing

Eredis uses the same queueing mechanism as Erldis. `eredis:q/2` uses
`gen_server:call/2` to do a blocking call to the client
gen_server. The client will immediately send the request to Redis, add
the caller to the queue and reply with `noreply`. This frees the
gen_server up to accept new requests and parse responses as they come
on the socket.

When data is received on the socket, we call `eredis_parser:parse/2`
until it returns a value, we then use `gen_server:reply/2` to reply to
the first process waiting in the queue.

This queueing mechanism works because Redis guarantees that the
response will be in the same order as the requests.

## Response parsing

The response parser is the biggest difference between Eredis and other
libraries like Erldis, redis-erl and redis_pool. The common approach
is to either directly block or use active once to get the first part
of the response, then repeatedly use `gen_tcp:recv/2` to get more data
when needed. Profiling identified this as a bottleneck, in particular
for `MGET` and `HMGET`.

To be as fast as possible, Eredis takes a different approach. The
socket is always set to active once, which will let us receive data
fast without blocking the gen_server. The tradeoff is that we must
parse partial responses, which makes the parser more complex.

In order to make multibulk responses more efficient, the parser
will parse all data available and continue where it left off when more
data is available.

## Future improvements

When the parser is accumulating data, a new binary is generated for
every call to `parse/2`. This might create binaries that will be
reference counted. This could be improved by replacing it with an
iolist.

When parsing bulk replies, the parser knows the size of the bulk. If the
bulk is big and would come in many chunks, this could improved by
having the client explicitly use `gen_tcp:recv/2` to fetch the entire
bulk at once.

## Credits

Although this project is almost a complete rewrite, many patterns are
the same as you find in Erldis, most notably the queueing of requests.

`create_multibulk/1` and `to_binary/1` were taken verbatim from Erldis.
