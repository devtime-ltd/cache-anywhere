# cache-anywhere

`actions/cache` with a backend switch. Pass `backend: local` to cache against a
self-hosted S3 store (Garage, MinIO, ...) on your runner network; pass anything
else to use GitHub's cache. Same `path` / `key` / `restore-keys` interface as
`actions/cache`, so it drops in.

Built for self-hosted fleets that want NVMe-speed caching on the fleet but must
degrade to GitHub's cache when a job runs on a hosted/fallback runner. Pairs
with [runner-failover](https://github.com/devtime-ltd/runner-failover): its
`used-fallback` output tells you which backend to pick.

## Usage

```yaml
jobs:
  runner:
    uses: ./.github/workflows/determine-runner.yml   # runner-failover wrapper
    secrets: { RUNNER_READ_PAT: "${{ secrets.RUNNER_READ_PAT }}" }

  build:
    needs: runner
    runs-on: ${{ fromJson(needs.runner.outputs.runner) }}
    steps:
      - uses: actions/checkout@v7
      - uses: devtime-ltd/cache-anywhere@v1
        with:
          path: vendor
          key: composer-${{ hashFiles('composer.lock') }}
          backend: ${{ needs.runner.outputs.backend }}   # "local" on the fleet, "github" on fallback
          s3-access-key: ${{ secrets.CI_CACHE_ACCESS_KEY }}
          s3-secret-key: ${{ secrets.CI_CACHE_SECRET_KEY }}
```

On the fleet, `backend` is `local` and caching hits the S3 store at
`s3-endpoint:s3-port` (defaults `garage:3900`, reachable only inside the runner
network). On a fallback runner, `backend` is `github` and it uses
`actions/cache`. Nothing else in the job changes.

## Inputs

| Input | Default | Notes |
| --- | --- | --- |
| `path` | (required) | Same as `actions/cache`. |
| `key` | (required) | Same as `actions/cache`. |
| `restore-keys` | | Same as `actions/cache`. |
| `backend` | (required) | `local` = S3; anything else = GitHub cache. |
| `s3-endpoint` | `garage` | Host of the local S3 store. |
| `s3-port` | `3900` | Port of the local S3 store. |
| `s3-insecure` | `true` | Plain HTTP (internal network). |
| `s3-bucket` | `ci-cache` | Bucket name. |
| `s3-access-key` / `s3-secret-key` | | Local S3 credentials (from secrets). |

## Notes

- The local backend wraps [`tespkg/actions-cache`](https://github.com/tespkg/actions-cache);
  pin it to a commit SHA in `action.yml` before production use.
- The local S3 store should live on the isolated runner network only, never
  exposed to the LAN.
- If the local store is unreachable, the job sees a cache miss (not a failure)
  and proceeds; correctness never depends on the cache.

## License

MIT
