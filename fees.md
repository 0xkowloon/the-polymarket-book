# Fees

Polymarket has a very good introduction on how fees works so I will not go into details. I recommend reading this page

{% @github-files/github-code-block %}

Essentially, the fee a trader pays depends on the share price. It is not right to apply the same fee rate to the same position represented by two sides with different odds. For example, when I buy 100 YES at 1c, I am essentially selling 100 NO at 99c as they represent the same opinion. If the trading fee rate is 1%, I cannot pay 100 shares x 1c x 1% = 1c for buying YES and 100 shares x 99c x 1% = 99c for selling NO as they are the same thing. Hence the formula, no matter when buying or selling, always multiplies the base fee rate by `min(price, 1 - price)`. If the price is 1c, then the min is 1c. If the price is 99c, the min should also be 1c. This makes the fee structure symmetric.

Also, the closer you are to the midpoint (50c), the higher the fee rate you pay. The further you are from the midpoint, the lower the fee rate you pay.
