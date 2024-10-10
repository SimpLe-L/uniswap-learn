## uniswap v3
v3 版本相较于v2有了以下改进：
1. 引入了 集中流动性（Concentrated Liquidity） 机制
  添加流动性时选择价格区间，只有用户在该价格区间swap，才能收到手续费
2. LP Token 改为了不可互换的 NFT
  v2中流动性是整个池子共享的，每个人占有多少流动性可以用份额来表示，而份额是可以互换的，所以可以用 ERC20 Token 来作为流动性代币。但 UniswapV3 中，流动性增加了价格区间的限制之后，就不再是共享的了，每一次添加的流动性都基本是独一无二的，因此，已经不适合继续使用 ERC20 来作为流动性代币，但使用 ERC721 却非常合适，每一次添加的流动性也称为一个头寸（position），就用一个单独的 NFT 来表示。
3. 每个交易对可以有多个不同费率的池子
  每个交易对有且仅有一个流动性池，且交易手续费率都是统一的千分之三。但 UniswapV3 可以为同个交易对创建不同费率的池子，即 token0、token1 加上 fee 三个字段组成了一个 Pool 的唯一性。
4. 协议手续费治理更灵活
  v2是1/6，v3可配置1/N 或 0(4 <= N <= 10)
5. 改进了价格预言机
  V2 的方式是直接记录价格的累加值，使用时再除以时间间隔，这是一种算术平均。而 V3 累加的是 log(price, 1.0001) 也就是价格的幂，使用时再除以时间间隔，这是几何平均。几何平均数相比算术平均数，能更好的反应真实的价格，受短期波动影响更小。

## 核心代码
1. UniswapV3Factory
该合约用于创建不同代币对的流动性池子，构造函数除了初始化 owner 之外，最主要就是初始化 feeAmountTickSpacing 状态变量。这个变量是用来存储支持的交易手续费率的配置的，key 代表费率，value 代表 tickSpacing。初始的费率值分别设为了 500、3000、10000，分别代表了 0.05%、0.3%、1%。

--核心函数：createPool
三个入参就是组成一个池子唯一性的 tokenA、tokenB 和 fee。调用 deploy 函数返回 pool 后，会存储到 getPool 状态变量里。deploy定义了结构体 Parameters 和该结构体类型的状态变量 parameters。然后，在 deploy 函数里，先对 parameters 进行赋值，接着通过 new UniswapV3Pool 部署了新池子合约，使用 token0、token1 和 fee 三个字段拼接的哈希值作为盐值。最后再将 parameters 删除。总共就三行代码。但其中有两个用法，是在以前的项目中还没出现过的：
    使用 new UniswapV3Pool 部署新合约时，还可以指定 salt。这其实也是 create2 的一种新写法，相比于 UniswapV2Factory 中使用内联汇编的方式，明显简化了很多。
    parameters 其实是传给 UniswapV3Pool 的参数，在 UniswapV3Pool 的构造函数里，是这样来接收这些参数的：(factory, token0, token1, fee, _tickSpacing) = IUniswapV3PoolDeployer(msg.sender).parameters(); msg.sender 其实就是工厂合约。

2. UniswapV3Pool

--核心函数：mint，添加流动性的底层函数
  添加流动性的主要操作其实是在 _modifyPosition 私有函数里，执行完该函数后，返回值包括了需要添加到池子里的两种 token 的具体数额 amount0 和 amount1,之后，查询并临时记录下两种 token 在池子里的当前余额。然后，调用 msg.sender 的回调函数 uniswapV3MintCallback，在回调函数中需要完成两种 token 的支付。msg.sender 一般是 NonfungiblePositionManager 合约
--核心函数：burn
  该函数移除的是 msg.sender 的流动性头寸。其有三个入参，tickLower 和 tickUpper 用来指定要移动哪个头寸，amount 指定要移除的流动性数额。
  和 mint 的时候一样，第一步核心操作也是先 _modifyPosition。不过，因为是减少流动性，所以传入的 liquidityDelta 参数转为负数。而返回的 amount0Int 和 amount1Int 也会是负数，所以转为 uint256 类型的 amount0 和 amount1 时，又需要加上负号将负数再转为正数。之后，将 amount0 和 amount1 分别累加到了头寸的 tokensOwed0 和 tokensOwed1。与 v2 不同的是，UniswapV3 的处理方式并不是移除流动性时直接把两种 token 资产转给用户，而是先累加到 tokensOwed0 和 tokensOwed1，代表这是欠用户的资产，其中也包括该头寸已赚取到的手续费。之后，用户其实是要通过 collect 函数来提取 tokensOwed0 和 tokensOwed1 里的资产。
