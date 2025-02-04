# Echidna: A Fast Smart Contract Fuzzer <a href="https://raw.githubusercontent.com/crytic/echidna/master/echidna.png"><img src="https://raw.githubusercontent.com/crytic/echidna/master/echidna.png" width="75"/></a>

![Build Status](https://github.com/crytic/echidna/workflows/CI/badge.svg)

Echidna is a weird creature that eats bugs and is highly electrosensitive (with apologies to Jacob Stanley)

More seriously, Echidna is a Haskell program designed for fuzzing/property-based testing of Ethereum smarts contracts. It uses sophisticated grammar-based fuzzing campaigns based on a [contract ABI](https://solidity.readthedocs.io/en/develop/abi-spec.html) to falsify user-defined predicates or [Solidity assertions](https://solidity.readthedocs.io/en/develop/control-structures.html#id4). We designed Echidna with modularity in mind, so it can be easily extended to include new mutations or test specific contracts in specific cases.

## Features

* Generates inputs tailored to your actual code
* Optional corpus collection, mutation and coverage guidance to find deeper bugs
* Powered by [Slither](https://github.com/crytic/slither) to extract useful information before the fuzzing campaign
* Source code integration to identify which lines are covered after the fuzzing campaign
* Curses-based retro UI, text-only or JSON output
* Automatic testcase minimization for quick triage
* Seamless integration into the development workflow
* Maximum gas usage reporting of the fuzzing campaign
* Support for a complex contract initialization with [Etheno](https://github.com/crytic/etheno) and Truffle

.. and [a beautiful high-resolution handcrafted logo](https://raw.githubusercontent.com/crytic/echidna/master/echidna.png).

<a href="https://trailofbits.files.wordpress.com/2020/03/image5.png"><img src="https://trailofbits.files.wordpress.com/2020/03/image5.png" width="650"/></a>

## Usage

### Executing the test runner

The core Echidna functionality is an executable called `echidna-test`. `echidna-test` takes a contract and a list of invariants (properties that should always remain true) as input. For each invariant, it generates random sequences of calls to the contract and checks if the invariant holds. If it can find some way to falsify the invariant, it prints the call sequence that does so. If it can't, you have some assurance the contract is safe.

### Writing invariants

Invariants are expressed as Solidity functions with names that begin with `echidna_`, have no arguments, and return a boolean. For example, if you have some `balance` variable that should never go below `20`, you can write an extra function in your contract like this one:

```solidity
function echidna_check_balance() public returns (bool) {
    return(balance >= 20);
}
```

To check these invariants, run:

```
$ echidna-test myContract.sol
```

An example contract with tests can be found [examples/solidity/basic/flags.sol](examples/solidity/basic/flags.sol). To run it, you should execute:
```
$ echidna-test examples/solidity/basic/flags.sol
```

Echidna should find a a call sequence that falsifies `echidna_sometimesfalse` and should be unable to find a falsifying input for `echidna_alwaystrue`.

### Collecting and visualizing coverage

After finishing a campaign, Echidna can save a coverage maximizing **corpus** in a special directory specified with the `corpusDir` config option. This directory will contain two entries: (1) a directory named `coverage` with JSON files that can be replayed by Echidna and (2) a plain-text file named `covered.txt`, a copy of the source code with coverage annotations.

If you run `examples/solidity/basic/flags.sol` example, Echidna will save a few files serialized transactions in the `coverage` directory and a `covered.$(date +%s).txt` file with the following lines:

```
*r  |  function set0(int val) public returns (bool){
*   |    if (val % 100 == 0)
*   |      flag0 = false;
  }

*r  |  function set1(int val) public returns (bool){
*   |    if (val % 10 == 0 && !flag0)
*   |      flag1 = false;
  }
```

Our tool signals each execution trace in the corpus with the following "line marker":
 - `*` if an execution ended with a STOP
 - `r` if an execution ended with a REVERT
 - `o` if an execution ended with an out-of-gas error
 - `e` if an execution ended with any other error (zero division, assertion failure, etc)

### Support for smart contract build systems

Echidna can test contracts compiled with different smart contract build systems, including [Truffle](https://truffleframework.com/), [Embark](https://framework.embarklabs.io/) and even [Vyper](https://vyper.readthedocs.io), using [crytic-compile](https://github.com/crytic/crytic-compile). For instance,
we can uncover an integer overflow in the [Metacoin Truffle box](https://github.com/truffle-box/metacoin-box) using a
[contract with Echidna properties to test](examples/solidity/truffle/metacoin/contracts/MetaCoinEchidna.sol):

```
$ cd examples/solidity/truffle/metacoin
$ echidna-test . --contract TEST
...
echidna_convert: failed!💥
  Call sequence:
    mint(57896044618658097711785492504343953926634992332820282019728792003956564819968)
```

Echidna supports two modes of testing complex contracts. Firstly, one can [describe an initialization procedure with Truffle and Etheno](https://github.com/crytic/building-secure-contracts/blob/master/program-analysis/echidna/end-to-end-testing.md) and use that as the base state for Echidna. Secondly, echidna can call into any contract with a known ABI by passing in the corresponding solidity source in the CLI. Use `multi-abi: true` in your config to turn this on.

### Configuration options

Echidna's CLI can be used to choose the contract to test and load a
configuration file.

```
$ echidna-test contract.sol --contract TEST --config config.yaml
```

The configuration file allows users to choose EVM and test generation
parameters. An example of a complete and annotated config file with the default
options can be found at
[examples/solidity/basic/default.yaml](examples/solidity/basic/default.yaml).
More detailed documentation on the configuration options is available in our
[wiki](https://github.com/trailofbits/echidna/wiki/Config).

Echidna supports three different output drivers. There is the default `text`
driver, a `json` driver, and a `none` driver, which should suppress all
`stdout` output. The JSON driver reports the overall campaign as follows.

```json
Campaign = {
  "success"      : bool,
  "error"        : string?,
  "tests"        : [Test],
  "seed"         : number,
  "coverage"     : Coverage,
  "gas_info"     : [GasInfo]
}
Test = {
  "contract"     : string,
  "name"         : string,
  "status"       : string,
  "error"        : string?,
  "testType"     : string,
  "transactions" : [Transaction]?
}
Transaction = {
  "contract"     : string,
  "function"     : string,
  "arguments"    : [string]?,
  "gas"          : number,
  "gasprice"     : number
}
```

`Coverage` is a dict describing certain coverage increasing calls.
Each `GasInfo` entry is a tuple that describes how maximal
gas usage was achieved, and also not too important. These interfaces are
subject to change to be slightly more user friendly at a later date. `testType`
will either be `property` or `assertion`, and `status` always takes on either
`fuzzing`, `shrinking`, `solved`, `passed`, or `error`.

### Using Echidna in a GitHub Actions workflow

There is an Echidna action which can be used to run `echidna-test` as part of a
GitHub Actions workflow. Please refer to the
[crytic/echidna-action](https://github.com/crytic/echidna-action) repository for
usage instructions and examples.

### Debugging Performance Problems

The best way to deal with an Echidna performance issue is to run `echidna-test` with profiling on.
This creates a text file, `echidna-test.prof`, which shows which functions take up the most CPU and memory usage.

To build a version of `echidna-test` that supports profiling, either Stack or Nix should be used.
With Stack, adding the flag `--profile` will make the build support profiling.
With Nix, running `nix-build --arg profiling true` will make the build support profiling.

To run with profiling on, add the flags `+RTS -p` to your original `echidna-test` command.

Performance issues in the past have been because of functions getting called repeatedly when they could be memoized,
and memory leaks related to Haskell's lazy evaluation;
checking for these would be a good place to start.

### Crash course on Echidna

Our [Building Secure Smart Contracts](https://github.com/crytic/building-secure-contracts/tree/master/program-analysis/echidna#echidna-tutorial) repository contains a crash course on Echidna, including examples, lessons and exercises.

## Limitations and known issues

EVM emulation and testing is hard. Echidna has a number of limitations in the latest release. Some of these are inherited from [hevm](https://github.com/dapphub/dapptools/tree/master/src/hevm) while some are results from design/performance decisions or simply bugs in our code. We list them here including their corresponding issue and the status ("wont fix", "in review", "fixed"). Issues that are "fixed" are expected to be included in the next Echidna release.

| Description |  Issue   | Status   |
| :--- |     :---:              |         :---:   |
| Debug information can be insufficient | [#656](https://github.com/crytic/echidna/issues/656) | *[in review for 2.0](https://github.com/crytic/echidna/pull/674)* |
| Vyper support is limited | [#652](https://github.com/crytic/echidna/issues/652) | *wont fix* |
| Limited library support for testing | [#651](https://github.com/crytic/echidna/issues/651) | *wont fix* |
| If the contract is not properly linked, Echidna will crash | [#514](https://github.com/crytic/echidna/issues/514) | *in review* |
| Assertions are not detected in internal transactions | [#601](https://github.com/crytic/echidna/issues/601) | *[in review for 2.0](https://github.com/crytic/echidna/pull/674)* |
| Assertions are not detected in solc 0.8.x | [#669](https://github.com/crytic/echidna/issues/669) | *[in review for 2.0](https://github.com/crytic/echidna/pull/674)* |
| Value generation can fail in multi-abi mode, since the function hash is not precise enough | [#579](https://github.com/crytic/echidna/issues/579) | *[in review for 2.0](https://github.com/crytic/echidna/pull/674)*|

## Installation

### Precompiled binaries

Before starting, make sure Slither is [installed](https://github.com/crytic/slither) (`pip3 install slither-analyzer --user`).
If you want to quickly test Echidna in Linux or MacOS, we provide statically linked Linux binaries built on Ubuntu and mostly static MacOS binaries on our [releases page](https://github.com/crytic/echidna/releases). You can also grab the same type of binaries from our [CI pipeline](https://github.com/crytic/echidna/actions?query=workflow%3ACI+branch%3Amaster+event%3Apush), just click the commit to find binaries for Linux or MacOS.

### Docker container

If you prefer to use a pre-built Docker container, log into Github on your local `docker` client and check out our [docker package](https://github.com/crytic/echidna/packages/136575), which are also auto-built via Github Actions.
Otherwise, if you want to install the latest released version of Echidna, we recommend using docker:

```
$ docker build -t echidna .
```

Then, run it via:

```
$ docker run -it -v `pwd`:/src echidna echidna-test /src/examples/solidity/basic/flags.sol
```

### Building using Stack

If you'd prefer to build from source, use [Stack](https://docs.haskellstack.org/en/stable/README/). `stack install` should build and compile `echidna-test` in `~/.local/bin`. You will need to link against libreadline and libsecp256k1 (built with recovery enabled), which should be installed with the package manager of your choosing. You also need to install the latest release of [libff](https://github.com/scipr-lab/libff). Refer to our [CI tests](.github/scripts/install-libff.sh) for guidance.

Some Linux distributions do not ship static libraries for certain things that Haskell needs, e.g. Arch Linux, which will cause `stack build` to fail with linking errors because we use the `-static` flag. Removing these from `package.yaml` should get everything to build if you are not looking for a static build.

If you're getting errors building related to linking, try tinkering with `--extra-include-dirs` and `--extra-lib-dirs`.

### Building using Nix (works natively on Apple M1 systems)

[Nix users](https://nixos.org/download.html) can install the lastest Echidna with:
```
$ nix-env -i -f https://github.com/crytic/echidna/tarball/master
```

To build a standalone release for non-Nix macOS systems, the following will
bundle Echidna and all linked dylibs in a tarball:

```
$ nix-build macos-release.nix
$ ll result/
bin    echidna-1.7.3-aarch64-darwin.tar.gz
```

It is possible to develop Echidna with Cabal inside `nix-shell`. Nix will automatically
install all the dependencies required for development including `crytic-compile` and `solc`.
A quick way to get GHCi with Echidna ready for work:
```
$ git clone https://github.com/crytic/echidna
$ cd echidna
$ nix-shell
[nix-shell]$ cabal new-repl
```

Running the test suite:
```
nix-shell --run 'cabal test'
```

## Public use of Echidna

### Property testing suites

This is a partial list of smart contracts projects that use Echidna for testing:

* [Uniswap-v3](https://github.com/search?q=org%3AUniswap+echidna&type=commits)
* [Balancer](https://github.com/balancer-labs/balancer-core/tree/master/echidna)
* [MakerDAO vest](https://github.com/makerdao/dss-vest/pull/16)
* [Optimism DAI Bridge](https://github.com/BellwoodStudios/optimism-dai-bridge/blob/master/contracts/test/DaiEchidnaTest.sol)
* [WETH10](https://github.com/WETH10/WETH10/tree/main/contracts/fuzzing)
* [Yield](https://github.com/yieldprotocol/fyDai/pull/312)
* [Convexity Protocol](https://github.com/opynfinance/ConvexityProtocol/tree/dev/contracts/echidna)
* [Aragon Staking](https://github.com/aragon/staking/blob/82bf54a3e11ec4e50d470d66048a2dd3154f940b/packages/protocol/contracts/test/lib/EchidnaStaking.sol)
* [Centre Token](https://github.com/centrehq/centre-tokens/tree/master/echidna_tests)
* [Tokencard](https://github.com/tokencard/contracts/tree/master/tools/echidna)
* [Minimalist USD Stablecoin](https://github.com/usmfum/USM/pull/41)

### Trophies

The following security vulnerabilities were found by Echidna. If you found a security vulnerability using our tool, please submit a PR with the relevant information.

| Project | Vulnerability | Date |
|--|--|--|
[0x Protocol](https://github.com/trailofbits/publications/blob/master/reviews/0x-protocol.pdf) | If an order cannot be filled, then it cannot be canceled | Oct 2019
[0x Protocol](https://github.com/trailofbits/publications/blob/master/reviews/0x-protocol.pdf) | If an order can be partially filled with zero, then it can be partially filled with one token | Oct 2019
[0x Protocol](https://github.com/trailofbits/publications/blob/master/reviews/0x-protocol.pdf) | The cobbdouglas function does not revert when valid input parameters are used | Oct 2019
[Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) | An attacker cannot steal assets from a public pool | Jan 2020
[Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) | An attacker cannot generate free pool tokens with joinPool | Jan 2020
[Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) | Calling joinPool-exitPool does not lead to free pool tokens | Jan 2020
[Balancer Core](https://github.com/trailofbits/publications/blob/master/reviews/BalancerCore.pdf) |  Calling exitswapExternAmountOut does not lead to free assets | Jan 2020
[Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | [Closing troves require to hold the full amount of LUSD minted](https://github.com/liquity/dev/blob/echidna_ToB_final/packages/contracts/contracts/TestContracts/E2E.sol#L242-L298) | Dec 2020
[Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | [Troves can be improperly removed](https://github.com/liquity/dev/blob/echidna_ToB_final/packages/contracts/contracts/TestContracts/E2E.sol#L242-L298) | Dec 2020
[Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | Initial redeem can revert unexpectedly | Dec 2020
[Liquity Dollar](https://github.com/trailofbits/publications/blob/master/reviews/Liquity.pdf) | Redeem without redemptions might still return success | Dec 2020
[Origin Dollar](https://github.com/trailofbits/publications/blob/master/reviews/OriginDollar.pdf) | Users are allowed to transfer more tokens that they have | Nov 2020
[Origin Dollar](https://github.com/trailofbits/publications/blob/master/reviews/OriginDollar.pdf) | User balances can be larger than total supply | Nov 2020
[Yield Protocol](https://github.com/trailofbits/publications/blob/master/reviews/YieldProtocol.pdf) | Arithmetic computation for buying and selling tokens is imprecise | Aug 2020

### Research

We can also use Echidna to reproduce research examples from smart contract fuzzing papers to show how quickly it can find the solution. All these can be solved, from a few seconds to one or two minutes on a laptop computer.

| Source | Code
|--|--
[Using automatic analysis tools with MakerDAO contracts](https://forum.openzeppelin.com/t/using-automatic-analysis-tools-with-makerdao-contracts/1021) | [SimpleDSChief](https://github.com/crytic/echidna/blob/master/examples/solidity/research/vera_dschief.sol)
[Integer precision bug in Sigma Prime](https://github.com/b-mueller/sabre#example-2-integer-precision-bug) | [VerifyFunWithNumbers](https://github.com/crytic/echidna/blob/master/examples/solidity/research/solcfuzz_funwithnumbers.sol)
[Learning to Fuzz from Symbolic Execution with Application to Smart Contracts](https://files.sri.inf.ethz.ch/website/papers/ccs19-ilf.pdf) | [Crowdsale](https://github.com/crytic/echidna/blob/master/examples/solidity/research/ilf_crowdsale.sol)
[Harvey: A Greybox Fuzzer for Smart Contracts](https://arxiv.org/abs/1905.06944) | [Foo](https://github.com/crytic/echidna/blob/master/examples/solidity/research/harvey_foo.sol), [Baz](https://github.com/crytic/echidna/blob/master/examples/solidity/research/harvey_baz.sol)

### Academic Publications

| Paper Title | Venue | Publication Date |
| --- | --- | --- |
| [echidna-parade: Diverse multicore smart contract fuzzing](https://agroce.github.io/issta21.pdf) | [ISSTA 2021](https://conf.researchr.org/home/issta-2021) | July 2021 |
| [Echidna: Effective, usable, and fast fuzzing for smart contracts](papers/echidna_issta2020.pdf) | [ISSTA 2020](https://conf.researchr.org/home/issta-2020) | July 2020 |
| [Echidna: A Practical Smart Contract Fuzzer](papers/echidna_fc_poster.pdf) | [FC 2020](https://fc20.ifca.ai/program.html) | Feb 2020 |

If you are using Echidna for academic work, consider applying to the [Crytic $10k Research Prize](https://blog.trailofbits.com/2019/11/13/announcing-the-crytic-10k-research-prize/).

## Getting help

Feel free to stop by our #ethereum slack channel in [Empire Hacking](https://empireslacking.herokuapp.com/) for help using or extending Echidna.

* Get started by reviewing these simple [Echidna invariants](examples/solidity/basic/flags.sol)

* Review the [Solidity examples](examples/solidity) directory for more extensive Echidna use cases

* Considering [emailing](mailto:echidna-dev@trailofbits.com) the Echidna development team directly for more detailed questions

## License

Echidna is licensed and distributed under the [AGPLv3 license](https://github.com/crytic/echidna/blob/master/LICENSE).
