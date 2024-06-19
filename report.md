# **HATs Arbitration Contracts Audit Competition on Hats.finance** 


## Introduction to Hats.finance


Hats.finance builds autonomous security infrastructure for integration with major DeFi protocols to secure users' assets. 
It aims to be the decentralized choice for Web3 security, offering proactive security mechanisms like decentralized audit competitions and bug bounties. 
The protocol facilitates audit competitions to quickly secure smart contracts by having auditors compete, thereby reducing auditing costs and accelerating submissions. 
This aligns with their mission of fostering a robust, secure, and scalable Web3 ecosystem through decentralized security solutions​.

## About Hats Audit Competition


Hats Audit Competitions offer a unique and decentralized approach to enhancing the security of web3 projects. Leveraging the large collective expertise of hundreds of skilled auditors, these competitions foster a proactive bug hunting environment to fortify projects before their launch. Unlike traditional security assessments, Hats Audit Competitions operate on a time-based and results-driven model, ensuring that only successful auditors are rewarded for their contributions. This pay-for-results ethos not only allocates budgets more efficiently by paying exclusively for identified vulnerabilities but also retains funds if no issues are discovered. With a streamlined evaluation process, Hats prioritizes quality over quantity by rewarding the first submitter of a vulnerability, thus eliminating duplicate efforts and attracting top talent in web3 auditing. The process embodies Hats Finance's commitment to reducing fees, maintaining project control, and promoting high-quality security assessments, setting a new standard for decentralized security in the web3 space​​.

## HATs Arbitration Contracts Overview

We turn black hat hackers into white hat hackers using the right incentives. Creating the future of security on _EVM

## Competition Details


- Type: A public audit competition hosted by HATs Arbitration Contracts
- Duration: 2 weeks
- Maximum Reward: $28,000
- Submissions: 74
- Total Payout: $24,900.4 distributed among 8 participants.

## Scope of Audit

The scope of the competition includes all the contracts that make up the HATs system, but we are specifically interested in the changes made since v2.0.

There are a number of changes since our last release:

- We added an arbitration procedure: if there is any kind of dispute about the payout, the decision can be  escalated to an Expert Committee, and if that committee cannot find consensus, to the Kleros court. The intended behavior of that system is documented here: https://github.com/hats-finance/hats-contracts/blob/audit/docs/claims.md
- The functionality of the main contract, HATVault.sol, was split into two new files (HATVault.sol and HATClaimsManager.sol) 
- We added some helper contracts, such as the PaymentSplitter.sol

The contracts that are currently in use, v2.0, can be found here https://github.com/hats-finance/hats-contracts/tree/v2.0.  


In this competition, we are interested in the usual programming errors, but we are also interested in any scenarios in which the arbitration procedure can be abused or hijacked. (For such attacks, it is the behavior that is implemented that serves as a reference - pointing out mistakes in documentation is appreciated, but will not be rewarded). 





## High severity issues


- **Vulnerability in refundExpiredSubmitClaimRequest() allows draining of all funds in HATArbitrator contract**

  Funds in the `HATArbitrator` contract are at risk of being permanently lost. An attacker can exploit the `refundExpiredSubmitClaimRequest()` function to repeatedly drain funds. This vulnerability arises due to several issues:

1. The function does not verify if the `_internalClaimId` corresponds to an existing `submitClaimRequest`.
2. An attacker can bypass the if-statement since a non-existing claim results in a zero timestamp.
3. The submitter address is not checked to be non-zero, allowing transfers to the zero address.
4. The transfer amount is hardcoded rather than reflecting the actual amount.
5. The function can be called by anyone, not just the original claimant.

Additionally, popular ERC-20 tokens, like USDT, DAI, and WETH, allow transfers to the zero address, compounding the issue. A proof of concept shows how an attacker can use a token like DAI, adjust the configuration, and repeatedly call the function to drain all funds from the `HATArbitrator`.

