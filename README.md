# pulumi-stripe

Pulumi SDKs for [Stripe](https://stripe.com), generated from
[`stripe/terraform-provider-stripe`](https://github.com/stripe/terraform-provider-stripe)
via Pulumi's Terraform provider bridge.

This repository does exactly one thing: it watches the upstream provider for new
releases, regenerates the SDKs, and tags and releases the result under the same
version. Tag `v0.2.3` here contains the SDKs generated from upstream `v0.2.3`.
Everything under `sdks/` is generated — never edit it by hand.

## Using the SDKs

The supported way to consume the Stripe provider is to let Pulumi generate the
SDK into your own project, which is the same command this repository automates:

```bash
pulumi package add terraform-provider stripe/stripe
```

Pass a version as a trailing argument to pin to a specific upstream release:

```bash
pulumi package add terraform-provider stripe/stripe 0.2.3
```

The Go SDK is the exception: it is a real module and can be depended on
directly, which spares a consumer from vendoring it behind a `replace`.

```bash
go get github.com/gvtlabs/pulumi-stripe/sdks/go@v0.2.3
```

```go
import "github.com/gvtlabs/pulumi-stripe/sdks/go/stripe"
```

The bridge stamps its own canonical path
(`github.com/pulumi/pulumi-terraform-provider/sdks/go/stripe`) into every Go SDK
it generates, which matches no repository and so resolves nowhere. The sync
rewrites it to this one; see below.

The other four SDKs are not published to PyPI, npm, NuGet, or Maven. For those,
treat this repository as a versioned, reviewable record of what the bridge
produces: useful for diffing the generated API surface between provider
releases, auditing what lands in your stack before you upgrade, and vendoring.

## Configuration

The provider takes the same credential as the Terraform provider — a Stripe
secret API key:

```bash
pulumi config set --secret stripe:apiKey <STRIPE_SECRET_KEY>
```

See the [upstream provider documentation](https://registry.terraform.io/providers/stripe/stripe/latest/docs)
for the full resource and data source reference. Resource names are the
Terraform names in each language's idiomatic casing, so `stripe_billing_meter`
becomes `BillingMeter`.

## How the sync works

[`.github/workflows/sync.yml`](.github/workflows/sync.yml) runs
[`scripts/sync.sh`](scripts/sync.sh) hourly. The script reads the latest upstream
release, exits early if a matching tag already exists, and otherwise regenerates
all five SDKs, commits them, and creates the mirroring tags. The workflow then
publishes a GitHub release for the `v<version>` tag, which is what a consumer
can subscribe to; a tag on its own notifies nobody. Git tags are the only state
the sync decision rests on, so the job is idempotent and needs no external
bookkeeping.

Two steps exist purely to keep the Go SDK consumable. The script rewrites the
bridge's canonical module path to this repository's and fails loudly if any
occurrence survives, so a change in the bridge's layout is caught rather than
silently publishing an unresolvable module. It then writes two tags per release:
`v<version>`, which names the release, and `sdks/go/v<version>`, which is the
only form the Go module proxy accepts for a module in a subdirectory. Without
the second, `go get` can see the module but resolve no versions of it.

GitHub does not allow one repository to subscribe to another repository's release
events, so polling is the only trigger available to us without cooperation from
the upstream maintainers. The workflow also accepts a `repository_dispatch` event
of type `upstream-release` if you ever want to wire up a relay and skip the
polling delay:

```bash
gh api repos/gvtlabs/pulumi-stripe/dispatches \
  -f event_type=upstream-release \
  -F 'client_payload[version]=0.2.3'
```

One wrinkle the script handles: the bridge resolves the provider from
`registry.opentofu.org`, which can lag a GitHub release by minutes or hours. When
the release exists upstream but is not yet resolvable, the script exits `75` and
the workflow reports a neutral notice instead of a failure, leaving the next
scheduled run to pick it up. The script also verifies that the SDK it generated
reports the version that was asked for, so a registry fallback to an older
provider can never be committed under the wrong tag.

## Running it locally

```bash
./scripts/sync.sh                     # sync the latest upstream release
./scripts/sync.sh --version 0.2.2     # backfill a specific version
./scripts/sync.sh --no-commit         # regenerate without committing
./scripts/sync.sh --force             # regenerate even if the tag exists
```

Requires the [Pulumi CLI](https://www.pulumi.com/docs/install/); no language
toolchains are needed, since the SDKs are generated but never built.

## License

MIT, matching the upstream provider. See [LICENSE](LICENSE).
