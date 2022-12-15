# KiBot example for KiCost on GitHub CI/CD

This is an example of how to get component prices using KiBot and KiCost inside a GitHub CI/CD workflow.

This example also shows how to use GitHub caches to keep data between KiBot runs.

Note: Use KiCost 1.1.15 or newer

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

## The API secrets

In order to protect them this example uses *repository secrets*. In the current GitHub interface they can
be defined choosing `Settings` (in the main menu bar, where the code, issues, etc. are located), then
`Secrets` (bottom left), `Actions` and finally selecting `New repository secret`.
Here is the sequence in the [GitHub docs](https://docs.github.com/en/actions/security-guides/encrypted-secrets?tool=webui#creating-encrypted-secrets-for-a-repository).

The name of the secrets is arbitrary, but I recommend using the same name used for the associated environment
variable. The variables are named **DISTRIBUTOR**_**OPTION**, all in uppercase. As an example, the name for
the environment variable to set the Mouser key is **MOUSER_KEY**. So I recommend defining a repository secret
named **MOUSER_KEY**. It should contain the key you got from Mouser.

In this repo we define the following secrets:

- ELEMENT14_KEY
- MOUSER_KEY
- TME_TOKEN
- TME_APP_SECRET
- NEXAR_CLIENT_ID
- NEXAR_CLIENT_SECRET

Note that we left Digi-Key outside. This is because the current API plug-in needs user intervention.

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
        - Arrow
        - Farnell
```

Note that these are **distributors**, not APIs. Here we want *Digi-Key* prices, even when we don't have
access to it. In this case the prices will come from *Nexar* API. The same goes for *Arrow*. We'll try
to get *Mouser*, *TME* and *Farnell* (*Element14*) prices directly.

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
Disabling *KitSpace* is really important, you'll get the same information that you get from *Nexar*, but
you'll load *KitSpace*, and risk to get an error.

```yaml
      xlsx:
        kicost_api_disable:
          # Don't use KitSpace, we have keys
          - KitSpace
          # Digi-Key is tricky and we can't currently use it on CI/CD
          - Digi-Key
```

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
        NEXAR_CLIENT_ID: ${{ secrets.NEXAR_CLIENT_ID }}
        NEXAR_CLIENT_SECRET: ${{ secrets.NEXAR_CLIENT_SECRET }}
      run: |
        mkdir KiCost
        kibot -vvvv 2> KiCost/kibot.log
```

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
we got from the APIs using GitHub caches. The relevant step is:

```yaml
    - name: Cache KiCost data
      id: kicost-cache
      #uses: actions/cache@v3
      uses: set-soft/cache@main
      with:
        path: ~/kicost_cache
        key: kicost_cache
```

Which is used to create a cache that can be retrieved using the `kicost_cache` key. The cache will contain the
data from `~/kicost_cache`.

Note that we are using a forked repo, not the original *actions/cache@v3*. This is because the original action
doesn't store the results if the workflow fails. But we want to save the progress, so we want to store it
anyways. The *set-soft/cache@main* is a forked repo with the success condition removed.

This is all you need to make the KiCost cache persistent. If you need to invalidate the cache you can just
remove it from the [GitHub UI](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows).

## Example results

The `results` folder contains the results of two runs, one without cache, and the other using the cache
generated in the previous run.

## Troubleshooting

- Use the full debug information as shown above. The example workflow will create the artifacts even on fail,
  so you'll get KiBot logs.
- Error 26 means that something went wrong during KiCost call.
- Element14 seems to be returning 403 (Forbidden) in some cases. I guess this happens when we exceed the
  maximum queries per second. But I need confirmation, I can't exceed it from my local machine, located in
  Argentina.

## Conclusions

- Using GitHub secrets and environment variables you can hide your API secrets.
- You can still use a KiCost configuration, just avoid including the secrets.
- You can exploit the KiCost cache using GitHub caches.
- We can offload KitSpace by using other APIs.
