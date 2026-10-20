# qrl-genesis-generator

Create QRL consensus and execution layer testnet genesis files, and optionally serve
the generated files over HTTP.

Run the commands below from this repository's root on a Linux AMD64 machine with
Docker and Buildx installed. Choose the tested image or build locally, then follow
the generation examples.

## Use the tested image

The following AMD64 image was used for the Web3.js HTTP and WebSocket contract
integration tests. It supports 64-byte addresses and the five-argument deposit
contract, including `randao_commitment`, and includes `deposit`, `validator`, and
`qrysmctl`.

- Published tag: `qrledger/qrysm:qrl-genesis-generator-a3979a15c3bb-amd64`
- Generator source: [`cyyber/qrl-genesis-generator` at `4eceb8a5fb6a`](https://github.com/cyyber/qrl-genesis-generator/commit/4eceb8a5fb6a856be21667d63bff5fab3007dbdb)
- Qrysm tools and compatible beacon/validator clients: [`theQRL/qrysm` at `a3979a15c3bb`](https://github.com/theQRL/qrysm/commit/a3979a15c3bb726e04b7bddb5a2fc27e69134d9c)
- Compatible execution client: [`theQRL/go-qrl` at `f3864c46b3f9`](https://github.com/theQRL/go-qrl/commit/f3864c46b3f94662c514105da1efaf3466394fae)

Select the tested image by its [immutable registry digest](https://docs.docker.com/reference/cli/docker/image/pull/#pull-an-image-by-digest-immutable-identifier):

```sh
GENESIS_IMAGE=qrledger/qrysm@sha256:082bc6b0be7add21f1499ecb24d428b3a3b19bfb04ddd69b0cc2a65a2cffed06
docker pull --platform linux/amd64 "$GENESIS_IMAGE"
```

These source revisions describe the published image. New local builds use Qrysm's
`main` branch as shown below.

## Build locally from main

Build the generator from this checkout, with its Qrysm tools built from
`theQRL/qrysm`'s `main` branch:

```sh
docker buildx build \
  --platform linux/amd64 \
  --pull \
  --no-cache-filter builder \
  --build-arg QRYSM_GIT_REPO=https://github.com/theQRL/qrysm.git \
  --build-arg QRYSM_GIT_REF=main \
  --tag qrl-genesis-generator:local \
  --load \
  .

GENESIS_IMAGE=qrl-genesis-generator:local
```

`--load` makes the image available to local `docker run` commands. The
[`--no-cache-filter builder` option](https://docs.docker.com/reference/cli/docker/buildx/build/#no-cache-filter)
reruns the stage that fetches and compiles Qrysm, so rebuilding picks up changes to
`main`. The image is built locally; this command does not publish it.

## Generate genesis files

Create an output directory and a configuration file using
[defaults/defaults.env](defaults/defaults.env) as a template:

```sh
mkdir -p output config
if [ ! -f config/values.env ]; then
  cp defaults/defaults.env config/values.env
fi
```

Edit `config/values.env` for your testnet. With the tested mainnet-preset Qrysm
clients, `SLOTS_PER_EPOCH * EPOCHS_PER_EXECUTION_VOTING_PERIOD` must equal `512`.
The defaults are `128` and `4`; an eight-slot testnet must use `8` and `64`.

Use the `GENESIS_IMAGE` selected above to generate both EL and CL genesis:

```sh
docker run --rm --platform linux/amd64 --user "$(id -u):$(id -g)" \
  --volume "$PWD/output:/data" \
  --volume "$PWD/config/values.env:/config/values.env:ro" \
  "$GENESIS_IMAGE" all
```

Alternatively, generate the EL genesis first and then the CL genesis with the same
configuration and output directory. CL generation reads the EL genesis file.

```sh
docker run --rm --platform linux/amd64 --user "$(id -u):$(id -g)" \
  --volume "$PWD/output:/data" \
  --volume "$PWD/config/values.env:/config/values.env:ro" \
  "$GENESIS_IMAGE" el

docker run --rm --platform linux/amd64 --user "$(id -u):$(id -g)" \
  --volume "$PWD/output:/data" \
  --volume "$PWD/config/values.env:/config/values.env:ro" \
  "$GENESIS_IMAGE" cl
```

Generated genesis files are written under `output/metadata`. Existing genesis
files are skipped, so use a fresh output directory when changing the configuration.

## Environment variables

| Name | Default | Description |
| --- | --- | --- |
| SERVER_ENABLED | false | Serve the generated files over HTTP |
| SERVER_PORT | 8000 | HTTP server port |

To enable the server, add `-e SERVER_ENABLED=true -p 8000:8000` before the image
argument in a `docker run` command. The container stays running to serve the files.

Genesis settings are listed in [defaults/defaults.env](defaults/defaults.env).
Override them in your `config/values.env` file, which is loaded when the container
starts.
