# Fee Refund

Polymarket has a contract in the project [exchange-fee-module](https://github.com/Polymarket/exchange-fee-module) that proxies `matchOrders` to the CTF exchange. The fee module acts as an operator of the CTF exchange, which receives trading fees and the contract admin can decide how much refund to give to each maker and the taker. `takerFeeAmount` is the final fee amount to charge the taker and `makerFeeAmounts` is an array of final fee amounts to charge the makers.

```solidity
/// @notice Matches a taker order against a list of maker orders, refunding maker order fees if necessary
/// @param takerOrder           - The active order to be matched
/// @param makerOrders          - The array of maker orders to be matched against the active order
/// @param takerFillAmount      - The amount to fill on the taker order, always in terms of the maker amount
/// @param takerReceiveAmount   - The amount to that will be received by the taker order, always in terms of the taker amount
/// @param makerFillAmounts     - The array of amounts to fill on the maker orders, always in terms of the maker amount
/// @param takerFeeAmount       - The fee to be charged to the taker
/// @param makerFeeAmounts      - The fee to be charged to the maker orders
function matchOrders(
    Order memory takerOrder,
    Order[] memory makerOrders,
    uint256 takerFillAmount,
    uint256 takerReceiveAmount,
    uint256[] memory makerFillAmounts,
    uint256 takerFeeAmount,
    uint256[] memory makerFeeAmounts
) external onlyAdmin {
    // Match the orders on the exchange
    exchange.matchOrders(takerOrder, makerOrders, takerFillAmount, makerFillAmounts);

    // Refund taker fees
    _refundTakerFees(takerOrder, takerFillAmount, takerReceiveAmount, takerFeeAmount);

    // Refund maker fees
    _refundMakerFees(makerOrders, makerFillAmounts, makerFeeAmounts);
}
```

`_refundTakerFees` calculates the expected refund amount. The original fee amount to pay is calculated from the CTF exchange's fee formula, then if the operator specified fee amount is lower than the calculated fee amount, take the difference between the two as the refund amount. Otherwise there is no refund.

```solidity
function calculateRefund(
    uint256 orderFeeRateBps,
    uint256 operatorFeeAmount,
    uint256 outcomeTokens,
    uint256 makerAmount,
    uint256 takerAmount,
    Side side
) internal pure returns (uint256) {
    // Calculates the fee charged by the exchange
    uint256 exchangeFeeAmount = calculateExchangeFee(orderFeeRateBps, outcomeTokens, makerAmount, takerAmount, side);

    // Exchange fee must be greater than the operator fee
    if (exchangeFeeAmount <= operatorFeeAmount) return 0;

    return exchangeFeeAmount - operatorFeeAmount;
}
```

It then calls `_refundFee` which transfers USDC or CTF tokens to the taker.

```solidity
function _refundFee(bytes32 orderHash, uint256 id, address to, uint256 refund, uint256 feeAmount) internal {
    // If the refund is non-zero, transfer it to the order maker
    if (refund > 0) {
        address token = id == 0 ? collateral : ctf;
        _transfer(token, address(this), to, id, refund);
        emit FeeRefunded(orderHash, to, id, refund, feeAmount);
    }
}
```

`_refundMakerFees` contains basically the same logic as `_refundTakerFees`, except it has to loop through each maker order.

```solidity
function _refundMakerFees(Order[] memory orders, uint256[] memory fillAmounts, uint256[] memory feeAmounts)
    internal
{
    for (uint256 i = 0; i < orders.length; ++i) {
        Order memory order = orders[i];
        _refundFee(
            exchange.hashOrder(order),
            order.side == Side.BUY ? order.tokenId : 0,
            order.maker,
            _calculateMakerRefund(order, fillAmounts[i], feeAmounts[i]),
            feeAmounts[i]
        );
    }
}
```
