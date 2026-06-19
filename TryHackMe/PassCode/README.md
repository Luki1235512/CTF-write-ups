# [PassCode](https://tryhackme.com/room/hfb1passcode)

## From the Hackfinity Battle CTF event.

We may have found a way to break into the DarkInject blockchain, exploiting a vulnerability in their system. This might be our only chance to stop them—for good.

### What is the flag?

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Challenge {
    string private secret = "THM{}";
    bool private unlock_flag = false;
    uint256 private code;
    string private hint_text;

    constructor(string memory flag, string memory challenge_hint, uint256 challenge_code) {
        secret = flag;
        code = challenge_code;
        hint_text = challenge_hint;
    }

    function hint() external view returns (string memory) {
        return hint_text;
    }

    function unlock(uint256 input) external returns (bool) {
        if (input == code) {
            unlock_flag = true;
            return true;
        }
        return false;
    }

    function isSolved() external view returns (bool) {
        return unlock_flag;
    }

    function getFlag() external view returns (string memory) {
        require(unlock_flag, "Challenge not solved yet");
        return secret;
    }
}
```

> The objective is to call `getFlag()`, which requires `unlock_flag` to be `true`. That boolean is only set by calling `unlock()` with the exact value of `code`. The `code` variable is declared `private`, but in Solidity **`private` only restricts other contracts from reading a variable. It does not hide the data on-chain**. Every storage slot of every contract is permanently readable by anyone with access to the node's JSON-RPC endpoint.

1. Read the `code` value directly from the contract's storage. Solidity lays out state variables into sequential 32-byte slots in the order they are declared:

```bash
cast storage <CONTRACT_ADDRESS> 2 --rpc-url http://<TARGET_IP>:8545
```

2. Send a transaction calling `unlock()` with the retrieved code. The `--legacy` flag is required because this network does not support EIP-1559 transactions and expects the older transaction format:

```bash
cast send <CONTRACT_ADDRESS> "unlock(uint256)" <CODE> --rpc-url http://<TARGET_IP>:8545 --private-key <PRIVATE_KEY> --legacy
```

[SCREEN01]
