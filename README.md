# Blockchain Application for Electronic Voting

<p align="center">
<img src="https://coinspectator-com.s3.eu-west-2.amazonaws.com/production/20075/59e4686ec27f2263fa0bcac6603cace7.png" width="350">
</p>

Repository for Final Year Project - Blockchain Application for Electronic Voting.

## Introduction

Governments, for the sake of innovation, transparency and citizen participation are interested in pilot voting projects based on blockchain technology. This innovative technology allows the decentralization of the government of operations, specifically the voting of citizens in an immutable, auditable, safe and reliable distributed registry.   

## Architecture
There are a few different aspects to developing blockchain applications with Ethereum:

1. Smart contract development - writing code that gets deployed to the blockchain with the Solidity programming language.  
2. Developing websites or clients that interact with the blockchain - writing code that reads and writes data from the blockchain with smart contracts.  
Web3.js enables you to fulfill the second responsibility: developing clients that interact with The Ethereum Blockchain. It is a collection of libraries that allow you to perform actions like send Ether from one account to another, read and write data from smart contracts and create smart contracts.  

If you have a web development background, you might have used jQuery to make Ajax calls to a web server. That's a good starting point for understanding the function of Web3.js. Instead of using a jQuery to read and write data from a web server, you can use Web3.js to read and write to The Ethereum Blockchain.  

Web3.js talks to The Ethereum Blockchain with JSON RPC, which stands for "Remote Procedure Call" protocol. Ethereum is a peer-to-peer network of nodes that stores a copy of all the data and code on the blockchain. Web3.js allows us to make requests to an individual Ethereum node with JSON RPC in order to read and write data to the network. It's kind of like using jQuery with a JSON API to read and write data with a web server.  


<p align="center">
<img src="https://cdn-images-1.medium.com/max/1600/1*T_YAqogYLteDZ_h6XSp0zg.png" width="350">
</p>

The project is structured as follows:  
* A private blockchain network in AWS consisting in:
    * One miner
    * Five nodes
    * One NAT gateway
    * One external node

<p align="center">
<img src="doc/images/network.png" width="550">
</p>

## Deployment

### Step 1: Launch two EC2 instances.
Use t2.medium (2 vCPU, 4 GB RAM) with default 8G SSD. Pick Ubuntu OS. Make sure that the two nodes are with the same security group, which allows TCP 30303 (or 30000-30999 as I may use more ports on this range). Port 30303 by default is for peering among nodes.

<p align="center">
<img src="https://blockgeeks.com/wp-content/uploads/2017/07/image6-1.png">
</p>

__Experience Sharing__   
I first tried t2.micro as it is the free-tier offer. However, mining was not successful (“DAG” loop without ether reward). I next tried t2.small and mining worked. However, when I deployed contract (see Part 2), the rpc was unstable. Finally, I found t2.medium good enough for my setup.  

I usually STOP the instance after testing (to save money). Please note that the public IP address of the EC2 instance will be changed after you START it back. This does not have an impact on our setup here as I am using the private IP address of these instances for peering. Private IP address remains even after I STOP/START the instance. In any case if Public IP address is needed, the peering still works, but you may need to change the peering address every time.  

### Step 2: Install geth client on Node 1
Access to Node 1 with ssh and proper key. Follow the recommended process (link) for geth installation. Install from PPA is good enough.

Node 1
```
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository -y ppa:ethereum/ethereum
$ sudo apt-get update
$ sudo apt-get install ethereum
``` 

and verify it with $ which geth and you will see that geth is installed properly.

### Step 3: Prepare the Genesis.json
This same Genesis.json is applied to both nodes, as it is to ensure both nodes having the same genesis block. Here is the Genesis.json I used, which is adopted from here (link). I take out the initial allocation as I don’t need to modify the address on this file in each new setup. Each account will gain some ethers once it starts mining.

```
Genesis.json
{

"config": {

"chainId": 15,

"homesteadBlock": 0,

"eip155Block": 0,

"eip158Block": 0

},

"nonce": "0x0000000000000042",

"mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",

"difficulty": "0x200",

"alloc": {},

"coinbase": "0x0000000000000000000000000000000000000000",

"timestamp": "0x00",

"parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",

"gasLimit": "0xffffffff",

"alloc": {

}

}
```
 
Keep this file somewhere and later scp to the nodes, or just copy and paste with an editor on both nodes.

### Step 4: Init the geth with Genesis.json
Node 1	
```
$ geth init Genesis.json
```

### Step 5: Launch geth now
It is good practice to have two screens in parallel (or a split terminal). One screen is the console while another shows the log. Open a new terminal and ssh to Node 1, and keep reading the log.

Node 1	
```
$ geth --nodiscover console 2>> eth.log
```
In another terminal
```
$ tail -F eth.log
```

<p align="center">
<img src="https://blockgeeks.com/wp-content/uploads/2017/07/image10-1.png">
</p>

