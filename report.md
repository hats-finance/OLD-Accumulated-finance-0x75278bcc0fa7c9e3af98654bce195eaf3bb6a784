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
- Maximum Reward: $27,972.11
- Submissions: 74
- Total Payout: $24,875.6 distributed among 8 participants.

## Scope of Audit

The scope of the competition includes all the contracts that make up the HATs system, but we are specifically interested in the changes made since v2.0.

There are a number of changes since our last release:

- We added an arbitration procedure: if there is any kind of dispute about the payout, the decision can be  escalated to an Expert Committee, and if that committee cannot find consensus, to the Kleros court. The intended behavior of that system is documented here: https://github.com/hats-finance/hats-contracts/blob/audit/docs/claims.md
- The functionality of the main contract, HATVault.sol, was split into two new files (HATVault.sol and HATClaimsManager.sol) 
- We added some helper contracts, such as the PaymentSplitter.sol

The contracts that are currently in use, v2.0, can be found here https://github.com/hats-finance/hats-contracts/tree/v2.0.  


In this competition, we are interested in the usual programming errors, but we are also interested in any scenarios in which the arbitration procedure can be abused or hijacked. (For such attacks, it is the behavior that is implemented that serves as a reference - pointing out mistakes in documentation is appreciated, but will not be rewarded). 





## High severity issues


- **Vulnerability in HATArbitrator Contract Allows Attacker to Irrecoverably Burn User Funds**

  An attacker can exhaust all funds in the `HATArbitrator` contract by continuously calling the `refundExpiredSubmitClaimRequest()` function with a non-existent submit claim. The vulnerabilities stem from several weaknesses and insufficient validations in the function:

1. The function does not verify if the `submitClaimRequest` exists for a given `_internalClaimId`.
2. The `if` statement checking the claim's expiry can be bypassed since a non-existent claim's `submittedAt` is zero.
3. There is no validation to ensure that the submitter's address is non-zero.
4. The function uses a hardcoded transfer amount instead of the actual claim amount.
5. Anyone can call this function, regardless of whether they submitted a claim.

These issues are amplified by the fact that many popular tokens, such as USDT, DAI, and WETH, allow transfers to the zero address, meaning user funds can be burned inadvertently.

To demonstrate the vulnerability, a proof of concept using `DAI` simulates draining funds from the contract by repeatedly invoking the vulnerable function. 

Recommended mitigations include validating the existence and details (non-zero values for `submittedAt`, `bond`, and `submitter`) of a `SubmitClaimRequest`, using the actual bond amount for transfers, and restricting access to the function to only the original submitters. These changes would prevent unauthorized and erroneous fund transfers to the zero address.


  **Link**: [Issue #36](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/36)


- **Issue with HATArbitrator.reclaimBond Allowing Abuse Through Refunding Dismissed Dispute Claims**

  The function `HATArbitrator.reclaimBond()` incorrectly refunds previously dismissed claims. This flaw enables an attacker to create and dispute spam or malicious claims repeatedly and subsequently get refunded through the bonds of new claims. This action drains resources from the Expert Committee or future disputers.

**Scenario Breakdown:**
1. Claim A is initiated.
2. User 0 disputes Claim A and sends a bond to `HATArbitrator`, which is stored in the `disputersBonds` mapping.
3. The Expert Committee dismisses the dispute, and bonds are transferred to them.
4. A new Claim B is initiated.
5. User 1 disputes Claim B and sends a bond to `HATArbitrator`, stored as before.
6. User 0 reclaims their bond from Claim A, exploiting the system as this should fail.
7. When the Expert Committee dismisses the dispute on Claim B, the bond transfer fails because User 0 has already reclaimed a portion, leaving insufficient balance.

**Proposed Fixes:**
- Adjust the `reclaimBond()` function to subtract the reclaimed values correctly from `totalBondsOnClaim[_vault][_claimId]`, ensuring it fails after `dismissDispute()` is called for that claim.
- Similarly modify `_confiscateDisputers()` for consistency.
- Reset `totalBondsOnClaim[_vault][_claimId]` to zero after the transfer in `dismissDispute()`.

These adjustments ensure the system handles bonds correctly, preventing exploitative refunds.


  **Link**: [Issue #37](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/37)

## Medium severity issues


- **Unvested and Vested Tokens Stuck Permanently in TokenLock After Revoke**

  In the `TokenLock.revoke()` function, when the contract is revoked, only the unvested tokens are sent to the owner, and the contract is subsequently destroyed. This causes an issue where the vested tokens that haven't been released yet and the surplus amount of tokens remain stuck in the contract permanently. To resolve this, it is recommended to return both the vested tokens and the surplus amount to either the owner or the beneficiary before the contract's destruction. Suggested code modifications include transferring these amounts to the owner or beneficiary and ensuring the contract is properly closed without leaving funds trapped.


  **Link**: [Issue #54](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/54)

## Low severity issues


- **Using the Deprecated Transfer Function May Cause Transaction Failures**

  Using the deprecated `transfer` and `send` functions can cause transactions to fail when specific conditions are met, such as when certain multisig wallets require more than 2300 gas. This can lead to potential fund loss. To avoid this, it is recommended to use the `call` function instead. This issue affects code in the `HatKlerosConnector` and `HatKlerosConnectorV2` contracts.


  **Link**: [Issue #16](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/16)


- **Add Input Validation to `predictSplitterAddress` to Prevent Fund Loss**

  The function `predictSplitterAddress` in `HATPaymentSplitterFactory` predicts the address for deploying a `HATPaymentSplitter`. However, input validation in `HATPaymentSplitter.__PaymentSplitter_init` can prevent the creation of certain addresses. Funds sent to these addresses before deployment would be lost. Adding input validation to `predictSplitterAddress` is recommended to address this issue.


  **Link**: [Issue #48](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/48)



## Conclusion

The HATs Arbitration Contracts Audit Competition on Hats.finance evaluated the HATs system, focusing on changes since version 2.0. Over the two-week competition, participants identified vulnerabilities, leading to significant security improvements. The key findings included critical issues in contracts like `HATArbitrator`: a vulnerability allowing attackers to burn funds using the `refundExpiredSubmitClaimRequest()` function and an issue with `reclaimBond()` enabling exploitation through refunding dismissed claims. Mitigation measures were recommended to validate claim existence, restrict function access, and properly handle refunds and bond reclamations. Medium-severity issues were identified in the `TokenLock.revoke()` function, potentially causing tokens to be stuck permanently. Suggested changes included transferring vested tokens and surplus amounts to the owner or beneficiary before contract destruction. Low-severity issues included the use of deprecated `transfer` functions and lack of input validation in `HATPaymentSplitterFactory`. Payouts of $24,875.60 were distributed among eight participants. The audit underscored Hats.finance's commitment to proactive, decentralized security in the Web3 space.

## Disclaimer


This report does not assert that the audited contracts are completely secure. Continuous review and comprehensive testing are advised before deploying critical smart contracts.


The HATs Arbitration Contracts audit competition illustrates the collaborative effort in identifying and rectifying potential vulnerabilities, enhancing the overall security and functionality of the platform.


Hats.finance does not provide any guarantee or warranty regarding the security of this project. Smart contract software should be used at the sole risk and responsibility of users.