--核心函数：collect
  5 个入参，recipient 就是接收 token 的地址，tickLower 和 tickUpper指定了头寸区间，amount0Requested 和 amount1Requested 是用户希望提取的数额。返回值 amount0 和 amount1 就是实际提取的数额。通过 msg.sender、tickLower、tickUpper 来读取出用户的头寸。接着判断用户希望提取的数额 amount0Requested 和头寸里的 tokensOwed0 哪个值小就实际提取哪个，amount1 的也同样。之后就是从头寸的 tokensOwed 里减掉提取的数额并转账给接收地址。最后发送 Collect 事件。
--核心函数：swap
  一笔交易有时候会跨越多个流动性区间，所以需要使用循环处理在每一个区间内的交易。当剩余可交易金额已经消耗完，或价格已经达到了指定的限定价格后，循环也就结束了，即交易主流程结束了。之后就是一些交易收尾的工作了，包括更新 tick、价格、流动性、手续费增长系数等。最后很关键的一步就是做转账和支付，先将 tokenOut 转给了用户，然后执行了回调函数 uniswapV3SwapCallback，在回调函数里会完成 tokenIn 的支付，执行完回调函数后的余额校验是为了确保回调函数确实完成了 tokenIn 的支付。因为先将 tokenOut 转给了用户，之后才完成支付，因此在回调函数中其实还可以做和 UniswapV2 一样的 flash swap。
--核心函数：flash
  实现了闪电贷功能，与 flash swap 不同，闪电贷借什么就需要还什么。另外，UniswapV3 的闪电贷可以两种 token 都借。入参有 4 个，recipient 是接收所贷 token 的地址，amount0 和 amount1 是所要借贷的两个 token 数量，data 是给回调函数的参数。还款则需在 uniswapV3FlashCallback 回调函数中完成。最终，闪电贷赚取的手续费也是分配给 LP 和协议费。


3. NonfungiblePositionManager
实现用户层面的流动性头寸管理，NonfungiblePositionManager 继承了 ERC721，所有 LP Token（即头寸）都是在 NonfungiblePositionManager 合约里进行管理的。

--创建并初始化流动性池（createAndInitializePoolIfNecessary）
  根据 token0、token1、fee 这三个字段从工厂合约查询 pool 是否已经存在。如果不存在的话，则通过工厂合约的 createPool 函数创建池子，再通过 IUniswapV3Pool （即池子的合约）的 initialize 函数初始化池子的价格和 tick 等状态。如果从工厂合约查询到 pool 已经存在了，那就通过读取池子合约的 slot0 来获取到其当前价格，如果价格为 0 就表示还没初始化，所以再执行一次 initialize。
--创建新头寸
  通过结构体 MintParams 传参给mint函数创建头寸，第一步是调用了内部函数 addLiquidity 来添加流动性，其中的 pool 地址是通过 PoolAddress 库的 computeAddress 函数计算。流动性liquidity的计算可分为三种情况：
  (1). 当前价格小于价格区间下限时，流动性 L 的计算公式:liquidity = amount0 * (sqrt(upper) * sqrt(lower)) / (sqrt(upper) - sqrt(lower))
  (2). 当前价格大于价格区间上限时，则使用如下计算公式：liquidity = amount1 / (sqrt(upper) - sqrt(lower))
  (3). 当前价格处于价格区间内时，以上两个公式都计算，然后取最小值的那个
  之后，就调用 pool 合约的 mint 函数实现底层的添加流动性操作了，会返回实际成交的 amount0 和 amount1。执行完 addLiquidity 函数之后，调用了 _mint 函数，这其实是 ERC721 实现的内部函数，也是铸造新 NFT 的函数，第一个参数是接收该 NFT 的地址，第二个参数是该 NFT 的 tokenId。每个 NFT 就是一个头寸。最后，再计算出 poolId 等数据，创建一个 Position 对象实例，以 tokenId 为 key，存储到 _positions 里。_positons 是一个 mapping 类型的私有状态变量，用于存储所有头寸对象。