The recommended mitigations include verifying that claim requests are legitimate by checking if `submittedAt`, `bond`, and `submitter` are non-zero. The transfer amount should match `submitClaimRequest.bond`, and function access should be restricted to the actual submitter. Implementing such changes will safeguard the contract from these exploitative vulnerabilities.


  **Link**: [Issue #36](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/36)


- **Bug in `HATArbitrator.reclaimBond()` Allows Improper Refunds of Previously Dismissed Claims**

  `HATArbitrator.reclaimBond()` includes a vulnerability where it erroneously refunds previously dismissed claims. This flaw enables an attacker to create repetitive or malicious disputes and then reclaim the bonds through subsequent claims, adversely affecting the Expert Committee or future disputers.

**Attack Scenario:**
1. A claim (Claim A) is submitted.
2. User 0 disputes Claim A by calling `dispute()`, and their bond is stored in the `disputersBonds` mapping.
3. The Expert Committee dismisses the dispute over Claim A, transferring the bonds to themselves.
4. Another claim (Claim B) for the same vault is submitted.
5. User 1 disputes Claim B by calling `dispute()`, and their bond is also stored.
6. User 0 calls `reclaimBond()` to reclaim the bond used in disputing Claim A, incorrectly retrieving the bond amount due to prior storage, impacting the balance for Claim B.
7. When the Expert Committee dismisses the dispute over Claim B and attempts to transfer the remaining bonds, the operation fails due to insufficient funds caused by User 0’s reclamation.

**Recommendation:**
To address this issue, modify `reclaimBond()` to subtract reclaimed values from `totalBondsOnClaim[_vault][_claimId]` and reset this value after bonds are transferred in `dismissDispute()`. Ensure similar adjustments in `_confiscateDisputers()` for consistency. 

This approach prevents unauthorized bond reclamation after a dispute is dismissed, preserving the integrity of the arbitration process and protecting against exploitative claims.


  **Link**: [Issue #37](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/37)

## Medium severity issues


- **Tokens in TokenLock Contract Permanently Stuck After Revocation and Destruction**

  In the `TokenLock.revoke()` function, unvested tokens are sent to the owner and the contract is subsequently destroyed. However, the total tokens in `TokenLock` include unvested tokens, vested tokens unreleased by the beneficiary (accessible via `releasableAmount()`), and surplus tokens (`surplusAmount()`). Upon contract destruction, `releasableAmount()` and `surplusAmount()` become unrecoverable, effectively locking these tokens permanently. To prevent this, it is recommended to transfer `releasableAmount() + surplusAmount()` to either the owner or the beneficiary before destroying the contract. Suggested code changes demonstrate how to safely return these funds either to the owner or the beneficiary, ensuring all tokens are retrieved.


  **Link**: [Issue #54](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/54)

## Low severity issues


- **Deprecated transfer and send functions cause transaction failures and potential fund loss**

  Using the deprecated `transfer` function can cause transaction failures under specific conditions, such as when using multisig wallets requiring more than 2300 gas. The `send()` function also has similar issues, lacking checks for successful fund transfers. Switching to the `call()` function is recommended to prevent potential fund loss. Affected lines are HatKlerosConnector (#168, #322) and HatKlerosConnectorV2 (#100).


  **Link**: [Issue #16](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/16)


- **Potential Fund Loss if Validation Fails in predictSplitterAddress in HATPaymentSplitterFactory**

  The `predictSplitterAddress` function in `HATPaymentSplitterFactory` may predict addresses that can't be created due to input validation in `HATPaymentSplitter.__PaymentSplitter_init`. This could result in permanently lost funds if users send funds to these undeployable addresses. It is recommended to add the same input validation to `predictSplitterAddress` to prevent this issue.


  **Link**: [Issue #48](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/48)



## Conclusion

The Hats.finance audit competition, focusing on the security of HATs Arbitration Contracts, showcased a decentralized approach to smart contract auditing through competitive bug hunting. Over two weeks, 74 submissions were made, with $24,900.4 distributed among eight participants. The audit identified several critical vulnerabilities. A significant high-severity issue was found in the `refundExpiredSubmitClaimRequest()` function, which could allow attackers to drain all funds from the `HATArbitrator` contract. Mitigations were recommended involving proper verification and access control in the refund logic. Another high-severity bug in `HATArbitrator.reclaimBond()` could allow improper refunds of previously dismissed claims, necessitating changes to prevent unauthorized bond reclamation. Additionally, medium-severity issues were noted, such as potentially stuck tokens in the `TokenLock` contract upon revocation. Low-severity issues included deprecated functions causing potential transaction failures and fund loss. Overall, the competition successfully highlighted vulnerabilities needing resolution to enhance the security framework of the arbitration contracts.

## Disclaimer


This report does not assert that the audited contracts are completely secure. Continuous review and comprehensive testing are advised before deploying critical smart contracts.


The HATs Arbitration Contracts audit competition illustrates the collaborative effort in identifying and rectifying potential vulnerabilities, enhancing the overall security and functionality of the platform.


Hats.finance does not provide any guarantee or warranty regarding the security of this project. Smart contract software should be used at the sole risk and responsibility of users.

