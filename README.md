# ExinSwap 开发文档

ExinSwap 是 Exin 旗下的自动化做市(AMM)交易平台，也是 Exin 团队推出的首款 MiFi (Mixin DeFi) 产品。

## 历史版本
[V1 开发文档](https://shuxie.feishu.cn/docs/doccnklwh8P4Wy2aIe3D7TlksSb)

V2 相比 V1 的主要变化：

* 交易支持了稳定币兑换，修改了滑点的设置方式，并增加了最晚执行时间的限制。
* API 调整了 pairs 接口结构，提高效率；

## 交易

要发起交易，需要将待交易的币以指定的 Memo 格式转账给 ExinSwap 机器人 (Mixin ID: `7000102352` User ID: `29f23576-4651-47ff-8c16-6c8a5d76985e`)

### 用户转账 Memo 规范

每个字段用 `$` 隔开，并以 BASE64 编码：

`ACTION$FIELD1$FIELD2$FIELD3$FIELD4`

| 行为 | ACTION | FIELD1 | FIELD2 | FIELD3 | FIELD4 |
| ---- | ---- | ---- | ----| ---- | ---- |
| 交易 | 0 | 目标资产UUID | 最小获取数量。可选。 | 最迟交易时间。可选 | 交易路线。可选 |
| 提供流动性 | 1 | 目标资产UUID | 随机的 uuid，两笔转账该字段应该相同。 |
| 取回流动性 | 2 |


在交易行为中：
* 最小获取数量 是允许成交的最小所得数量；

  如果成交结果少于该数量，会交易失败自动退币；
  
  不传或传入 `0` 则不会因滑点而退币。

* 最迟交易时间 是交易执行结束的最晚 unix 时间，单位是秒。

  如果成交的时间晚于该时间，会交易失败自动退币；

  不传或传入 `0` 则不会因为超时而退币。

* 交易路线 用于指定交易会途径哪些交易对。

  由途径所有资产的 `routeId` (可以通过接口获得) 以下划线组合而成，必须以支付资产开头，以所得资产结尾。

  不传该参数会使用支付资产与所得资产组成的交易对。


以 XIN/USDT 为例：
> 以 XIN 兑换 USDT，可以接受最小的交易所得 USDT 数量是 129.5621

对 `0$4d8c508b-91c5-375b-92b0-ee702ed2dac5$129.5621` 进行 BASE64 即 `MCQ0ZDhjNTA4Yi05MWM1LTM3NWItOTJiMC1lZTcwMmVkMmRhYzUkMTI5LjU2MjE=`

> 以 USDT 兑换 XIN，尽可能成交而不退币，但最晚 `2021-01-07 21:13:26` 前交易结束

对 `0$c94ac88f-4671-3976-b60a-09064f1811e8$0$1610025206`  进行 BASE64 即 `MCRjOTRhYzg4Zi00NjcxLTM5NzYtYjYwYS0wOTA2NGYxODExZTgkMCQxNjEwMDI1MjA2`

> 以 XIN 兑换 USDT，尽可能成交，不限制交易时间，手动指定 XIN-WGT-USDT 路线(假设XIN、USDT、WGT的 `routeId` 依次为 1、2、3)

对 `0$4d8c508b-91c5-375b-92b0-ee702ed2dac5$0$0$1_3_2` 进行 BASE64 即 `MCQ0ZDhjNTA4Yi05MWM1LTM3NWItOTJiMC1lZTcwMmVkMmRhYzUkMCQwJDFfM`

> 为 XIN/USDT 提供流动性

XIN Memo: 

对  `1$4d8c508b-91c5-375b-92b0-ee702ed2dac5$5632c09e-e20d-4ae3-9e12-7996fb06a395` 进行 BASE64 即 `MSQ0ZDhjNTA4Yi05MWM1LTM3NWItOTJiMC1lZTcwMmVkMmRhYzUkNTYzMmMwOWUtZTIwZC00YWUzLTllMTItNzk5NmZiMDZhMzk1`

USDT Memo: 

对 `1$c94ac88f-4671-3976-b60a-09064f1811e8$5632c09e-e20d-4ae3-9e12-7996fb06a395` 进行 BASE64 即 `MSRjOTRhYzg4Zi00NjcxLTM5NzYtYjYwYS0wOTA2NGYxODExZTgkNTYzMmMwOWUtZTIwZC00YWUzLTllMTItNzk5NmZiMDZhMzk1`

> 支付 LP Token 取回之前提供的 XIN 与 USDT

对 `2` 进行 BASE64 即 `Mg==`


### 服务器转账 Memo 规范
每个字段会以 `|` 隔开，并以 BASE64 编码：

RESULT|TRACE|SOURCE|TYPE

| 字段 | RESULT | TRACE | SOURCE | TYPE |
| ---- | ---- | ---- | ---- | ---- |
| 描述 | 成功 0 /失败 1 | 用以追踪来源的UUID | 行为来源 | 行为类型 |
| 由于不符合规则等原因，执行失败退币 | 1 | 用户支付的Trace ID | SN (SNapshot) | RF (RFund) |
| 交易 | 成功 0 /失败 1 | 用户支付的Trace ID | SW (SWap) | 成功 RL (ReLease)<br/>失败 RF (RFund) 
| 存入 | 成功 0 /失败 1 | 存入Memo中，用来对应两笔入账的 UUID | AL (Add Liquidity) | 放币 RL (ReLease)<br/>初始化资金池 IN (INit)<br/>失败 RF (RFund) |
| 取出 | 成功 0 /失败 1 | 用户支付的Trace ID | RL (Remove Liquidity) | 成功 RL (ReLease)<br/>失败 RF (RFund) |

## API

版本 V2

EndPoint : https://app.exinswap.com/api/v2

### 获取交易对数据
**GET /pairs**

请求参数： 无

响应：
```json
{
    "code": 0,
    "success": true,
    "message": "",
    "data": [
        {
            "asset0Uuid": "31d2ea9c-95eb-3355-b65b-ba096853bc18",
            "asset1Uuid": "54c61a72-b982-4034-a556-0d99e3c21e39",
            "lpAssetUuid": "f2b36161-4273-37be-8e4a-b73e43ea2383",
            "asset0Balance": "36.12596763",
            "asset1Balance": "0.99671476",
            "lpAssetSupply": "5.999999",
            "tradeType": "product",
            "curveAmplifier": "100.00000000",
            "createdAt": 1616038495,
            "updatedAt": 1616157061,
            "usdtTradeVolume24hours": "1.73962902",
            "priceRate24hours": "-2.09",
            "priceReverseRate24hours": "2.13"
        }
    ],
    "timestampMs": 1616157128977
}
```


### 获取资产列表
**GET /assets**

请求参数： 无

响应：
```json
{
    "code": 0,
    "success": true,
    "message": "",
    "data": [
        {
            "uuid": "4d8c508b-91c5-375b-92b0-ee702ed2dac5",
            "symbol": "USDT",
            "name": "Tether USD",
            "iconUrl": "https://mixin-images.zeromesh.net/ndNBEpObYs7450U08oAOMnSEPzN66SL8Mh-f2pPWBDeWaKbXTPUIdrZph7yj8Z93Rl8uZ16m7Qjz-E-9JFKSsJ-F=s128",
            "routeId": "1",
            "priceUsdt": "1",
            "chainAsset": {
                "uuid": "43d61dcd-e413-450d-80b8-101d5e903357",
                "symbol": "ETH",
                "name": "Ether",
                "iconUrl": "https://mixin-images.zeromesh.net/zVDjOxNTQvVsA8h2B4ZVxuHoCF3DJszufYKWpd9duXUSbSapoZadC7_13cnWBqg0EmwmRcKGbJaUpA8wFfpgZA=s128",
                "routeId": "7"
            }
        },
    ],
    "timestampMs": 1616157128977
}
```