--为头寸增加流动性
  increaseLiquidity函数实现在已有头寸上再增加流动性，实现逻辑就是使用 tokenId 为 key，从 _positions 读取出 Position 对象，后面的逻辑主要目的也是更新该 Position 对象的数据。然后，也和 mint 函数一样调用了 addLiquidity 函数来完成内部的添加流动性逻辑。之后，会把该头寸内之前的流动性所赚取的手续费收益结算到 Position 对象的 tokensOwed0 和 tokensOwed1 字段，并更新 feeGrowthInside0LastX128 和 feeGrowthInside1LastX128，以及累加流动性 liquidity。
--减少流动性
  decreaseLiquidity函数实现在已有头寸上减少流动性，参除了指定头寸的 tokenId，还指定了 liquidity，这是要移除的流动性数量。减少流动性函数多了另一个函数修饰器 isAuthorizedForToken，这是为了校验调用者 msg.sender 是否有权限操作该头寸对应的 tokenId，只有该 tokenId 的 owner 或获得授权的用户才可以执行减少流动性的操作。先通过 tokenId 从 _positions 读取出 Position 对象，然后校验头寸里的流动性不能小于要移除的流动性。之后，计算出 pool 地址，并调用 pool 合约底层的 burn 函数来实现底层的移除流动性操作。然后，和增加流动性时一样，结算之前的手续费收益并更新手续费相关字段，移除的流动性也相应从头寸中减少。另外，有一点要注意，结算头寸的 tokensOwed 时，除了手续费收益之外，移除流动性计算所得的 amount0 和 amount1 也同样结算到了该字段里。都统一通过 collect 函数进行提取。
--提取手续费收益
  UniswapV2 的手续费收益是在移除流动性时一起提取走的，但 UniswapV3 的手续费收益是单独提取的，通过 collect 函数进行提取。而且，移除流动性结算所得的两个代币也是通过 collect 函数一起提取。入参有 4 个，tokenId 指定要从哪个头寸提取，recipient 是接收地址，amount0Max 和 amount1Max 是要提取的最大数量。提取的代币，是从 Position 对象中的 tokensOwed0 和 tokensOwed1 中提取的。另外 tokensOwed0 和 tokensOwed1 只是记录了上一次结算的收益，因此，还会把从上一次结算到当前的收益也计算出来，也累加到 tokensOwed0 和 tokensOwed1。并更新最新的 feeGrowthInside0LastX128 和 feeGrowthInside1LastX128。而且，还会调用 pool 合约底层的 burn 函数来更新手续费，因为没有移除流动性，所以调用 burn 时的第三个参数为 0。如果最后计算出来的 tokensOwed 大于入参的最大提取数量，那就只会提取最大数量。而实际提取的操作，也是在 pool 合约底层的 collect 函数执行的。
--销毁头寸
  当一个头寸已经没有流动性了，手续费收益等也已经提取完了，则可以将该头寸进行销毁操作。只有该头寸的拥有者或授权者才有权限执行销毁操作。其次，require 里表明了需要该头寸的 liquidity、tokensOwed0、tokensOwed1 都为 0 的情况下才允许销毁。然后，通过 delete _positions[tokenId] 删除该 tokenId 所存储的头寸数据。再通过 _burn(tokenId) 销毁该 NFT。

4. SwapRouter
SwapRouter 合约封装了面向用户的交易接口

