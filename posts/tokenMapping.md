  谈到token的映射，这是一个在实际当中比较容易遇到的问题，即在以太的ERC20中发放代币，然后待公链上线后，从ERC20的token中把已经发放的代币映射到主网上面。
  
这里一共谈论了两种方法，其实有第三种，具体我们后面在讲。

第一种是需要一个可以提供rpc服务访问的full node，然后attach到他的console或者用http or https方法从远端通过网络访问，执行第一个nodejs脚本即可以获取所有关于这个合约的holder地址。

优点：可信性高，所有交易记录和地址余额都是从链上直接获取。

缺点：速度相对较慢，我在4Core 8G的阿里云上大概1秒钟遍历一个块（实际速度视块的大小及网络带宽而定）

```
var Web3 = require("web3");
var fs = require('fs');
var web3 =new Web3();

var contractAddress = "0x0a25c807291e58716ab78752f8bb15eae8370e7d"; //合约地址
web3.setProvider(new Web3.providers.HttpProvider("http://xxx.xxx.xxx.xxx:xxx"));一个全节点的rpc端口 

var abi=[{"constant":true,"inputs":[],"name":"mintingFinished","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"name","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_spender","type":"address"},{"name":"_value","type":"uint256"}],"name":"approve","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"totalSupply","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_from","type":"address"},{"name":"_to","type":"address"},{"name":"_value","type":"uint256"}],"name":"transferFrom","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"plockFlag","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"decimals","outputs":[{"name":"","type":"uint8"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_to","type":"address"},{"name":"_amount","type":"uint256"}],"name":"mint","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_value","type":"uint256"}],"name":"burn","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_addr","type":"address"}],"name":"removeLock","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"transferEnabled","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_addr","type":"address"}],"name":"setExclude","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_spender","type":"address"},{"name":"_subtractedValue","type":"uint256"}],"name":"decreaseApproval","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[{"name":"_owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[],"name":"renounceOwnership","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[],"name":"withdrawEther","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[],"name":"finishMinting","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_addr","type":"address"}],"name":"addLock","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"owner","outputs":[{"name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"symbol","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_owners","type":"address[]"},{"name":"_values","type":"uint256[]"}],"name":"allocateTokens","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_to","type":"address"},{"name":"_value","type":"uint256"}],"name":"transfer","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_enable","type":"bool"}],"name":"enableLockFlag","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_spender","type":"address"},{"name":"_addedValue","type":"uint256"}],"name":"increaseApproval","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[{"name":"_owner","type":"address"},{"name":"_spender","type":"address"}],"name":"allowance","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_enable","type":"bool"}],"name":"enableTransfer","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_newOwner","type":"address"}],"name":"transferOwnership","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"payable":true,"stateMutability":"payable","type":"fallback"},{"anonymous":false,"inputs":[{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Mint","type":"event"},{"anonymous":false,"inputs":[],"name":"MintFinished","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"previousOwner","type":"address"}],"name":"OwnershipRenounced","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"previousOwner","type":"address"},{"indexed":true,"name":"newOwner","type":"address"}],"name":"OwnershipTransferred","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"burner","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Burn","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"from","type":"address"},{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Transfer","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"owner","type":"address"},{"indexed":true,"name":"spender","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Approval","type":"event"}] //合约的abi

function checkTx(transaction) {
    var toRow = new Array();
    var inputRow = new Array();
    toRow[0] = "to";
    inputRow[0] = "input";
    transaction = transaction.toString().replace(/("*)/g, "")
    var transactionTo = JSON.stringify(web3.eth.getTransaction(transaction), toRow)
    var transactionInput = JSON.stringify(web3.eth.getTransaction(transaction), inputRow)
        if (transactionTo.search(contractAddress) > 0) {
        transfer = transactionInput.substring(10, 43);
        account = transactionInput.substring(44, 84);
        account = "0x" + account;
        balance = metacoin.balanceOf.call(account);
        console.log("=======writing TX:" + transactionInput + "write account:" + account + "write balance:" + balance.toNumber())
        writeAccountAndBalance(account, balance)
    }else{
        console.log("SKIP TX:" + "transactionTo:" + transactionTo + "transactionInput:" + transactionInput.substring(0, 100))
    }
}

function writeAccountAndBalance(account, balance) {
    fs.writeFileSync(fileName, account + " " + Number(balance) + "\n", { 'flag': 'a' }, function(err) {
        if (err) {
           throw err;
        }
    });
}

function splitTX(transactionInfo) {
    if (transactionInfo.search ("0x") < 0 ) {
            console.log("there is no any transactoins\n");
            return 0;
    }
    tmpString = transactionInfo.toString().split('[')
    tmpString = tmpString[1].toString().split(']')
    trancationArray = tmpString[0].toString().split(',')

    return trancationArray
}

function run (){
   var currentBlock = 6038014; //此合约部署的起始区块
   
   var latestBlock = web3.eth.blockNumber;
   console.log("currentBlock:", currentBlock + "lastestBlock" + latestBlock);
   var displayRow = new Array();
   displayRow[0] = "transactions"

   for (var j = 0; currentBlock < latestBlock; j++) {
       console.log("current  block:" + currentBlock)
       if (transactionArray < 1) continue;
       transactionInfo = JSON.stringify(web3.eth.getBlock(currentBlock), displayRow)
       var transactionArray = splitTX(transactionInfo)
       if (transactionArray < 1) continue;
           console.log("\nIn the block:" + currentBlock + ", there are: " + transactionArray.length + " trancations")
           for (var k = 0; k < trancationArray.length; k++) {
               checkTx(transactionArray[k])
           }
       currentBlock = Number(currentBlock) + 1
   }
}

var fileName = process.argv[2];
var metacoin = web3.eth.contract(abi).at(contractAddress);
run();
```
第二种是利用https://etherscan.io提供的api，提取所有关于这个token的tx，然后根据tx的input参数得到每一笔转账的参数中的to的部分，也就是账户地址，然后运行web3接口跟据账户地址去调用合约里的balanceof 方法最后得到每个账户的余额

