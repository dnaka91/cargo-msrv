# Output format: json

The `json` output format is intended to be used as a machine-readable output, to be interpreted by tooling, altough
humans may also use it, as it provides the most detailed output of all supported output formats.

As described on the [output-formats](index.md) page, `cargo-msrv` reports the status of the program via
events. A processor transforms these events into their output-format. In case of the `json` output format, 
events are almost 1-on-1 serialized to json (there are a few exceptions), and then printed to `stderr`.
Each json serialized event ends with a newline. Each line thus represents a single serialized event. 

To use the `json` output format, run `cargo-msrv` with the `--output-format json` option. 
For example, if you want to find the MSRV, you could run `cargo msrv --output-format json`.


In the next section, you can find a description of the common fields of events.
The section thereafter gives an overview of each of the supported events, with for each event its event specific fields.

# Common fields on events

| name  | optional | values           | description                              |
|-------|----------|------------------|------------------------------------------|
| type  | no       |                  | Identifies a specific event              |
| scope | yes      | "start" or "end" | Marks the begin or end of a scoped event |

The **type** field can be used to identify a specific event.

The **scope** field is only present for scoped events. The `start` value marks the start of a scoped event, while `end`
marks the end of a scoped event. The scope is not an inherent property of the event itself. A scope adds a span during
which an event took place.


# Events

## Event: Meta

**type:** meta

**description:** Reports metadata about the currently running `cargo-msrv` program instance.

**fields:**

| name           | description                                                                                             |
|----------------|---------------------------------------------------------------------------------------------------------|
| instance       | Name of the running `cargo-msrv` program as defined on compile time. This will usually be `cargo-msrv`. |
| version        | Version of the running `cargo-msrv` program as defined on compile time.                                 |
| sha_short      | Short SHA hash of the git commit used to compile and build `cargo-msrv`.                                |
| target_triple  | Target triple of toolchain used to compile and build `cargo-msrv`.                                      |
| cargo_features | Features which were enabled during the compilation of `cargo-msrv`.                                     |
| rustc          | Version of `rustc` used to compile `cargo-msrv`.                                                        |

**example:**

```json lines
{"type":"meta","instance":"cargo-msrv","version":"0.15.1","sha_short":"79582b6","target_triple":"x86_64-pc-windows-msvc","cargo_features":"default,rust_releases_dist_source","rustc":"1.62.0"}
```

## Event: FetchIndex

**type:** fetch_index

