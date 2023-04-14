# Overview

Original links are as follows: 

|#|Issue|Original link|
|-|:-|:-:|
| [QA] | Quality Assurance (grade-a) | [Link](https://github.com/code-423n4/2023-01-opensea-findings/issues/21) |


Typos may have been fixed and, the discussion part, was added at the end where applicable.

# Quality Assurance (grade-a)

# [1] incorrect size round up in _encodeBytes

Context: https://github.com/ProjectOpenSea/seaport/blob/5de7302bc773d9821ba4759e47fc981680911ea0/contracts/lib/ConsiderationEncoder.sol#L73

`_encodeBytes` attempts to round up the provided size of src memory pointer to the nearest word but instead it rounds up to the second nearest word by using `AlmostTwoWords` instead of `AlmostOneWord`

```
    // Mask the length of the bytes array to protect against overflow
    // and round up to the nearest word.
    size = (src.readUint256() + AlmostTwoWords) & OnlyFullWordMask;
```

`_encodeBytes` is used in 3 places:
- `_encodeGenerateOrder` ->  over extens the size of the `bytes memory context` in the calldata
- `_encodeRatifyOrder` -> over extens the size of the `bytes memory context` in the calldata
- `_encodeValidateOrder` -> over extens the size of `bytes memory extraData` in the calldata

In all cases, the over increase in size dose not pose an issue internally as the values are only passed through a `call` to the calling `ContractOffererInterface` implementation functions. Here, depending on the logic used by the implementer the size may cause an issue/hidden bug, althought of a low severity.

### Proof of Concept

```
pragma solidity ^0.8.13.0;

contract Testing {

    function _encodeBytes(uint256 src) external pure returns (uint256 size) {
        uint256 AlmostTwoWords = 0x3f;
        uint256 OnlyFullWordMask = 0xff_ff_ff_e0;

        unchecked {
            size = (src + AlmostTwoWords) & OnlyFullWordMask;
        }
    }
}
```

Input: *0x12345678*

Expectation: *0x12345680*

Output: *0x123456A0*

### Recommended Mitigation Steps
change `AlmostTwoWords` to `AlmostOneWord`

# [2] Natspec minor issues

Natspec typos of wrongly named arguments

For function `_encodeBytes` second param is named `dst` but the natspec has it as `src`. Simply rename it to fix.

Context: https://github.com/ProjectOpenSea/seaport/blob/5de7302bc773d9821ba4759e47fc981680911ea0/contracts/lib/ConsiderationEncoder.sol#L60


# Discussions

HickupHH3: 

Nice find for the 1st issue! Possibly grade A because of it.

Regarding the 1st issue, impl is working as expected, whereby word storing the length needs to be copied over as well. See https://github.com/ProjectOpenSea/seaport/pull/906/files

Keeping the grade because the issue helped to clarify the return argument of the function => finding is of value.