# Payment Processor - Sapphire DAO - Findings Report

# Table of contents

- ## [Audit Summary](#audit-summary)
- ## [Findings Summary](#findings-summary)
- ## High Risk Findings
    - ### [H-01. Invoices for which the hold period is manually set by admin are bricked and payment cannot be collected by the invoice creator](#M-01)
- ## Medium Risk Findings
    - ### [M-01. An invoice cannot be paid if the protocol fee is set to greater than or equal to the invoice price](#M-01)
- ## Low Risk Findings
    - ### [L-01. Protocol fees is deducted even from rejected invoices](#L-01)

# <a id='audit-summary'></a>Audit Summary

### Protocol: Sapphire DAO

### Dates: Mar 18th, 2025 - Mar 19th, 2025

# <a id='findings-summary'></a>Findings Summary

### Number of findings:

- High: 1
- Medium: 1
- Low: 1
- Informational: 0

# High Risk Findings

## <a id='H-01'></a>H-01. Invoices for which the hold period is manually set by admin are bricked and payment cannot be collected by the invoice creator

## Summary

An invoice is not released instantaneously to the creator once it's accepted. Instead, it is kept on hold for a `defaultHoldPeriod` before the funds are released. Additionally, the hold period can be manually set by the admin at any point in the lifespan of an invoice. If the hold period for an invoice is set before the invoice is accepted, then after the invoice acceptance, the payment is bricked, and the creator cannot collect the funds for an extremely  

## Root Cause

Consider the code segment [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L201C1-L215C6). The hold period is set as a UNIX timestamp in seconds. However, [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L180C1-L181C72) the pre-set hold period is treated as a duration in seconds. Thus, the current timestamp is added to the previously set timestamp. This makes the hold period extremely long (around 110 years in the test case provided below). The hold period can then not be set to a smaller duration by the admin because of the check [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L210C1-L212C10). There is no way to rescue the funds.

## Internal pre-conditions

The admin manually sets the hold period before the invoice payment acceptance by the creator.

## Impact

The invoice payment collection will fail by the creator. The funds will be locked up in the escrow for an extremely long time (>110 years at the time of writing).

## PoC

Add the following test case to payment-processor/test/unit/PaymentProcessorTest.t.sol,

```js
    function test_manuallySettingTheHoldPeriodByAdminBricksPaymentCollection() public {
        uint256 invoicePrice = 1.5 ether;

        vm.prank(creatorOne);
        uint256 invoiceId = pp.createInvoice(invoicePrice);

        vm.prank(payerOne);
        pp.makeInvoicePayment{ value: invoicePrice }(invoiceId);

        uint32 holdPeriod = 2 days;

        vm.prank(owner);
        pp.setInvoiceHoldPeriod(invoiceId, holdPeriod);

        vm.prank(creatorOne);
        pp.creatorsAction(invoiceId, true);

        uint256 expectedHoldPeriod = block.timestamp + holdPeriod;
        Invoice memory invoiceData = pp.getInvoiceData(invoiceId);

        console.log(expectedHoldPeriod, invoiceData.holdPeriod);
        assert(invoiceData.holdPeriod > expectedHoldPeriod);
    }
```

Run the test with the command: `forge test --mt test_manuallySettingTheHoldPeriodByAdminBricksPaymentCollection -vvvv --fork-url "https://arbitrum-sepolia-rpc.publicnode.com"` (using Arbitrum Sepolia to better understand the impact), which will output the following:

