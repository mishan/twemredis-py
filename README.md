# twemredis-py

> **Archived.** This library targets [twemproxy](https://github.com/twitter/twemproxy)-style
> sharded Redis, which was how you built a Redis cluster before Redis Cluster was
> production-ready. That era is over, and this code has not been maintained since 2016.
> It is preserved for the technique documented below, which is still useful — see
> [Reviving the idea](#reviving-the-idea).

Originally built at if(we) to drive a fleet of md5-sharded Redis instances behind
twemproxy. It reimplements twemproxy's key distribution in Python so application
code can talk to the shards directly — bypassing the proxy for operations twemproxy
could not route, while still landing every key on the shard the proxy would have
chosen.

## What it did

Three things, in rough order of how much they mattered:

1. **Reimplemented twemproxy's md5 distribution.** `get_shard_num_by_key_id` is a
   direct port of `nc_md5.c`: take the md5 of the key id, read the first four bytes
   as a little-endian u32, modulo the shard count. Any key routed by this library
   lands where twemproxy would have put it.

2. **Fanned operations out across shards.** `keys`, `mget`, and `mset` batch by
   destination shard and issue the minimum number of round trips.
   `run_on_all_shards` and `execute_on_all_shards` broadcast anything else. Scanning
   commands (`scan`, `hscan`, `sscan`, `zscan`) are explicitly disallowed, since
   their cursors are meaningless across shards.

3. **Computed canonical keys.** This is the interesting part, and the reason the
   repo is still here.

## The canonical key technique

The problem: you have a set that every shard needs to read — a blocklist, a set of
canceled records, a feature flag cohort. If you store it once, every read on every
shard becomes a cross-shard round trip. What you want is a *copy on every shard*,
addressable without knowing which shard you're on.

Hash tags give you the addressing primitive. twemproxy (and later Redis Cluster)
hash only the substring inside `{}`, so `canceled:{12345}` routes on `12345`. That
means the key id is a knob you can turn to steer a key onto a chosen shard.

So: for each shard, find a key id that lands on it. Concretely, walk the integers
from 1 upward and keep the first one that maps to each shard:

```python
def compute_canonical_key_ids(self, search_amplifier=100):
    canonical_keys = {}
    for key_id in range(1, (self.num_shards() ** 2) * search_amplifier):
        shard_num = self.get_shard_num_by_key(str(key_id))
        if shard_num in canonical_keys:
            continue
        canonical_keys[shard_num] = str(key_id)
        if len(canonical_keys) == self.num_shards():
            break
    return canonical_keys
```

That yields a table like `{0: '3', 1: '7', 2: '1', ...}` — one *canonical key id*
per shard, deterministic given the shard count. Now:

```python
tr.get_canonical_key('canceled', '5430406808')   # -> 'canceled:{12}'
```

`5430406808` hashes to some shard; the canonical id for that shard is `12`; so
`canceled:{12}` is guaranteed to live on the same shard as `5430406808`. Write the
set under all N canonical keys, and every read is shard-local no matter which key
you started from.

Redis Cluster has no equivalent. It gives you hash tags, but it does not give you
a stable way to enumerate "one key per node," because node membership changes with
resharding.

## Reviving the idea

The technique generalizes to Redis Cluster and Valkey, but **not** in the obvious
way. The naive port — compute one canonical key per *node* — is genuinely unsafe:
slot ownership moves during resharding, and a node that gains slots but no canonical
key is silently blind. Simulating random resharding events, the per-node approach
loses coverage on **96.7%** of topology changes.

The fix is to pin canonical keys to *slots* rather than nodes. There are exactly
16384 slots, the CRC16 that maps keys to them is frozen by the cluster spec, and
Redis migrates keys along with their slot. Precompute one canonical tag per slot —
a permanent 16384-entry table — and coverage becomes a structural guarantee rather
than something you hope survives the next reshard.

See `SPEC.md` in the successor project for the full design, the generated slot
table, and the correctness argument.

## Known defects

Left as-is; noted so nobody mistakes them for working code.

- `init_shards_from_masters` references `redis.StrictRedis`, but only
  `redis.sentinel.Sentinel` is imported. The non-Sentinel path raises `NameError`
  and has never worked.
- `yaml.load()` is called without a `Loader`, which is an error on PyYAML ≥ 6.
- Several error paths mix `%`-style and `.format()` placeholders
  (`"...%d...".format(...)`), producing garbled messages.
- The Sentinel port `8422` is hardcoded inside `init_shards_from_sentinel`.
- `run_on_all_shards` tests `if method is not None` after `getattr`, which raises
  rather than returning `None` — the branch is dead.
- Python 2-era class declaration and test tooling (`mockredispy`).

## License

Apache 2.0. See `LICENSE`.
