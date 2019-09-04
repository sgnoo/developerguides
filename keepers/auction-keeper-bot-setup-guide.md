# Auction Keeper Bot Setup Guide 

### Guide Agenda

This guide will show how to use the auction-keeper to interact with the Kovan deployment of the MCD smart contracts. More specifically, the guide will showcase how to go through the following stages of setting up and running an Auction Keeper bot:

1. Introduction
2. Bidding Models
    - Starting and stopping bidding models
    - Communicating with bidding models
3. Setting up the Keeper Bot (Flip Auction Keeper) 
    - Prerequisites 
    -   Installation
4. Running your Keeper Bot (Usage)
    - Keeper Limitations
5. Accounting
6. Testing
7. Support

We are proud to say that since the Maker Protocol is an open-source platform, all of the code we have created to run the Keeper bot is free and accessible to all.

# 1. Introduction

Auction Keepers participate in auctions as a result of liquidation events and thereby acquire collateral at attractive prices. An `auction-keeper` can participate in three different types of auctions:

1. [Collateral Auction (`flip`)](https://github.com/makerdao/dss/blob/master/src/flip.sol)
2. [Surplus Auction (`flap`)](https://github.com/makerdao/dss/blob/master/src/flap.sol)
3. [Debt Auction (`flop`)](https://github.com/makerdao/dss/blob/master/src/flop.sol)

Auction Keepers have the unique ability to plug in external *bidding models*, which communicate information to the Keeper on when and how high to bid (these types of Keepers can be left safely running in the background). Shortly after an Auction Keeper notices or starts a new auction, it will spawn a new instance of a *bidding model* and act according to its specified instructions. Bidding models will be automatically terminated by the Auction Keeper the moment the auction expires. 

**Note:** 

Auction Keepers will automatically call `deal` (claiming a winning bid / settling a completed auction) if the Keeper's address won the auction. 

## Auction Keeper Architecture

As mentioned above, Auction Keepers directly interact with `Flipper`, `Flapper` and `Flopper` auction contracts deployed to the Ethereum mainnet. All decisions which involve pricing details are delegated to the *bidding models*. The Bidding models are simply executable strategies, external to the main `auction-keeper` process. This means that the bidding models themselves do not have to know anything about the Ethereum blockchain and its smart contracts, as they can be implemented in basically any programming language. However, they do need to have the ability to read and write JSON documents, as this is how they communicate/exchange with `auction-keeper`. It's important to note that as a developer running an Auction Keeper, it is required that you have basic knowledge on how to properly start and configure the auction-keeper. For example, providing startup parameters as keystore / password are required to setup and run a Keeper. Additionally, you should be familiar with the MCD system, as the model will receive auction details from auction-keeper in the form of a JSON message containing keys such as lot, beg, guy, etc. 

**Simple Bidding Model Example:** 

A simple bidding model could be a shell script which echoes a fixed price (further details below).

## The Purpose of Auction Keepers

**The main purpose of Auction Keepers are:**

- To discover new opportunities and start new auctions.
- To constantly monitor all ongoing auctions.
- To detect auctions started by other participants.
- To Bid on auctions by converting token prices into bids.
- To ensure that instances of *bidding model* are running for each auction type as well as making sure the instances match the current status of their auctions. This ensure that Keepers are bidding according to decisions outlined by the bidding model.

The auction discovery and monitoring mechanisms work by operating as a loop, which initiates on every new block and enumerates all auctions from `1` to `kicks`. When this occurs, even when the *bidding model* decides to send a bid, it will not be processed by the Keeper until the next iteration of that loop. It's important to note that the `auction-keeper` not only monitors existing auctions and discovers new ones, but it also identifies and takes opportunities to create new auctions.

# 2. Bidding Models

## Starting and Stopping Bidding Models

Auction Keeper maintains a collection of child processes, as each *bidding model* is its own dedicated process. New processes (new *bidding model* instances) are spawned by executing a command according to the `--model` command-line parameter. These processes are automatically terminated (via `SIGKILL`) by the keeper shortly after their associated auction expires. Whenever the *bidding model* process dies, it gets automatically re-spawned by the Keeper.

**Example:**

    bin/auction-keeper --model '../my-bidding-model.sh' [...]

## **Communicating with *bidding models***

Auction Keepers communicate with *bidding models* via their standard input/standard output. Once the process has started and every time the auction state changes, the Keeper sends a one-line JSON document to the **standard input** of the *bidding model.* 

**A sample JSON message sent from the keeper to the model looks like the:**
```
{"id": "6", "flapper": " 0xf0afc3108bb8f196cf8d076c8c4877a4c53d4e7c ", "bid": "7.142857142857142857", "lot": "10000.000000000000000000", "beg": "1.050000000000000000", "guy": " 0x00531a10c4fbd906313768d277585292aa7c923a ", "era": 1530530620, "tic": 1530541420, "end": 1531135256, "price": "1400.000000000000000028"}
```
## Glossary (Bidding Models):

- `id` - auction identifier.
- `flipper` - Ethereum address of the `Flipper` contract (only for `flip` auctions).
- `flapper` - Ethereum address of the `Flapper` contract (only for `flap` auctions).
- `flopper` - Ethereum address of the `Flopper` contract (only for `flop` auctions).
- `bid` - current highest bid (will go up for `flip` and `flap` auctions).
- `lot` - amount being currently auctioned (will go down for `flip` and `flop` auctions).
- `tab` - bid value (not to be confused with the bid price) which will cause the auction to enter the `dent` phase (only for `flip` auctions).
- `beg` - minimum price increment (`1.05` means minimum 5% price increment).
- `guy` - Ethereum address of the current highest bidder.
- `era` - current time (in seconds since the UNIX epoch).
- `tic` - time when the current bid will expire (`None` if no bids yet).
- `end` - time when the entire auction will expire (end is set to `0` is the auction is no longer live).
- `price` - current price being tendered (can be `None` if price is infinity).

---

*Bidding models* should never make an assumption that messages will be sent only when auction state changes. It is perfectly fine for the `auction-keeper` to periodically send the same message(s) to *bidding models*. 

At the same time, the `auction-keeper` reads one-line messages from the **standard output** of the *bidding model* process and tries to parse them as JSON documents. It will then extract the two following fields from that document:

- `price` - the maximum (for `flip` and `flop` auctions) or the minimum (for `flap` auctions) price the model is willing to bid.
- `gasPrice` (optional) - gas price in Wei to use when sending a bid.

**An example of a message sent from the Bidding Model to the Auction Keeper may look like:**
```
    {"price": "150.0", "gasPrice": 7000000000}
```
In the case of when Auction Keepers and Bidding Models communicate in terms of prices, it is the MKR/DAI price (for `flap` and `flop` auctions) or the collateral price expressed in DAI for `flip` auctions (for example, OMG/DAI).

Any messages written by a Bidding Model to **stderr** (standard error) will be passed through by the Auction Keeper to its logs. This is the most convenient way of implementing logging from Bidding Models.

# 3. Setting up the Auction Keeper Bot (Installation)

### Prerequisite

- Git
- [Python v3.6.6](https://www.python.org/downloads/release/python-366/)
- [virtualenv](https://virtualenv.pypa.io/en/latest/)
    - This project requires *virtualenv* to be installed if you want to use Maker's python tools. This helps to ensure that you are running the right version of python as well as check that all of the pip packages that are installed in the [install.sh](http://install.sh) are in the right place and have the correct versions.
- [X-code](https://apps.apple.com/ca/app/xcode/id497799835?mt=12) (for Macs)
- [Docker-Compose](https://docs.docker.com/compose/install/)

## Getting Started

### Installation from source:

1. **Clone the `auction-keeper` repository:** 
```
git clone https://github.com/makerdao/auction-keeper.git
```
2. **Switch into the `auction-keeper` directory:** 
```
cd auction-keeper
```
3. **Install required third-party packages:**
```
git submodule update --init --recursive
```
4. **Set up the virtual env and activate it:**
```
python3 -m venv _virtualenv
source _virtualenv/bin/activate
```
**5. Install requirements:**
```
pip3 install -r requirements.txt
```
### Potential Errors:
- Needing to upgrade pip version to 19.2.2:
    - Fix by running `pip install --upgrade pip`.

For other known Ubuntu and macOS issues please visit the [pymaker](https://github.com/makerdao/pymaker) README.

# 4. Running your Keeper Bot

### 1. Creating your bidding model (an example detailing the simplest possible bidding model)

The stdout (standard output) provides a price for the collateral (for `flip` auctions) or MKR (for `flap` and `flop` auctions). The `sleep` locks the price in place for a minute, after which the keeper will restart the price model and read a new price (consider this your price update interval).

The simplest possible *bidding model* you can set up is when you use a fixed price for each auction. For example: 
```
    #!/usr/bin/env bash
    echo "{\"price\": \"150.0\"}" # put your desired fixed price amount here 
    sleep 60 # locking the price for a 60 seconds period
```
Once you have created your bidding model, save it as `model-eth.sh` (or whatever name you feel seems appropriate).

### 2. Setting up an Auction Keeper for a Collateral (Flip) Auction

Collateral Auctions will be the most common type of auction that the community will want to create and operate Auction keepers for. This is due to the fact that Collateral auctions will occur much more frequently than Flap and Flop auctions. 

### **Example (Flip Auction Keeper):**

- This example/process assumes that the user has an already existing shell script that manages their environment and connects to the Ethereum blockchain.
```
#!/bin/bash
dir="$(dirname "$0")"

source my_environment.sh  # Set the RPC host, account address, and keys.
source _virtualenv/bin/activate # Run virtual environment

# Allows keepers to bid different prices
MODEL=$1

bin/auction-keeper \
    --rpc-host ${SERVER_ETH_RPC_HOST:?} \
    --rpc-port ${SERVER_ETH_RPC_PORT?:} \
    --rpc-timeout 30 \
    --eth-from ${ACCOUNT_ADDRESS?:} \
    --eth-key ${ACCOUNT_KEY?:} \
    --type flip \ # type of auction the keeper will be working with
    --ilk ETH-A \
    --network kovan \
    --vat-dai-target 1000 \
    --model ${dir}/${MODEL} \
    2> >(tee -a ${LOGS_DIR?:}/auction-keeper-flip-ETH-A.log >&2)
```
Once finalized, you should save your script to run your Auction Keeper as `flip-eth-a.sh ` (or something similar to identify that this Auction Keeper is for a Flip Auction). 

**Notes:**

- All Collateral types (`ilk`'s) combine the name of the token and a letter corresponding to a set of risk parameters. For example, as you can see above, the example uses ETH-A. Note that ETH-A and ETH-B are two different collateral types for the same underlying token (WETH) but have different risk parameters.
- For the MCD addresses, we simply pass `--network mainnet|kovan` in and it will load the required JSON files bundled within auction-keeper (or pymaker).
    - If you pass in `mainnet` before MCD is released, it will fail with an appropriate error message.

### 3. Passing the bidding the model as an argument to the Keeper script

1. Confirm that both your bidding model (model-eth.sh) and your script (flip-eth-a.sh) to run your Auction Keeper are saved.
2. The next step is to `chmod +x` both of them.
3. Lastly, run `flip-eth-a.sh model-eth.sh` to pass your bidding model into your Auction Keeper script. 

### Auction Keeper Arguments Explained

To participate in all auctions, a separate keeper must be configured for `flip` of each collateral type, as well as one for `flap` and another for `flop`.

1. `--type` - the type of auction the keeper is used for. In this particular scenario, it will be set to `flip`. 
2. `--ilk` - the type of collateral.  
3. `--addresses` - .json of all of the addresses of the MCD contracts as well as the collateral types allowed/used in the system.
4. `--vat-dai-target` - the amount of DAI which the keeper will attempt to maintain in the Vat, to use for bidding. It will rebalance it upon keeper startup and upon `deal`ing an auction.
5. `--model` - the bidding model that will be used for bidding.  

Call `bin/auction-keeper --help` for a complete list of arguments.

## Auction Keeper Limitations

- If an auction starts before the auction Keeper has started, the Keeper will not participate in the auction until the next block has been mined.
- Keepers do not explicitly handle global settlement (`End`). If global settlement occurs while a winning bid is outstanding, the Keeper will not request a `yank` to refund the bid. The workaround is to call `yank` directly using `seth`.
- There are some Keeper functions that incur gas fees regardless of whether a bid is submitted. This includes, but is not limited to, the following actions:
    - Submitting approvals.
    - Adjusting the balance of surplus to debt.
    - Biting a CDP or starting a flap or flop auction, even if insufficient funds exist to participate in the auction.
- The Keeper will not check model prices until an auction officially exists. As such, it will `kick`, `flap`, or `flop` in response to opportunities regardless of whether or not your DAI or MKR balance is sufficient to participate. This imposes a gas fee that must be paid.
    - After procuring more DAI, the Keeper must be restarted to add it to the `Vat`.

# 5. Accounting

The Auction contracts exclusively interact with DAI (for all auctions types) and collateral (for `flip` auctions) in the `Vat`. More explicitly speaking:

- The DAI that is used to bid on auctions is withdrawn from the `Vat`.
- The Collateral and surplus DAI won at auction end is placed in the `Vat`.

By default, all the DAI and collateral within your `eth-from` account is `exit`'ed from the Vat and added to your account token balance when the Keeper is shut down. Note that this feature may be disabled using the `keep-dai-in-vat-on-exit` and `keep-gem-in-vat-on-exit` switches, respectively. The use of an `eth-from` account with an open CDP is discouraged, as debt will hinder the auction contracts' ability to access your DAI, and the `auction-keeper`'s ability to `exit` DAI from the `Vat`.

When running multiple Auction Keepers using the same account, the balance of DAI in the `Vat` will be shared across all of the Keepers. If using this feature, you should set `--vat-dai-target` to the same value for each Keeper, as well as sufficiently high in order to cover total desired exposure.

**Note:**

MKR used to bid on `flap` auctions is directly withdrawn from your token balance. The MKR won at `flop` auctions is directly deposited to your token balance.

# 6. Testing your Keeper
**Required for testing:** 

  - docker-compose
  - This project uses [pytest](https://docs.pytest.org/en/latest/) for unit testing. Testing depends upon on a Dockerized local testchain that is included in `lib\pymaker\tests\config`.


1. In order to be able to run tests, please install development dependencies first by executing:
```
    pip3 install -r requirements-dev.txt
```
2. You can then run all the unit tests by running the following script:
```
    ./test.sh
```
**Example Output**
```
    ============================= test session starts ==============================
    platform darwin -- Python 3.6.6, pytest-3.3.0, py-1.8.0, pluggy-0.6.0
    rootdir: /Users/CP/auction-keeper, inifile:
    plugins: timeout-1.2.1, mock-1.6.3, cov-2.5.1, asyncio-0.8.0
    collected 94 items                                                             
    
    tests/test_accounting.py .........                                       [  9%]
    tests/test_bite.py .                                                     [ 10%]
    tests/test_config.py ......                                              [ 17%]
    tests/test_flap.py ....................                                  [ 38%]
    tests/test_flip.py .........................                             [ 64%]
    tests/test_flop.py ....................                                  [ 86%]
    tests/test_process.py .............                                      [100%]
    
    ---------- coverage: platform darwin, python 3.6.6-final-0 -----------
    
    Name                         Stmts   Miss  Cover
    ------------------------------------------------
    auction_keeper/__init__.py       0      0   100%
    auction_keeper/gas.py           11      0   100%
    auction_keeper/logic.py         52      0   100%
    auction_keeper/main.py         328      0   100%
    auction_keeper/model.py        135      0   100%
    auction_keeper/process.py       87      0   100%
    auction_keeper/strategy.py      94      0   100%
    ------------------------------------------------
    TOTAL                          707      0   100%
```
## Manual Testing

Within `tests/manual` resides a collection of python and shell scripts that may be used to test auction-keeper, pymaker's auction facilities, and relevant smart contracts in [dss.](https://github.com/makerdao/dss/tree/master/src)

### Testing Scenarios

- `create_unsafe_cdp.py`
    - Creates a single CDP close to the liquidation ratio, and then slightly drops the price of ETH such that the CDP becomes undercollateralized. If this test is run on an already-undercollateralized CDP, it will submit no transaction. In either case, it prints a message stating whether an action was performed. This may be used to test `flip` auctions as well as build debt for `flop` auctions.
- `create_surplus.py`
    - Creates a CDP and then calls `jug.drip` to establish a surplus. The age of the test chain and the amount of collateral placed in this CDP determines how much surplus is generated.
- `print.py`

    Provides status information based on the parameter passed:

    - `--auctions` prints the most recent bid for each active auction.
    - `--balances` shows the balance of surplus and debt.
- `purchase_dai.py`
    - Creates a CDP using another address and transfers DAI to the keeper account for bidding on `flip` and `flop` auctions. The amount of DAI to purchase is passed as the only argument.

- `mint_mkr.py`
    - Generates MKR tokens for bidding on `flap` auctions. The amount of MKR to mint is passed as the only argument.

For more information on how to perform manual testing on your keeper manually, click [here.](https://github.com/makerdao/auction-keeper/tree/master/tests/manual#auction-keeper-manual-testing)

# 7. Support

We welcome any questions or concerns about the Auction Keepers in the [#keeper](https://chat.makerdao.com/channel/keeper) channel in the Maker Chat.

## Disclaimer

You (meaning any individual or entity accessing, using or both the software included in this Github repository) expressly understand and agree that your use of the software is at your sole risk. The software in this Github repository is provided “as is”, without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose and noninfringement. In no event shall the authors or copyright holders be liable for any claim, damages or other liability, whether in an action of contract, tort or otherwise, arising from, out of or in connection with the software or the use or other dealings in the software. You release authors or copyright holders from all liability for you having acquired or not acquired content in this GitHub repository. The authors or copyright holders make no representations concerning any content contained in or accessed through the service, and the authors or copyright holders will not be responsible or liable for the accuracy, copyright compliance, legality or decency of material contained in or accessed through this GitHub repository.