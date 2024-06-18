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


- **Vulnerability in `refundExpiredSubmitClaimRequest` Function Allows Draining of HATArbitrator Funds**

  An attacker can exploit a function in the `HATArbitrator` contract to drain funds by calling `refundExpiredSubmitClaimRequest()` with non-existent claim requests, burning all user funds in the process. Several vulnerabilities enable this attack: 

1. The function can accept non-existing `submitClaimRequest`.
2. An attacker can bypass the if statement because a non-existing claim will have `submittedAt` set to zero.
3. The function does not verify that the submitter address is non-zero.
4. The transfer amount is hardcoded, not reflecting the actual bond amount.
5. Any user, not just the submitter, can call this function.

Additionally, SafeERC20 does not prevent transfers to the zero address, and many popular tokens allow such transfers. These vulnerabilities together enable complete draining of the contract.

The recommended mitigation includes adding validations to ensure the `submitClaimRequest` is valid, checking that `submittedAt`, `bond`, and `submitter` are non-zero, and ensuring only the actual submitter can call the function. The amount transferred should be based on the actual claim request bond rather than a hardcoded value. Implementing these changes will prevent unauthorized fund drainage.


  **Link**: [Issue #36](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/36)


- **Refunding Dismissed Claims Allows Attackers to Exploit Dispute Bonds**

  The `HATArbitrator.reclaimBond()` function incorrectly refunds previously dismissed claims, allowing attackers to exploit the system by making spam or malicious disputes and reclaiming the bonds of subsequent claims. This occurs at the expense of the Expert Committee or future disputers. The attack scenario involves submitting a claim, disputing it, and having the Expert Committee dismiss the dispute, after which a new claim is filed. Another user then disputes this new claim, enabling the first user to reclaim the previously used bonds, which should not be allowed. This leaves the `HATArbitrator` contract with insufficient funds to settle the new dispute, causing the transfer to fail.

The recommendation to fix this involves subtracting the values reclaimed by disputers from `totalBondsOnClaim[_vault][_claimId]`. Additionally, a similar modification is suggested for `_confiscateDisputers()` to ensure consistent tracking of `totalBondsOnClaim`. Moreover, after transferring bonds to the Expert Committee, `totalBondsOnClaim[_vault][_claimId]` should be reset to zero in the `dismissDispute()` function to prevent further misuse. 

A proof of concept file was provided to illustrate the issue and confirm that the proposed fixes resolve the problem effectively.


  **Link**: [Issue #37](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/37)

## Medium severity issues


- **Ensure Unreleased and Surplus Tokens Are Returned Before `TokenLock` Destruction**

  In the `TokenLock.revoke()` function, the unvested tokens are transferred to the owner and the contract is subsequently destroyed. However, the function only accounts for unvested tokens, leaving behind vested tokens that have not been released by the beneficiary and any surplus amount. These tokens will be permanently stuck in the contract once it is destroyed. To prevent this, it is recommended to also transfer the sum of the releasable and surplus amounts either to the owner or the beneficiary before the contract is terminated. This can be achieved by modifying the function to ensure all tokens are properly redistributed.


  **Link**: [Issue #54](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/54)

## Low severity issues


- **Deprecated Transfer Function Could Fail Transactions for Non-Payable or High Gas Usage**

  Using the deprecated transfer function might cause transactions to fail when the receiver's payable function requires more than 2300 gas, which many multisig wallets do. The send() function has similar issues and lacks checks for successful transfers. The recommended fix is to use the call() function instead. The affected lines are in HatKlerosConnector (#168 and #322) and HatKlerosConnectorV2 (#100).


  **Link**: [Issue #16](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/16)


- **Add Input Validation to `predictSplitterAddress` to Prevent Loss of Funds**

  The `predictSplitterAddress` function in `HATPaymentSplitterFactory` uses `Clones.predictDeterministicAddress` to predict deployment addresses. However, due to input validation in `HATPaymentSplitter.__PaymentSplitter_init`, certain addresses predicted by `predictSplitterAddress` may never be created. Thus, sending funds to these addresses before deployment could result in permanent loss. It is recommended to add input validation to ensure address validity.


  **Link**: [Issue #48](https://github.com/hats-finance/HATs-Arbitration-Contracts-0x79a618f675857b45934ca1c413fd5f409cf89735/issues/48)



## Conclusion

The Hats Audit Competition on Hats.finance aims to enhance Web3 security through decentralized audits. Leveraging a large pool of skilled auditors, it provides a proactive bug-hunting environment. The recent competition assessed HATs Arbitration Contracts with a focus on new changes, notably the arbitration procedure and splitting of the HATVault contract into smaller sections. The competition saw 74 submissions, with $24,900.4 distributed among 8 participants out of a maximum $28,000 reward. High-severity issues included a vulnerability in `refundExpiredSubmitClaimRequest` that could drain funds and exploitation of `reclaimBond` enabling repeated bond reclaiming. Both issues were addressed with detailed mitigation strategies. Medium and low-severity issues highlighted were improper token handling and deprecated transfer functions, both of which received recommended fixes. Overall, the competition effectively identified and proposed solutions for critical vulnerabilities, demonstrating the efficiency of decentralized audit mechanisms in enhancing smart contract security.

## Disclaimer


This report does not assert that the audited contracts are completely secure. Continuous review and comprehensive testing are advised before deploying critical smart contracts.


The HATs Arbitration Contracts audit competition illustrates the collaborative effort in identifying and rectifying potential vulnerabilities, enhancing the overall security and functionality of the platform.


Hats.finance does not provide any guarantee or warranty regarding the security of this project. Smart contract software should be used at the sole risk and responsibility of users.

