# ecoinWallet为什么使用React-Native开发
首先,这不是一篇严肃的技术讨论,只说明了我们团队为何选择React-Native.  
目前市场上Native的开发已经非常成熟,开发一款类似ecoinWallet的应用并不复杂.但是会写Native并且懂得区块链知识的开发人员稀缺.我们将围绕APP开发和区块链开发两部分来论述.

### APP开发
React-Native的优缺点:
- 优点:
  - React-Native使用js发开, 门槛低
  - React-Native调试非常快速(增量编译), 开发体验更好
  - 对发布的应用支持code-push热更新(苹果/谷歌都支持code-push,具体规则看官方说明)
  - React-Native跨平台(一份代码同时在android/ios上运行)
  - React-Native的UI是native的(js代码会通过JsBridge转成native的代码), 界面流畅度和原生一致

- 缺点:
  - React-Native当前仍然是beta版
  - 一些增强功能,定制功能需要native开发人员特别编写React-Native扩展供js调用
  - 在一个APP中同时存在native和React-Native是及其艰巨的任务(可以看看Airbnb的[React Native at Airbnb](https://medium.com/airbnb-engineering/react-native-at-airbnb-f95aa460be1c))
  - native开发人员通常不认同js语言, 让他们重新学习React-Native不容易

### 区块链开发
- 使用React-Native开发ethereum library  
在区块链领域, NodeJS的生态非常完善. 而React-Native是运行在JavascriptCore上, 一部分代码近乎可以直接运行.同时也因为JavascriptCore缺少Node扩展如:Buffer/Crypto等, 需要使用native编写的第三方代码来扩展React-Native.好在这部分代码在社区都有实现.

- 使用native开发ethereum library  
native开发区块链library,需要同时在android/ios上保持两条分支独自开发(需要的人手比React-Native多了不少);同时native开发区块链能借鉴的项目目前并不多,要实现诸如abi encode/decode, RLP encode/decode, offline sign transaction并保证代码质量绝非易事.

### ecoinWallet的选择
ecoinWallet团队成员有丰富的Golang开发经验, 对[ethereum/go-ethereum](https://github.com/ethereum/go-ethereum)较熟悉.  
同时,团队在NodeJS上也有较多经验, 移植NodeJS代码到React-Native也还算顺利.  
最关键的是,团队成员不多,我们觉得对于一个从0到1的过程, 还是更应该追求开发效率.在今后React-Native不能满足我们的业务需求时,再使用native重写.