Two-Node Setup of a Private Ethereum on AWS with Contract Deployment  

__Experience Sharing__  
In many examples, it is recommended to use –datadir to specify the directory for private ethereum blockchain. This is good practice when you are interacting with different chains. My example is an isolated environment. Therefore I omit this option, and my private chain is stored in the directory ~/.ethereum/.

### Step 6: Create an Account on Node 1
Node 1 geth
```
> personal.newAccount()

> eth.getBalance(eth.accounts[0])
```
Or
```
> web3.fromWei(eth.getBalance(eth.accounts[0]), “ether”)
```

Now we have an account on this node (always check it with `> eth.accounts[0]` or `> eth.coinbase`). And currently, there is no ether in the account balance as we have not allocated any in Genesis.json.  

### Step 7: Start Mining
We can begin mining process.

1. From the log terminal, we will see “Generating DAG in progress” and after an epoch, a block is being mined.  
2. Once a block is mined, 5 ethers are added to the account balance. This is a good indicator whether mining is successful or not.  
3. If we keep mining, the amount of account balance keeps increasing as more new blocks are mined.  

Node 1 geth	
```
> miner.start()
> eth.getBalance(eth.accounts[0])
```
Feel free to keep mining, or we can turn it off by `> miner.stop()`.
Now Node 1 is ready. Let’s work on Node 2.  

### Step 8: Repeat Step 2-6 on Node 2
Make sure the same Genesis.json is used when init the blockchain on Node 2.  
Do not start mining as we want Node 2 on the same blockchain as Node 1 (through Peering).  

### Step 9: Peering
Now we begin peering the two nodes. There are several ways to peer. Here I am using “admin addPeer” to do the peering: in Node 2, add Node 1 enode information for peering.

First, check both nodes that there is no peering.
```
Node 1	> admin.peers
Node 2	> admin.peers
```

Obtain enode information from Node 1
```
Node 1	> admin.nodeInfo.enode
```
We will get something like this:
```
"enode://c667fdf1f6846af74ed14070ef9ffeee33e98ff8ab0dd43f67415868974d8205e0fb7f55f6f37e9e1ebb112adfc0b88755714c7bc83a7ac47d30f8eb53118687@[::]:30303?discport=0"
```
Add this information to Node 2. Change the [::] with the Private Address of Node 1.  
Node 2
```
> admin.addPeer("enode://c667fdf1f6846af74ed14070ef9ffeee33e98ff8ab0dd43f67415868974d8205e0fb7f55f6f37e9e1ebb112adfc0b88755714c7bc83a7ac47d30f8eb53118687@172.31.62.34:30303?discport=0")
```
After that check, both nodes and they are peering each other.
```
Node 1	> admin.peers
Node 2	> admin.peers
```

The left-hand-side is Node 1 and right-hand-side is Node 2. Note that after `> addPeer()` on Node 2, two nodes are peered. We do not need to do the same thing on Node 1.


Also, from the log, we see that, after peering, we see “Block synchronization started” and “Imported new chain segment” on Node 2 log (bottom-right terminal).

 

__Experience Sharing__  
Make sure to use the same Genesis.json to initiate the blockchain. In one trial I forgot to init step and peering was unsuccessful.  
AWS EC2 instance comes with a private IP address and a public IP address. Both works fine in add peering, but using private IP address is more convenient as it is not changed after instance STOP/START.

### Step 10: Send Ethers Between Accounts
 
As both nodes are in the same Ethereum blockchain, we will send some ethers between the accounts. In this example, 10 ethers are sent from the account of Node 1 to account of Node 2. This is one of the best ways to verify if the setup is successful.  
Node 1
```
> web3.fromWei(eth.getBalance(eth.coinbase), “ether”)
> personal.unlockAccount(eth.coinbase)
> eth.sendTransaction({from: eth.coinbase, to: "0xabc65de992289401dcff3a70d4fcfe643f7d2271", value: web3.toWei(10, “ether”)})
> miner.start()
```
Node 2
```
> web3.fromWei(eth.getBalance(eth.coinbase), “ether”)
```

<p align="center">
<img src="https://blockgeeks.com/wp-content/uploads/2017/07/image4-1.png">
</p>

Note that we see “pending transaction” as this transaction is not mined yet. Once we start mining in Node 1, the transaction is done and account balance on Node 2 is now 10 ethers.  

And we see 45 ethers in Node 1 (not the expected 20-10=10 ethers). It is because Node 1 keeps receiving mining reward (5 ethers per block). The balance will keep increasing until we stop mining.  




## Technologies Used

* Ethereum (geth client)
* Solidity
* Web3js
* Nodejs
* Linux
* HTML5, CSS3 and JavaScript

## Author

* **Javier Mantilla**

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details

## Acknowledgments

* Daniel Cregg 