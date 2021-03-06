# statetrack_plugin

This plugin adds hooks on chainbase operations emplace, modify, remove, and undo,
and routes the objects and revision numbers to a ZMQ socket.
This approach has different purposes:

- Allowing developers to get fine-grained access to the state DB operations.
  - To mirror the state DB into a traditional database system such as MongoDB
    or Postgres without them having to implement any further logic or duplicate
    the DB write logic of their smart contracts.
  - To execute side-effects associated to specific database operations rather than
    specific contract actions.
  - To propagate database operations to the dapp frontends through WebSockets
    for enabling real-time updates to user interfaces.
- Seamlessly handling any forks and transaction failures by leveraging nodeos' own
  behavior in maintaining the consistency of its own databases through
  undo and commit operations.
- Minimizing the work that happens inside nodeos and letting receivers such as
  [statemirror](https://github.com/andresberrios/statemirror) or a demux-js reader
  handle the more complex logic and more performance intensive work.

The plugin can hook to the operations that happen on the state DBs of all contracts
deployed to the chain, and it can use filters to limit the operations
that should be sent to only the ones happening on the contracts, scopes, and tables
that the user is interested in tracking, thus limiting the performance impact to a minimum.

The plugin will also send operations happening on accounts and account permissions.
It will also send a stream of applied actions.

This is a very early stage of the implementation, and more testing is required.
This should not be considered production ready yet.

## Configuration

The following configuration statements in `config.ini` are recognized:

- `plugin = eosio::statetrack_plugin` -- Enables the state track plugin
- `st-zmq-sender-bind = ENDPOINT` -- Specifies the PUSH socket connection endpoint.
  Default value: tcp://127.0.0.1:3000.
- `st-filter-on = code:scope:table` -- Track DB operations which match code:scope:table.
- `st-filter-out = code:scope:table` -- Do not track DB operations which match code:scope:table.

## Important notes

- Accounts are sent as DB operations with `(code, scope, table)` being
  `(system, system, accounts)`.
  - Disable tracking of accounts using `st-filter-out = system:system:accounts`.
- Account permissions are sent as DB operations with `(code, scope, table)`
  being `(system, system, permissions)`.
  - Disable tracking of account permissions using `st-filter-out = system:system:permissions`.
- Actions are sent as DB operations with `(code, scope, table)` being
  `(system, system, actions)`.
  - Disable tracking of actions using `st-filter-out = system:system:actions`.
- Filters are not tested yet, but the implementation is taken from
  the history plugin, so it should work fine.
- We're adding separate filters for actions to make it possible to
  filter by specific actions of specific contracts.

## Open questions

- In Chainbase, we are emitting the operations inside an undo just before the
  operation is actually executed and checked for success
  ([See here](https://github.com/EOSIO/chainbase/compare/master...mmcs85:master#diff-298563b6c76ef92100c2ea27c06cb08bR390)).
  - We would like advice on the best way (without degrading performance) to get
    the item after it is given to the `modify` and `emplace` functions using `std::move`.
  - We don't like this inconsistency since it is emitting the event before being
    sure that the operation actually took place, but we think it might not
    be an issue since those operations are undoing previous ones and it
    would not be an expected scenario for them to fail unless there was a bug
    or a database inconsistency (maybe caused by bad locking mechanisms when
    using multiple threads?).
- Performance
  - Serialization options
    - JSON serializing on the plugin
    - Sending binary and decoding to JSON in the receiver
      - Would need to fetch ABIs, which would make it slower overall but put less load on nodeos
    - Place operations in a queue during transaction processing
      - Process queue, serialize each message to JSON and send to receiver
        - After transaction is processed (prevents failed transactions from sending do and undo operations)
        - During transaction processing but in a separate thread
  - Risk of making transaction processing take longer and hit CPU time limit
    - Only for transactions sent directly to this node?
    - Does it affect validated transactions/blocks received from the network?
- Consensus
  - Forks
    - How to induce and test recovery
  - Exceptions
  - Other potential disruptions
- Socket communications
  - In the current implementation, sending a message blocks until it is received (ZMQ PUSH socket)
  - This is useful in case receiver goes down, since nodeos will wait for it to come back up
    - Instead of maintaining a local in-memory queue of messages that can build up too much if it carries on
    - However, it will make nodeos unavailable for API calls, and maybe other side-effects
  - Lost messages
    - When receiver pulls a message from the socket but then crashes
      - Implement message acknowledgement signal?
      - Implement an intermediate receiver that only manages the queue?
      - Implement a very safe persistent received-messages queue in receiver from which to recover if it crashes?

## Building

The plugin needs to be built using the ZeroMQ C++ headers.

### Adding ZMQ library on Mac

```
brew install zmq
brew tap jmuncaster/homebrew-header-only
brew install jmuncaster/header-only/cppzmq
```

### Adding ZMQ library on Ubuntu

```
apt-get install -y pkg-config libzmq5-dev
```