优点：速度快，4000多个tx大概8分钟左右就可以算出余额

缺点：tx来源于第三方，不是直接从链上读数据，可信性值得商榷。

下面就是两种方法的具体实现：
```
var URL = "http://api.etherscan.io/api?module=account&action=txlist&address=0x0a25c807291e58716ab78752f8bb15eae8370e7d&startblock=0&endblock=99999999&sort=asc&apikey=FXHHTPFXJPKAI1IZ35WQZSIJQG4KMIJZZ" //apikey是你在etherscan上的apikey，这个key需要自己去申请，如果发现访问异常，尝试翻墙
//wget "http://api.etherscan.io/api?module=account&action=txlist&address=0x0a25c807291e58716ab78752f8bb15eae8370e7d&startblock=0&endblock=99999999&sort=asc&apikey=FXHHTPFXJPKAIIZ35WQZSIJRXQG4KMIJZZ" -o tx.lst
var request = require("request");
var fs = require('fs');
var prefix = "\"input\"\:\"0xa9059cbb000000000000000000000000"; //合约里的交易方法
var lines = new Array();
var accounts = new Map();

request.get(URL, function(err, response, body){
                console.log(response.body);

var abi=[{"constant":true,"inputs":[],"name":"mintingFinished","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"name","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_spender","type":"address"},{"name":"_value","type":"uint256"}],"name":"approve","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"totalSupply","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_from","type":"address"},{"name":"_to","type":"address"},{"name":"_value","type":"uint256"}],"name":"transferFrom","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"plockFlag","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"decimals","outputs":[{"name":"","type":"uint8"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_to","type":"address"},{"name":"_amount","type":"uint256"}],"name":"mint","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_value","type":"uint256"}],"name":"burn","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_addr","type":"address"}],"name":"removeLock","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"transferEnabled","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_addr","type":"address"}],"name":"setExclude","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_spender","type":"address"},{"name":"_subtractedValue","type":"uint256"}],"name":"decreaseApproval","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[{"name":"_owner","type":"address"}],"name":"balanceOf","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[],"name":"renounceOwnership","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[],"name":"withdrawEther","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[],"name":"finishMinting","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_addr","type":"address"}],"name":"addLock","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"owner","outputs":[{"name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"symbol","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_owners","type":"address[]"},{"name":"_values","type":"uint256[]"}],"name":"allocateTokens","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_to","type":"address"},{"name":"_value","type":"uint256"}],"name":"transfer","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_enable","type":"bool"}],"name":"enableLockFlag","outputs":[{"name":"success","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_spender","type":"address"},{"name":"_addedValue","type":"uint256"}],"name":"increaseApproval","outputs":[{"name":"","type":"bool"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[{"name":"_owner","type":"address"},{"name":"_spender","type":"address"}],"name":"allowance","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_enable","type":"bool"}],"name":"enableTransfer","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"name":"_newOwner","type":"address"}],"name":"transferOwnership","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"inputs":[],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"payable":true,"stateMutability":"payable","type":"fallback"},{"anonymous":false,"inputs":[{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"amount","type":"uint256"}],"name":"Mint","type":"event"},{"anonymous":false,"inputs":[],"name":"MintFinished","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"previousOwner","type":"address"}],"name":"OwnershipRenounced","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"previousOwner","type":"address"},{"indexed":true,"name":"newOwner","type":"address"}],"name":"OwnershipTransferred","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"burner","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Burn","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"from","type":"address"},{"indexed":true,"name":"to","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Transfer","type":"event"},{"anonymous":false,"inputs":[{"indexed":true,"name":"owner","type":"address"},{"indexed":true,"name":"spender","type":"address"},{"indexed":false,"name":"value","type":"uint256"}],"name":"Approval","type":"event"}]
var Web3 = require("web3");
var web3 =new Web3();
var contractAddress = "0x0a25c807291e58716ab78752f8bb15eae8370e7d"; //合约地址
web3.setProvider(new Web3.providers.HttpProvider("http://xxx.xxx.xxx:xxx"));

var metacoin = web3.eth.contract(abi).at(contractAddress);

lines = response.body.split(",");

for(var n = 0; n < lines.length; n++) {

     if (lines[n].search(prefix) > -1){
            account = lines[n].substr(prefix.length, 40);
            accounts.set(account, "00");
         }
  }
  for (var key of accounts.keys()) {
     var balance = metacoin.balanceOf.call("0x" + key);
     console.log(key + ":" + balance.toNumber());
     fs.writeFileSync("account.lst", key + ":" + Number(balance) + "\n", { 'flag': 'a' }, function(err) {
        if (err) {
           throw err;
        }
     });

  }
});
```
