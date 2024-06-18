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


- **Vulnerability in refundExpiredSubmitClaimRequest Function Allows Draining of HATArbitrator Contract**

  An attacker can exploit the `refundExpiredSubmitClaimRequest()` function in the `HATArbitrator` contract to drain user funds indefinitely. The primary issue arises from the function's ability to be called with a non-existing `submitClaimRequest`, allowing anyone to bypass several checks and perform unauthorized fund transfers. Specifically, the function does not verify whether the `submitClaimRequest` actually exists, leading to potential bypasses where unverified claim data results in inappropriate actions.

Here are the critical vulnerabilities:
1. The function accepts non-existing `submitClaimRequest` inputs.
2. The condition within the function can be bypassed because it assumes the `submittedAt` timestamp will always be valid, which is set to zero for non-existing claims.
3. The submitter address is not validated, potentially allowing transfers to a zero address.
4. The transfer amount is hardcoded instead of using the actual bond amount, making it susceptible to misuse.
5. The function can be invoked by any user, not limited to the claim submitter.

Additionally, the SafeERC20 library does not prevent transfers to the zero address, and many popular tokens support such transfers, further exacerbating the problem.

To reproduce and validate this issue, a step-by-step setup using DAI as an example token demonstrates how the contract can be drained. Immediate remediation includes adding proper validations for the `submitClaimRequest`, ensuring the transfer amount reflects the actual bond, and restricting function access to the submitter only. The proposed code adjustments effectively address these concerns by adding error checks and restricting unauthorized access.


  **Link**: [Issue #36](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/36)


- **Bonds Refund Exploit in `reclaimBond()` Allows Attacker to Spam Disputes**

  The function `HATArbitrator.reclaimBond()` currently refunds previously dismissed claims erroneously, creating a vulnerability where malicious users can make spam disputes and then reclaim the bonds using the funds of successive claims. This can occur at the cost of the Expert Committee or future disputers.

In an attack scenario, a user submits a claim (Claim A), and another user disputes it by placing a bond. If the Expert Committee dismisses this dispute, the bonds are transferred to them. When a new claim (Claim B) is submitted and disputed with a new bond, the first user erroneously reclaims their bond for the dismissed Claim A, reducing the overall balance available for future disputes. This invalidates subsequent transactions when the Expert Committee attempts to dismiss the new dispute, causing failures due to an insufficient balance.

The proposed fix involves updating the `reclaimBond()` function to subtract the reclaimed bond values from the total tracked bonds for a claim, ensuring that calls to `reclaimBond()` will fail after a dispute is dismissed, thus preventing misuse. Additionally, modifications are recommended for the `_confiscateDisputers()` function to maintain consistency in the value tracked by `totalBondsOnClaim` and to reset `totalBondsOnClaim[_vault][_claimId]` to zero after transferring bonds to the Expert Committee when dismissing a dispute. These changes solidify the bond management, preventing unauthorized bond reclamations.


  **Link**: [Issue #37](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/37)

## Medium severity issues


- **Unvested tokens and excess funds permanently locked in TokenLock upon revocation**

  In the `TokenLock.revoke()` function, unvested tokens are sent to the owner and the contract is subsequently destroyed. However, this leaves behind two other types of tokens within the contract: vested tokens not yet released by the beneficiary, and surplus tokens. These are calculated using the `releasableAmount()` and `surplusAmount()` functions. Since the contract is destroyed after revocation, these tokens become permanently inaccessible. It is recommended to return both `releasableAmount()` and `surplusAmount()` either to the owner or the beneficiary before destroying the contract, ensuring all tokens are accounted for. Implementing this change will prevent tokens from being irretrievably locked within the contract.


  **Link**: [Issue #54](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/54)

## Low severity issues


- **Deprecated Transfer and Send Functions Cause Transaction Failures in Some Wallets**

  Using the deprecated transfer function can cause transactions to fail under various conditions, such as when multisig wallets require more than 2300 gas to receive funds. The send() function also has this issue, leading to possible fund loss without additional checks. To fix this, the call() function should be used instead. Affected files include HatKlerosConnector and HatKlerosConnectorV2.


  **Link**: [Issue #16](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/16)


- **Potential Fund Loss Due to Lack of Input Validation in predictSplitterAddress Function**

  The function `predictSplitterAddress` in `HATPaymentSplitterFactory` may predict addresses that can never be created due to input validation failure in `HATPaymentSplitter.__PaymentSplitter_init`. This could lead to irreversible loss of funds if users send money to these addresses before the payment splitter is deployed. Adding input validation to `predictSplitterAddress` is recommended.


  **Link**: [Issue #48](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/48)



## Conclusion

The HATs Arbitration Contracts Audit Competition, hosted by Hats.finance, seeks to enhance Web3 project security through decentralized and incentive-based audit competitions. Offering a maximum reward of $28,000, this particular event led to a total payout of $24,900.4 across eight participants. The competition specifically focused on reviewing changes in the HATs system since version 2.0, including the integration of an arbitration procedure and the separation of contract functionalities.

Two high-severity vulnerabilities were identified: one in the `refundExpiredSubmitClaimRequest` function that allowed indefinite draining of user funds, and another in the `reclaimBond` function that permitted spam disputes and unauthorized bond reclamation. Both issues were thoroughly analyzed and corrective measures were proposed.

Additionally, medium-severity issues included permanent locking of unvested tokens and excess funds upon contract revocation. Low-severity issues involved deprecated transfer functions and potential fund loss due to inadequate input validation. Overall, the audit highlighted critical areas needing improvement, reinforcing Hats.finance’s commitment to robust and secure decentralized auditing mechanisms for Web3 projects.

## Disclaimer


This report does not assert that the audited contracts are completely secure. Continuous review and comprehensive testing are advised before deploying critical smart contracts.


The HATs Arbitration Contracts audit competition illustrates the collaborative effort in identifying and rectifying potential vulnerabilities, enhancing the overall security and functionality of the platform.


Hats.finance does not provide any guarantee or warranty regarding the security of this project. Smart contract software should be used at the sole risk and responsibility of users.