```shell
Ran 1 test for test/unit/PaymentProcessorTest.t.sol:PaymentProcessorTest
[PASS] test_manuallySettingTheHoldPeriodByAdminBricksPaymentCollection() (gas: 399530)
Logs:
  1742487966 3484803132

Traces:
  [399530] PaymentProcessorTest::test_manuallySettingTheHoldPeriodByAdminBricksPaymentCollection()
    ├─ [0] VM::prank(creatorOne: [0xB0774252dbDB601F216B2bEDf34467429F9Bd5d9])
    │   └─ ← [Return] 
    ├─ [83837] PaymentProcessorV1::createInvoice(1500000000000000000 [1.5e18])
    │   ├─ emit InvoiceCreated(invoiceId: 1, creator: creatorOne: [0xB0774252dbDB601F216B2bEDf34467429F9Bd5d9], price: 1500000000000000000 [1.5e18])
    │   └─ ← [Return] 1
    ├─ [0] VM::prank(payerOne: [0x26D463F4d8d7aA390EE0cF6C751505781bE69a59])
    │   └─ ← [Return] 
    ├─ [273959] PaymentProcessorV1::makeInvoicePayment{value: 1500000000000000000}(1)
    │   ├─ [176167] → new Escrow@0xf801f3A6F4e09F82D6008505C67a0A5b39842406
    │   │   ├─ emit FundsDeposited(invoiceId: 1, value: 500000000000000000 [5e17])
    │   │   └─ ← [Return] 869 bytes of code
    │   ├─ emit InvoicePaid(invoiceId: 1, payer: payerOne: [0x26D463F4d8d7aA390EE0cF6C751505781bE69a59], amountPaid: 1500000000000000000 [1.5e18])
    │   └─ ← [Return] Escrow: [0xf801f3A6F4e09F82D6008505C67a0A5b39842406]
    ├─ [0] VM::prank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [5652] PaymentProcessorV1::setInvoiceHoldPeriod(1, 172800 [1.728e5])
    │   ├─ emit UpdateHoldPeriod(invoiceId: 1, releaseDueTimestamp: 1742487966 [1.742e9])
    │   └─ ← [Stop] 
    ├─ [0] VM::prank(creatorOne: [0xB0774252dbDB601F216B2bEDf34467429F9Bd5d9])
    │   └─ ← [Return] 
    ├─ [5821] PaymentProcessorV1::creatorsAction(1, true)
    │   ├─ emit InvoiceAccepted(invoiceId: 1)
    │   └─ ← [Stop] 
    ├─ [1930] PaymentProcessorV1::getInvoiceData(1) [staticcall]
    │   └─ ← [Return] Invoice({ creator: 0xB0774252dbDB601F216B2bEDf34467429F9Bd5d9, payer: 0x26D463F4d8d7aA390EE0cF6C751505781bE69a59, escrow: 0xf801f3A6F4e09F82D6008505C67a0A5b39842406, price: 1500000000000000000 [1.5e18], amountPaid: 500000000000000000 [5e17], createdAt: 1742315166 [1.742e9], paymentTime: 1742315166 [1.742e9], holdPeriod: 3484803132 [3.484e9], status: 2 })
    ├─ [0] console::log(1742487966 [1.742e9], 3484803132 [3.484e9]) [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.09s (744.99ms CPU time)
```

## Mitigation

Only allow modifying the hold period for an invoice after it is accepted by the creator.


# Medium Risk Findings

## <a id='M-01'></a>M-01. An invoice cannot be paid if the protocol fee is set to greater than or equal to the invoice price

## Summary

The protocol charges a static fee in native gas tokens to successfully paid invoices. When an invoice is created, a check is performed to see that the invoice price is greater than the protocol fee. The payment for the invoice is then made and the protocol fee is subtracted from the paid amount. However, if the protocol fee is changed between the time when the invoice is created and when it's paid to be equal to or greater than the invoice price, then the invoice cannot be paid.

## Root Cause

Consider the code segment [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L92). The user passes the native gas token along with the function call for invoice payment. The user also cannot pass an amount greater than the invoice price as can be seen [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L84). If the current protocol fee is greater than or equal to the invoice price, then the payment will fail.

## Internal pre-conditions

The protocol fee is set to greater than or equal to the invoice price.

## Impact

The invoice payment will fail, essentially rendering the invoice useless.

## PoC

Add the following test case to payment-processor/test/unit/PaymentProcessorTest.t.sol,

