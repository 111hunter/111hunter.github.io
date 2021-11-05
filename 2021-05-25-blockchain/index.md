# 单机版区块链


区块链是一个分布式数据库，任何人都可以读取的它的区块数据。区块链是不可变的，意味着一旦将区块添加到链中，就只能在使链的其余部分无效的情况下才能对其进行更改，这就是加密货币基于区块链来实现的原因。

## 区块和链

### 区块

区块链允许我们检测何时有人操纵了任何先前的区块，如何确保完整性呢？每个区块都根据前一个区块的哈希值，结合区块内的其他内容计算出自己的哈希值，作为该区块的唯一标识。

```js
const SHA256 = require("crypto-js/sha256");
class Block {
	constructor(timestamp, data, previousHash = '') {
		this.timestamp = timestamp;
		this.data = data;
		this.previousHash = previousHash;
		this.hash = this.calculateHash();
	}

	calculateHash() {
		return SHA256(this.timestamp + JSON.stringify(this.data) + this.previousHash).toString();
	}
}
```

### 链

但第一个区块是特殊的，它没有前一个区块，我们称它为创世区块。

```js
class Blockchain {
	constructor() {
		this.chain = [this.createGenesisBlock()];
	}

	createGenesisBlock() {
		return new Block("01/01/2021", "Genesis block", "0");
	}

	getLatestBlock() {
		return this.chain[this.chain.length - 1];
	}

	addBlock(newBlock) {
		newBlock.previousHash = this.getLatestBlock().hash;
		newBlock.hash = newBlock.calculateHash();
		this.chain.push(newBlock);
	}

	isChainValid() {
		for (let i = 1; i < this.chain.length; i++){
			const currentBlock = this.chain[i];
			const previousBlock = this.chain[i - 1];

			if (currentBlock.hash !== currentBlock.calculateHash()) {
				return false;
			}

			if (currentBlock.previousHash !== previousBlock.hash) {
				return false;
			}
		}
		return true;
	}
}
```

`isChainValid` 方法用于验证区块链是否被篡改。若有人想要篡改区块链上的某个区块，他必须要更改这个区块之后的所有区块，才能确保区块链仍是完整的。

## 工作量证明

现在，我们来总结上面的区块链中的问题：

- 1. 添加区块非常容易，攻击者可大量添加垃圾区块

- 2. 篡改区块并不会耗费多少时间

- 3. 结合上面两条可造出最长链

- 4. ...

如果区块的添加需要付出大量的算力成本，就可以在动机层面就杜绝攻击，区块链中这种机制叫工作量证明。简单来说，就是只添加哈希值满足特定条件的区块。

如果你想添加一个区块，首先需要付出算力，让你的区块的哈希值满足特定条件：

```js
class Block {
	constructor(timestamp, data, previousHash = '') {
		this.timestamp = timestamp;
		this.data = data;
		this.previousHash = previousHash;
		this.hash = this.calculateHash();
		this.nonce = 0;
	}

	calculateHash() {
		return SHA256(this.timestamp + JSON.stringify(this.data) + this.previousHash + this.nonce).toString();
	}

	mineBlock(difficulty) {
		while (this.hash.substring(0, difficulty) !== Array(difficulty + 1).join("0")) {
			this.nonce++;
			this.hash = this.calculateHash();
		}
		console.log("BLOCK MINED: " + this.hash);
	}
}
```

`mineBlock` 可筛选出的区块是哈希值的前 `difficulty` 位均为 0 的区块。通过调整 `difficulty` 的值，可以控制添加新区块的时间间隔：

```js
constructor() {
	this.chain = [this.createGenesisBlock()];
	this.difficulty = 2;
}
```

```js
addBlock(newBlock) {
	newBlock.previousHash = this.getLatestBlock().hash;
	newBlock.mineBlock(this.difficulty);
	this.chain.push(newBlock);
}
```

在比特币中，大约每10分钟才会添加一个新的区块，而这个计算有效哈希的过程被称为挖矿。

## 交易和矿工奖励

当区块链中储存的信息为转账信息时，区块链就成了分布式账本。

```js
class Block {
    constructor(timestamp, transactions, previousHash = '') {
        this.timestamp = timestamp;
        this.transactions = transactions;
        this.previousHash = previousHash;
        this.hash = this.calculateHash();
        this.nonce = 0;
    }

    calculateHash() {
        return SHA256(this.previousHash + this.timestamp + JSON.stringify(this.transactions) + this.nonce).toString();    
    }
}

class Transaction {
    constructor(fromAddress, toAddress, amount){
        this.fromAddress = fromAddress;
        this.toAddress = toAddress;
        this.amount = amount;
    }
}
```

我们的区块链中只能在一个区块中存储 1 个交易，并且矿工没有任何奖励，我们来解决这个问题。

## 数字签名

