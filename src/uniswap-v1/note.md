## uniswap v1概述
uniswap 对自动做市商（AMM-Automatic Market Maker）进行了尝试， 利用预设函数基于两头货币在各自流动性池子中的供给变化设定价格（x * y = k），用户可以提供货币增加流动性，获取收益。

## 一些概念
1. 基础公示：x*y=k，x和y各代表一种货币，乘积为常量。CPAMM(constant product auto market maker)
2. LP: Liquidity Provider，流动性；
3. LPT: Liquidity Provider Token，提供流动性的凭证，发给你新的token。
4. 初始流动性决定价格

## 不足
1. uniswap v1本质上只支持token 和 eth 之间的兑换，其他代币之间的swap实际上也是通过eth作为桥梁(token A --> eth --> token B)
2. 无偿损失：无偿损失是指流动性提供者因为代币价格波动而遭受的潜在损失
  ----AMM 使用恒定乘积公式（x * y = k）来保持流动性池的平衡。当交易发生时，池中的代币数量会变化以维持这个公式，从而导致价格变化。如果一个代币的价格相对于另一个代币大幅上涨或下跌，流动性提供者的资产组合会被重新平衡，导致潜在的损失。
3. 滑点：交易执行价格与预期价格之间的差异
  --p1 = x/y, p2 = △x/△y，p3 = (△x+x)/(y-△y)
  p1:交易前中间价
  p2:交易执行价
  p3:交易后中间价
  中间价和执行价是不一样的，称作滑点。