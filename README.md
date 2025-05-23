# rapid-gossip-sync-server

This is a server that connects to peers on the Lightning network and calculates compact rapid sync
gossip data.

These are the components it's comprised of.

## Demo Server

To experiment with a local instance of the RGS server, navigate to the cache directory specified using
the `RAPID_GOSSIP_SYNC_SERVER_CACHES_PATH` environment variable (`./res` by default) and therein to the `symlinks`
subdirectory:

```
<cache_path>/symlinks
```

From there, simply run:

```shell
python3 -m http.server 8011
```

The snapshots will be accessible via `http://localhost:8011/{timestamp}.bin`. The first timestamp value should be 0,
and subsequent timestamps will be extracted from the snapshots themselves by the RGS client.

Note that this is pretty much the same functionality you would need in a production environment, except hopefully using
something more robust than Python's `http.server` module.

## Modules

### config

A config file where the Postgres credentials and Lightning peers can be adjusted. Most adjustments
can be made by setting environment variables, whose usage is as follows:

| Name                                       | Default             | Description                                                                                                |
|:-------------------------------------------|:--------------------|:-----------------------------------------------------------------------------------------------------------|
| RAPID_GOSSIP_SYNC_SERVER_DB_HOST           | localhost           | Domain of the Postgres database                                                                            |
| RAPID_GOSSIP_SYNC_SERVER_DB_USER           | alice               | Username to access Postgres                                                                                |
| RAPID_GOSSIP_SYNC_SERVER_DB_PASSWORD       | _None_              | Password to access Postgres                                                                                |
| RAPID_GOSSIP_SYNC_SERVER_DB_NAME           | ln_graph_sync       | Name of the database to be used for gossip storage                                                         |
| RAPID_GOSSIP_SYNC_SERVER_NETWORK           | mainnet             | Network to operate in. Possible values are mainnet, testnet, signet, regtest                               |
| RAPID_GOSSIP_SYNC_SERVER_SNAPSHOT_INTERVAL | 10800               | The interval in seconds between snapshots                                                                  |
| BITCOIN_REST_DOMAIN                        | 127.0.0.1           | Domain of the [bitcoind REST server](https://github.com/bitcoin/bitcoin/blob/master/doc/REST-interface.md) |
| BITCOIN_REST_PORT                          | 8332                | HTTP port of the bitcoind REST server                                                                      |
| BITCOIN_REST_PATH                          | /rest/              | Path infix to access the bitcoind REST endpoints                                                           |
| LN_PEERS                                   | _Wallet of Satoshi_ | Comma separated list of LN peers to use for retrieving gossip                                              |

### downloader

The module responsible for initiating the scraping of the network graph from its peers.

### persistence

The module responsible for persisting all the downloaded graph data to Postgres.

### snapshot

The snapshotting module is responsible for calculating and storing snapshots. It's started up
as soon as the first full graph sync completes, and then keeps updating the snapshots at a
configurable interval with a 3-hour-default.

### lookup

The lookup module is responsible for fetching the latest data from the network graph and Postgres,
and reconciling it into an actionable delta set that the server can return in a serialized format.

It works by collecting all the channels that are currently in the network graph, and gathering
announcements as well as updates for each of them. For the updates specifically, the last update
seen prior to the given timestamp, the latest known updates, and, if necessary, all intermediate
updates are collected.

Then, any channel that has only had an announcement but never an update is dropped. Additionally,
every channel whose first update was seen after the given timestamp is collected alongside its
announcement.

Finally, all channel update transitions are evaluated and collected into either a full or an
incremental update.

## License

[Apache 2.0](LICENSE-APACHE.md) or [MIT](LICENSE-MIT.md), [at your option](LICENSE.md).
