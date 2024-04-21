# Hub Shuttle Package

Package to help move data from a hub to a database.

Currently in beta. APIs should be relatively stable, but still subject to change.

However, database schema is stable and will not change (except for addition of new tables).

## Stephan's contributions

- Batch messages by type and fid for backfilling
- Benchmark different values for concurrency (see `benchmark_results_1000_combined.csv`)
- Sets a high `query_timeout` for DB connection and increases the `max` pool size to 20 to prevent dropping inserts due to connection timeout
- Benchmarks
  - 23x speedup for backfilling FID 3
  - Backfills first 2000 FIDs in 312 seconds with CONCURRENCY=1
  - Backfills first 1000 FIDs in 80 seconds with CONCURRENCY=8

* Tests run on an MacBook M1 Pro

```
docker-compose down -v && docker-compose up -d && FIDS=3 yarn start bench-reconcile
```

```
docker-compose down -v && docker-compose up -d && MAX_FID=1000 CONCURRENCY=8 yarn start bench-backfill
```

## Architecture

![Architecture](./architecture.jpg)

There are 3 main components:

- a hub subscriber that listens to the hub for new messages
- an event processor that can process messages and write them to the database
- a reconciler that can backfill messages and ensures the messages are in sync with the hub

The event processor has a hook for your code to plug into the message processing loop. You must implement the `MessageHandler` interface
and provide it to the system. The system will call your `handleMessageMerge` method for each message it processes, within the same transaction
where the message is written to the db. The function is always expected to succeed, it is not possible to instruct it to "skip" a message.
If your function fails, the message will not be written to the db and will be retried at a later time.

## Usage

The package is meant to be used as a library. The app provided is just an example.

If you want to run the test the app, do the following:

```bash

# Ensure you have node 21 installed, use nvm to install it

# Within the package directory
yarn install && yarn build

# Start the dependencies
docker compose up postgres redis

# To perform reconciliation/backfill, start the worker (can run multiple processes to speed this up)
POSTGRES_URL=postgres://shuttle:password@0.0.0.0:6541 REDIS_URL=0.0.0.0:16379 HUB_HOST=<host>:<port> HUB_SSL=false yarn start worker

# Kick off the backfill process (configure with MAX_FID=100 or BACKFILL_FIDS=1,2,3)
POSTGRES_URL=postgres://shuttle:password@0.0.0.0:6541 REDIS_URL=0.0.0.0:16379 HUB_HOST=<host>:<port> HUB_SSL=false yarn start backfill

# Start the app and sync messages from the event stream
POSTGRES_URL=postgres://shuttle:password@0.0.0.0:6541 REDIS_URL=0.0.0.0:16379 HUB_HOST=<host>:<port> HUB_SSL=false yarn start start
```

If you are using this as a package implement the `MessageHandler` interface and take a look at the `App` class on how to call the other classes.

## TODO

- [ ] Onchain events and fnames
- [ ] Detect if already backfilled and only backfill if not
- [ ] Better retries and error handling
- [ ] More tests