--核心函数：exactInputSingle：指定输入数量的单池内交易
  实际的逻辑实现是在内部函数 exactInputInternal。查看该内部函数之前，我们先来了解下 SwapCallbackData。我们从上面代码可以看到，调用 exactInputInternal 时，最后一个传入的参数就是 SwapCallbackData，这其实是一个结构体，定义了两个属性：path （交易路径，由 tokenIn、fee、tokenOut 这三个变量拼接）；payer （付输入 token 的地址，即msg.sender）。exactInputInternal函数：如果 recipient 地址为零地址的话，那会把 recipient 重置为当前合约地址。接着，通过 data.path.decodeFirstPool() 从路径中解码得出 tokenIn、tokenOut 和 fee。decodeFirstPool 函数是在库合约 Path 里实现的。布尔类型的 zeroForOne 表示底层 token0 和 token1 的兑换方向，为 true 表示用 token0 兑换 token1，false 则反之。因为底层的 token0 是小于 token1 的，所以，当 tokenIn 也小于 tokenOut 的时候，说明 tokenIn == token0，所以 zeroForOne 为 true。然后，通过 getPool 函数可得到池子地址，再调用底层池子的 swap 函数来执行实际的交易逻辑。最后，我们要得到的是 amountOut，这是 amount0 和 amount1 中的其中一个。我们已经知道，zeroForOne 为 true 的时候，tokenIn 等于 token0，所以 tokenOut 就是 token1，因此 amountOut 就是 amount1。另外，对底层池子来说，属于输出的时候，返回的数值是负数，即 amount1 其实是一个负数，因此需要再加个负号转为正数的 uint256 类型。
--核心函数：exactOutputSingle：指定输出数量的单池内交易
  exactOutputSingle 函数的实现与 exactInputSingle 函数大同小异。首先，参数上，只有两个不同，exactInputSingle 函数指定的是 amountIn 和 amountOutMinimum；而 exactOutputSingle 函数改为了 amountOut 和 amountInMaximum，即输出是指定的，而输入则限制了最大值。其次，实际逻辑封装在了 exactOutputInternal 内部函数，而且传给该内部函数的最后一个参数的 path 组装顺序也不一样了，排在第一位的是 tokenOut。exactOutputInternal：和 exactInputInternal 的实现也是大同小异。不过调用底层的 swap 函数时，第三个传参转为了负数，这也是前面讲解 UniswapV3Pool 的 swap 函数时讲过的，当指定的交易数额是输出的数额时，则需传负数。
--核心函数：exactInput：指定输入数量和交易路径的交易
  于处理跨多个池子的指定输入数量的交易，相比单池交易会复杂一些。核心实现逻辑是，循环处理路径中的每一个配对池，每处理完一个池子的交易，就从路径中移除第一个 token 和 fee，直到路径只剩下最后一个池子就结束循环。期间，每一次执行 exactInputInternal 后，将返回的 amounOut 作为下一轮的 amountIn。第一轮兑换时，payer 是合约的调用者，即 msg.sender，而输出代币的 recipient 则是当前合约地址。中间的每一次兑换，payer 和 recipient 都是当前合约地址。到最后一次兑换时，recipient 才转为用户传入的地址。
--核心函数：exactOutput：指定输出数量和交易路径的交易
  用于处理跨多个池子的的交易，而指定的是输出的数量。其逻辑就直接调用内部函数 exactOutputInternal 完成交易，并没有像 exactInput 一样的循环处理。但在整个流程中，其实还是进行了遍历路径的多次交易的，只是这个流程完成得比较隐晦
--核心函数：uniswapV3SwapCallback
  swap 时的回调函数。首先把 _data 解码成 SwapCallbackData 结构体类型数据。接着，解码出路径的第一个池子。然后，通过 verifyCallback 校验调用当前回调函数的是否为底层 pool 合约，非底层 pool 合约是不允许调起回调函数的。isExactInput 和 amountToPay 的赋值需要拆解一下才好理解。首先需知道，amount0Delta 和 amount1Delta 其实是一正一负的，正数是输入的，负数是输出的。因此，amount0Delta 大于 0 的话则 amountToPay 就是 amount0Delta，否则就是 amount1Delta 了。 amount0Delta 大于 0 也说明了输入的是 token0，因此，当 tokenIn < tokenOut 的时候，说明 tokenIn 就是 token0，也即是说用户指定的是输入数量，所以这时候的 isExactInput 即为 true。当指定金额为输出的时候，也就是处理 exactOutput 和 exactOutputSingle 函数的时候。我们前面看到 exactOutput 的代码逻辑里并没有对路径进行遍历处理，这个遍历其实就