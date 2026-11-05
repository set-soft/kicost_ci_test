# KiBot example for KiCost on GitHub CI/CD

This is an example of how to get component prices using KiBot and KiCost inside a GitHub CI/CD workflow.

This example also shows how to use GitHub caches to keep data between KiBot runs.

> [!NOTE]
> - KitSpace and Nexar are no longer recommended.
> - Digi-Key can be used from CI/CD now.
> - GitHub changed the cache mechanism, here you'll find a working mechanism.

## The problematic

When running KiBot locally you can just use your KiCost configuration, or even provide an alternative file.
But when running on a public GitHub repo this approach has a problem: your API secrets must be available
in the KiCost configuration file. This exposes your secrets to anybody with read access to the repo.

One solution is to use *repository secrets*. In the GitHub case you can simply use
[action secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets).
This repo shows how to use them.

## The test case

To ilustrate the mechanism this repo contains a simple Arduino programmer, the same used to show
[variants](https://inti-cmnb.github.io/kibot_variants_arduprog/).

In this case we added the following fields to the schematic components:

- **manf**: The name of the manufacturer for the component
- **manf#**: The name used by the manufacturer for this component

If you are going to buy the components from an specific distributor I recommend also adding a field named
**DISTRIBUTOR#**, containing the code used by the distributor. As an example we added **digikey#**.

## KiCost configuration file

You can fine tune the KiCost APIs using a configuration file. This is the same configuration you use in your
desktop machine, but without any sensitive data. In this example we set the cache TTL to 1 day, just for
demonstration, and also selected some values for the *TME* API, again just to show how to do it.

The configuration example is [kicost_config.yaml](kicost_config.yaml)

In the above example you'll notice that we only list the APIs that needs some customization, the rest uses
default values. Also note that no *key* or *token* is used in the configuration file, they will be defined
using environment variables. Secrets for this repo.

> [!NOTE]
> In order to use Digi-Key you must use the V4 API

## The API secrets

In order to protect them this example uses *repository secrets*. In the current GitHub interface they can
be defined choosing `Settings` (in the main menu bar, where the code, issues, etc. are located), then
`Secrets and variables` (bottom left), `Actions` and finally selecting `New repository secret`.
Here is the sequence in the [GitHub docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets?tool=webui#creating-encrypted-secrets-for-a-repository).

The name of the secret is arbitrary, but I recommend using the same name used for the associated environment
variable. The variables are named *DISTRIBUTOR*_*OPTION*, all in uppercase. As an example, the name for
the environment variable to set the Mouser key is **MOUSER_KEY**. So I recommend defining a repository secret
named **MOUSER_KEY**. It should contain the key you got from Mouser.

In this repo we define the following secrets:

- ELEMENT14_KEY
- MOUSER_KEY
- TME_TOKEN
- TME_APP_SECRET
- DIGIKEY_CLIENT_ID
- DIGIKEY_CLIENT_SECRET

## KiBot configuration

Here is the full configuration: [t1.kibot.yaml](t1.kibot.yaml).

Consult the KiBot documentation for details, here I explain the most relevant parts:

### Selecting the distributors

You can select which distributors will be included in the spreadsheet using:

```yaml
      distributors:
        - Mouser
        - Digi-Key
        - TME
        - Farnell
```

Note that these are **distributors**, not APIs. Here we want *Farnell* prices, provided by *Element14*. We'll try
to get *Mouser*, *TME*, *Digi-Key* and *Farnell* (*Element14*) prices directly.

### Enabling KiCost use

```yaml
      xlsx:
        kicost: true
```

This will add the *Cost* sheets.

### Enabling specifications sheet

Some APIs can provide interesting information about the components, they will be stored in the *Specs* sheets.
To enable it use:

```yaml
      xlsx:
        specs: true
```

Note that you can filter this information, but you can start getting all the information.

### APIs detail

As we mentioned before you can fine tune the APIs using a KiCost configuration. Then you pass it to KiBot:

```yaml
      xlsx:
        kicost_config: kicost_config.yaml
```

### Saving time and avoiding problems

In order to avoid long time-outs, or even fails, you should disable APIs that you don't use.

## Workflow configuration

The full configuration for this example can be found here: [.github/workflows/kibot.yml](.github/workflows/kibot.yml)

The most relevant stuff is discussed here.

### Passing the secrets to KiCost/KiBot

Here we define environment variables to hold the secrets. Then we start KiBot.

```yaml
    - name: Run KiBot
      env:
        ELEMENT14_KEY: ${{ secrets.ELEMENT14_KEY }}
        MOUSER_KEY: ${{ secrets.MOUSER_KEY }}
        TME_TOKEN: ${{ secrets.TME_TOKEN }}
        TME_APP_SECRET: ${{ secrets.TME_APP_SECRET }}
        DIGIKEY_CLIENT_ID: ${{ secrets.DIGIKEY_CLIENT_ID }}
        DIGIKEY_CLIENT_SECRET: ${{ secrets.DIGIKEY_CLIENT_SECRET }}
      run: |
        mkdir KiCost
        kibot -vvvv 2> KiCost/kibot.log
```
As the name of the secrets is the name used for the environment variables the
mapping is trivial.

In this example we store full debug information in *KiCost/kibot.log*.
Note that KiCost will try to avoid dumping the keys to the debug file.
In this example none is found in the log.

For better security run KiBot with debug information only when needed.

### Using a cache

The default cache TTL is one week, we changed it to one day just for testing purposes.
The KiCost configuration says:

```yaml
# KiCost configuration file
kicost:
  version: 1
  # Prices are valid for a day
  cache_ttl: 1
  # We will store the cache here:
  cache_path: ~/kicost_cache
```

So caches will be stored at `~/kicost_cache`. The problem here is that this will go away as soon as the
CI/CD workflow finishes. This will defeat the whole idea. In order to make it work we store the information
we got from the APIs using GitHub caches. Before executing KiBot we restore the cached data using:

```yaml
    - name: Restore cached KiCost data
      id: kicost-cache-restore
      uses: actions/cache/restore@v5
      with:
        # This path must match the one declared in kicost_config.yaml
        path: ~/kicost_cache
        # The key must be unique, this will be the name used to save, not restore
        key: kicost_cache-${{ github.run_id }}-${{ github.run_attempt }}
        # We restore the newest available
        restore-keys: |
          kicost_cache
```

After running KiBot we update the cache using:

```yaml
    - name: Save cached KiCost data
      # Refresh the components cache
      id: kicost-cache-save
      if: always()
      uses: actions/cache/save@v5
      with:
        # This path must match the one declared in kicost_config.yaml
        path: ~/kicost_cache
        # Unique name
        key: kicost_cache-${{ github.run_id }}-${{ github.run_attempt }}
```

This will always update the cache, even if KiBot failed during the operation, we always keep
retrieved components.

The caches are deleted after some time by GitHub, only the last is kept. If you want to do
it manually to avoid using extra space you can add:

```yaml
    - name: Delete old cache entry
      # This is optional, caches are deleted automatically after some time
      if: >-
        always() &&
        steps.kicost-cache-save.outcome == 'success' &&
        steps.kicost-cache-restore.outputs.cache-matched-key != ''
      env:
        GH_TOKEN: ${{ github.token }}
      run: |
        curl -fsS -X DELETE \
          -H "Authorization: Bearer $GH_TOKEN" \
          -H "Accept: application/vnd.github+json" \
          "https://api.github.com/repos/${{ github.repository }}/actions/caches?key=${{ steps.kicost-cache-restore.outputs.cache-matched-key }}&ref=${{ github.ref }}" \
          || true
```

The name of the cache is `kicost_cache` followed by the run and retry ids. The cache will contain the
data from `~/kicost_cache`.

This is all you need to make the KiCost cache persistent. If you need to invalidate the cache you can just
remove it from the [GitHub UI](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows).

## Example results

The `results` folder contains the results of two runs, one without cache, and the other using the cache
generated in the previous run.

## Troubleshooting

- Use the full debug information as shown above. The example workflow will create the artifacts even on fail,
  so you'll get KiBot logs.
- Error 26 means that something went wrong during KiCost call.

## Conclusions

- Using GitHub secrets and environment variables you can hide your API secrets.
- You can still use a KiCost configuration, just avoid including the secrets.
- You can exploit the KiCost cache using GitHub caches.