```js
    function test_invoicePaymentFailsIfFeeIsSetEqualToOrGreaterThanTheInvoicePriceAfterInvoiceCreation(
    ) public {
        uint256 invoicePrice = 1.5 ether;

        vm.prank(creatorOne);
        uint256 invoiceId = pp.createInvoice(invoicePrice);

        uint256 newFees = 1.5 ether;

        vm.prank(owner);
        pp.setFee(newFees);

        vm.expectRevert(IPaymentProcessorV1.ValueIsTooLow.selector);
        vm.prank(payerOne);
        pp.makeInvoicePayment{ value: invoicePrice }(invoiceId);
    }
```

Run the test with the command: `forge test --mt test_invoicePaymentFailsIfFeeIsSetEqualToOrGreaterThanTheInvoicePriceAfterInvoiceCreation -vvvv`, which will output the following:

```shell
Ran 1 test for test/unit/PaymentProcessorTest.t.sol:PaymentProcessorTest
[PASS] test_invoicePaymentFailsIfFeeIsSetEqualToOrGreaterThanTheInvoicePriceAfterInvoiceCreation() (gas: 114359)
Traces:
  [114359] PaymentProcessorTest::test_invoicePaymentFailsIfFeeIsSetEqualToOrGreaterThanTheInvoicePriceAfterInvoiceCreation()
    ├─ [0] VM::prank(creatorOne: [0xB0774252dbDB601F216B2bEDf34467429F9Bd5d9])
    │   └─ ← [Return] 
    ├─ [83837] PaymentProcessorV1::createInvoice(1500000000000000000 [1.5e18])
    │   ├─ emit InvoiceCreated(invoiceId: 1, creator: creatorOne: [0xB0774252dbDB601F216B2bEDf34467429F9Bd5d9], price: 1500000000000000000 [1.5e18])
    │   └─ ← [Return] 1
    ├─ [0] VM::prank(owner: [0x7c8999dC9a822c1f0Df42023113EDB4FDd543266])
    │   └─ ← [Return] 
    ├─ [5385] PaymentProcessorV1::setFee(1500000000000000000 [1.5e18])
    │   └─ ← [Stop] 
    ├─ [0] VM::expectRevert(ValueIsTooLow())
    │   └─ ← [Return] 
    ├─ [0] VM::prank(payerOne: [0x26D463F4d8d7aA390EE0cF6C751505781bE69a59])
    │   └─ ← [Return] 
    ├─ [1694] PaymentProcessorV1::makeInvoicePayment{value: 1500000000000000000}(1)
    │   └─ ← [Revert] ValueIsTooLow()
    └─ ← [Stop] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.05ms (256.11µs CPU time)
```

## Mitigation

Consider not charging a static fee on invoices, rather take a small percentage of the invoice price as the protocol fee. Such as 2-5% of the invoice price.


# Low Risk Findings

## <a id='L-01'></a>L-01. Protocol fees is deducted even from rejected/unprocessed invoices

## Summary

When an invoice is paid, an escrow contract is created which holds the paid native gas token amount minus the protocol fee, which is collected in the `PaymentProcessor` contract. When an invoice payment is rejected or left unprocessed after the acceptance window, the payer is refunded from the escrow, however, the protocol fee is not returned. Protocol fee should only be deducted on successful invoice payments.

## Root Cause

Consider the code segment [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L96), [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L160C1-L169C6), and [here](https://github.com/SapphireDAOO/payment-processor/blob/bf02d4a62391018391fee815a8427dc0924aa6c5/src/PaymentProcessorV1.sol#L194C1-L198C6). When the invoice payment is made, the protocol fee is collected within the `PaymentProcessor` contract, and the remaining funds are held in the `Escrow` contract. On refund issuance, only the funds locked in the escrow are returned, and the fees paid still remains with the protocol.

## Impact

Users facing unprocessed/rejected invoice payments will lose fees paid on refund. Protocol fee should only be charged on successfully accepted invoices.

## Mitigation

Return the protocol fee from the `PaymentProcessor` contract while issuing a refund.