**description:** Prior to determining the MSRV of a crate, we have to figure out which Rust versions are available. 
We obtain those using the [rust-releases](https://crates.io/crates/rust-releases) library. The `FetchIndex` event
reports that the index is being fetched, and details which source is used. 

**fields:**

| name           | description                                               |
|----------------|-----------------------------------------------------------|
| source         | Place from where the available Rust releases are obtained |

**example:**

```json lines
{"type":"fetch_index","source":"rust_changelog","scope":"start"}
{"type":"fetch_index","source":"rust_changelog","scope":"end"}
```

## Event: CheckToolchain

**type:** check_toolchain

**description:** The primary way for `cargo-msrv` to determine whether a given Rust toolchain is compatible with your
crate, is by installing a toolchain and using it to check a crate for compatibility with this toolchain. The
`CheckToolchain` event is wrapped around this process and notifies you about the start and end of this process.
This event is called as a scoped event, and within it's scope, you'll find the following events: `setup_toolchain`,
`check_method` and `check_result`, which are described in more detail below. 

**fields:**

| name              | description                              |
|-------------------|------------------------------------------|
| toolchain         | The toolchain to be located or installed |
| toolchain.version | The Rust version of the toolchain        |
| toolchain.target  | The target-triple of the toolchain       |

**example:**

```json lines
{"type":"check_toolchain","toolchain":{"version":"1.35.0","target":"x86_64-pc-windows-msvc"},"scope":"start"}
{"type":"check_toolchain","toolchain":{"version":"1.35.0","target":"x86_64-pc-windows-msvc"},"scope":"end"}
```

## Event: SetupToolchain

**type:** setup_toolchain

**description:** The primary way for `cargo-msrv` to determine whether a given Rust toolchain is compatible with your
crate, is by installing a toolchain and using it to check a crate for compatibility with this toolchain. The
`SetupToolchain` event reports about the process of locating or installing a given toolchain.

**fields:**

| name              | description                              |
|-------------------|------------------------------------------|
| toolchain         | The toolchain to be located or installed |
| toolchain.version | The Rust version of the toolchain        |
| toolchain.target  | The target-triple of the toolchain       |

**example:**

```json lines
{"type":"setup_toolchain","toolchain":{"version":"1.47.0","target":"x86_64-pc-windows-msvc"},"scope":"start"}
{"type":"setup_toolchain","toolchain":{"version":"1.47.0","target":"x86_64-pc-windows-msvc"},"scope":"end"}
```

## Event: CheckMethod

**type:** check_method

**description:** Reports which method has been used to check whether a toolchain is compatible with a crate.

**fields:**

| name              | optional | condition                  | description                                |
|-------------------|----------|----------------------------|--------------------------------------------|
| toolchain         | no       |                            | The toolchain to be located or installed   |
| toolchain.version | no       |                            | The Rust version of the toolchain          |
| toolchain.target  | no       |                            | The target-triple of the toolchain         |
| method            | no       |                            | The method used to check for compatibility |
| method.type       | no       |                            | The type of method                         |
| method.args       | no       | method.type = `rustup_run` | The arguments provided to rustup           |
| method.path       | yes      | method.type = `rustup_run` | The path provided to rustup, if any        |


**example:**

```json lines
{"type":"check_method","toolchain":{"version":"1.37.0","target":"x86_64-pc-windows-msvc"},"method":{"rustup_run":{"args":["1.37.0-x86_64-pc-windows-msvc","cargo","check"],"path":"..\\air3\\"}}}
```

## Event: CheckResult

**type:** check_result

**description:** Reports the result of a `cargo-msrv` compatibility check.

**fields:**

| name              | optional | condition                       | description                                        |
|-------------------|----------|---------------------------------|----------------------------------------------------|
| toolchain         | no       |                                 | The toolchain to be located or installed           |
| toolchain.version | no       |                                 | The Rust version of the toolchain                  |
| toolchain.target  | no       |                                 | The target-triple of the toolchain                 |
| is_compatible     | no       |                                 | Boolean value stating compatibility                |
| report            | no       |                                 | Report about the compatibility                     |
| report.type       | no       |                                 | Type of the report, "compatible" or "incompatible" |
| report.error      | yes      | if report.type = `incompatible` | Error message of the compatibility check           |

**example:**

```json lines
{"type":"check_result","toolchain":{"version":"1.38.0","target":"x86_64-pc-windows-msvc"},"is_compatible":true,"report":{"type":"compatible"}}
```

```json lines
{"type":"check_result","toolchain":{"version":"1.37.0","target":"x86_64-pc-windows-msvc"},"is_compatible":false,"report":{"type":"incompatible","error":"error: failed to parse lock file at: ..\\air3\\Cargo.lock\n\nCaused by:\n  invalid serialized PackageId for key `package.dependencies`\n"}}
```

## Event: AuxiliaryOutput

**type:** auxiliary_output

**description:** Reports about additional output written by `cargo-msrv` when applicable. For example, if the
`--write-msrv` or `--write-toolchain-file` flag is provided, the MSRV will be written to the Cargo manifest or the
Rust toolchain file respectively. The act of writing this (additional) output is reported by this event.

**fields:**

| name             | optional | condition                       | description                                                                                      |
|------------------|----------|---------------------------------|--------------------------------------------------------------------------------------------------|
| destination      | no       |                                 | The destination of the auxiliary output                                                          |
| destination.type | no       |                                 | Type of destination, currently only "file"                                                       |
| destination.path | no       | if destination.type = `file`    | Path of the written or amended file                                                              |
| item             | no       |                                 | What kind of output is written                                                                   |
| item.type        | no       |                                 | Type of output item                                                                              |
| item.kind        | no       | if item.type = `msrv`           | To which field the MSRV was written in the Cargo manifest, "rust-version" or "metadata_fallback" |
| item.kind        | no       | if item.type = `toolchain_file` | Which toolchain file kind was written, "legacy" or "toml"                                        |


**example:**

```json lines
{"type":"auxiliary_output","destination":{"type":"file","path":"..\\air3\\Cargo.toml"},"item":{"type":"msrv","kind":"rust_version"}}
```

## Event: Progress

## Event: SubcommandInit

## Event: SubcommandResult

## Event: TerminateWithFailure


# Output by subcommand

## \# cargo msrv (find)

## \# cargo msrv list

## \# cargo msrv set

## \# cargo msrv show

## \# cargo msrv verify