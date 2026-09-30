# Part 79 | ขั้นตอนที่ 1381-1400 จาก 1000+

# Blockchain & Web3 Integration

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
1. เข้าใจ Blockchain fundamentals
2. integrate Ethereum กับ Node.js
3. ใช้ Web3.js และ ethers.js
4. deploy Smart Contracts
5. build NFT marketplace
6. implement DeFi patterns

---

## ขั้นตอนที่ 1381: Blockchain Fundamentals

```
Blockchain Architecture:

Block 1               Block 2               Block 3
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Hash: 0x0001... │   │ Hash: 0x0002... │   │ Hash: 0x0003... │
│ Prev: 0x0000... │──►│ Prev: 0x0001... │──►│ Prev: 0x0002... │
│ Nonce: 12345    │   │ Nonce: 67890    │   │ Nonce: 11111    │
│ Timestamp       │   │ Timestamp       │   │ Timestamp       │
│ Transactions:   │   │ Transactions:   │   │ Transactions:   │
│   [Tx1, Tx2]   │   │   [Tx3, Tx4]   │   │   [Tx5, Tx6]   │
└─────────────────┘   └─────────────────┘   └─────────────────┘

Key Properties:
- Immutable: Cannot modify past blocks
- Distributed: Multiple nodes have copies
- Transparent: All transactions are visible
- Trustless: No central authority needed
```

---

## ขั้นตอนที่ 1382: Web3.js Setup

```bash
# Install Web3.js
npm install web3

# Or ethers.js (more modern API)
npm install ethers

# Hardhat for local development
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox

# Initialize Hardhat project
npx hardhat init

# Additional tools
npm install --save-dev @openzeppelin/contracts
```

```typescript
// src/blockchain/web3-config.ts

import { Web3 } from "web3";
import { ethers } from "ethers";

// Connect to Ethereum node
export function createWeb3Provider(): Web3 {
  const rpcUrl = process.env.ETHEREUM_RPC_URL ?? "https://mainnet.infura.io/v3/YOUR_KEY";
  return new Web3(rpcUrl);
}

// Create ethers.js provider (preferred for new projects)
export function createEthersProvider(): ethers.JsonRpcProvider {
  const rpcUrl = process.env.ETHEREUM_RPC_URL ?? "https://mainnet.infura.io/v3/YOUR_KEY";
  return new ethers.JsonRpcProvider(rpcUrl);
}

// Create wallet from private key
export function createWallet(privateKey: string): ethers.Wallet {
  const provider = createEthersProvider();
  return new ethers.Wallet(privateKey, provider);
}
```

---

## ขั้นตอนที่ 1383: Smart Contract Basics

```solidity
// contracts/Token.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    uint256 public constant MAX_SUPPLY = 1_000_000 * 10**18;  // 1M tokens
    
    mapping(address => bool) public whitelist;
    bool public whitelistEnabled = true;
    
    event AddedToWhitelist(address indexed account);
    event WhitelistDisabled();
    
    constructor(address initialOwner) 
        ERC20("MyToken", "MTK") 
        Ownable(initialOwner)
    {
        // Mint initial supply to deployer
        _mint(initialOwner, 100_000 * 10**18);
    }
    
    function mint(address to, uint256 amount) external onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
    
    function addToWhitelist(address account) external onlyOwner {
        whitelist[account] = true;
        emit AddedToWhitelist(account);
    }
    
    function disableWhitelist() external onlyOwner {
        whitelistEnabled = false;
        emit WhitelistDisabled();
    }
    
    function _update(
        address from,
        address to,
        uint256 amount
    ) internal override {
        if (whitelistEnabled && from != address(0) && to != address(0)) {
            require(whitelist[from] || whitelist[to], "Not in whitelist");
        }
        super._update(from, to, amount);
    }
}
```

---

## ขั้นตอนที่ 1384: Deploying Smart Contracts

```typescript
// scripts/deploy.ts

import { ethers } from "hardhat";

async function main() {
  const [deployer] = await ethers.getSigners();
  
  console.log(`Deploying contracts with account: ${deployer.address}`);
  console.log(`Account balance: ${ethers.formatEther(await deployer.provider.getBalance(deployer.address))} ETH`);

  // Deploy Token contract
  const Token = await ethers.getContractFactory("MyToken");
  const token = await Token.deploy(deployer.address);
  
  await token.waitForDeployment();
  
  const tokenAddress = await token.getAddress();
  console.log(`MyToken deployed to: ${tokenAddress}`);
  
  // Verify on Etherscan (mainnet/testnet only)
  if (process.env.ETHERSCAN_API_KEY) {
    await new Promise(r => setTimeout(r, 30000));  // Wait for block confirmations
    
    await (hre as any).run("verify:verify", {
      address: tokenAddress,
      constructorArguments: [deployer.address]
    });
  }
  
  // Save deployment info
  const fs = await import("fs");
  const deployments = {
    network: (await ethers.provider.getNetwork()).name,
    token: {
      address: tokenAddress,
      deployedAt: new Date().toISOString(),
      deployedBy: deployer.address
    }
  };
  
  fs.writeFileSync("deployments.json", JSON.stringify(deployments, null, 2));
}

main().catch(console.error);
```

---

## ขั้นตอนที่ 1385: Interacting with Contracts

```typescript
// src/blockchain/token.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { ethers } from "ethers";

const TOKEN_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function totalSupply() view returns (uint256)",
  "function balanceOf(address) view returns (uint256)",
  "function transfer(address to, uint256 amount) returns (bool)",
  "function approve(address spender, uint256 amount) returns (bool)",
  "function allowance(address owner, address spender) view returns (uint256)",
  "event Transfer(address indexed from, address indexed to, uint256 amount)"
];

@Injectable()
export class TokenService {
  private readonly logger = new Logger(TokenService.name);
  private provider: ethers.JsonRpcProvider;
  private contract: ethers.Contract;
  private signer: ethers.Wallet;

  constructor() {
    this.provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
    
    this.signer = new ethers.Wallet(
      process.env.PRIVATE_KEY ?? "",
      this.provider
    );
    
    this.contract = new ethers.Contract(
      process.env.TOKEN_CONTRACT_ADDRESS ?? "",
      TOKEN_ABI,
      this.signer
    );
  }

  async getTokenInfo() {
    const [name, symbol, totalSupply] = await Promise.all([
      this.contract.name(),
      this.contract.symbol(),
      this.contract.totalSupply()
    ]);
    
    return {
      name,
      symbol,
      totalSupply: ethers.formatUnits(totalSupply, 18)
    };
  }

  async getBalance(address: string): Promise<string> {
    const balance = await this.contract.balanceOf(address);
    return ethers.formatUnits(balance, 18);
  }

  async transfer(to: string, amount: string): Promise<string> {
    const amountWei = ethers.parseUnits(amount, 18);
    
    // Estimate gas
    const gasEstimate = await this.contract.transfer.estimateGas(to, amountWei);
    const gasPrice = (await this.provider.getFeeData()).gasPrice;
    
    this.logger.log(
      `Transferring ${amount} tokens to ${to}. ` +
      `Estimated gas: ${gasEstimate}, price: ${ethers.formatUnits(gasPrice ?? 0n, "gwei")} gwei`
    );
    
    const tx = await this.contract.transfer(to, amountWei, {
      gasLimit: gasEstimate * 120n / 100n  // 20% buffer
    });
    
    this.logger.log(`Transaction sent: ${tx.hash}`);
    
    // Wait for confirmation
    const receipt = await tx.wait(1);  // 1 confirmation
    
    this.logger.log(`Transaction confirmed in block ${receipt.blockNumber}`);
    
    return tx.hash;
  }

  async listenForTransfers(callback: (from: string, to: string, amount: string) => void) {
    this.contract.on("Transfer", (from, to, amount) => {
      callback(from, to, ethers.formatUnits(amount, 18));
    });
  }
}
```

---

## ขั้นตอนที่ 1386: NFT Smart Contract

```solidity
// contracts/NFT.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyNFT is ERC721URIStorage, Ownable {
    uint256 private _tokenIds;
    uint256 public mintPrice = 0.01 ether;
    uint256 public maxSupply = 10000;
    
    mapping(address => uint256) public mintCount;
    uint256 public maxPerWallet = 10;
    
    bool public publicMintEnabled = false;
    
    event NFTMinted(address indexed recipient, uint256 tokenId, string tokenURI);
    event PriceUpdated(uint256 newPrice);

    constructor(address initialOwner) 
        ERC721("MyNFT", "MNFT") 
        Ownable(initialOwner) 
    {}

    // Owner mint (free)
    function ownerMint(address to, string memory tokenURI) external onlyOwner returns (uint256) {
        return _mintNFT(to, tokenURI);
    }

    // Public mint (requires payment)
    function publicMint(string memory tokenURI) external payable returns (uint256) {
        require(publicMintEnabled, "Public mint not enabled");
        require(msg.value >= mintPrice, "Insufficient payment");
        require(_tokenIds < maxSupply, "Max supply reached");
        require(mintCount[msg.sender] < maxPerWallet, "Exceeds max per wallet");
        
        return _mintNFT(msg.sender, tokenURI);
    }

    function _mintNFT(address to, string memory tokenURI) private returns (uint256) {
        _tokenIds++;
        uint256 newTokenId = _tokenIds;
        
        _safeMint(to, newTokenId);
        _setTokenURI(newTokenId, tokenURI);
        mintCount[to]++;
        
        emit NFTMinted(to, newTokenId, tokenURI);
        return newTokenId;
    }

    function enablePublicMint() external onlyOwner {
        publicMintEnabled = true;
    }

    function setMintPrice(uint256 newPrice) external onlyOwner {
        mintPrice = newPrice;
        emit PriceUpdated(newPrice);
    }

    function withdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No balance");
        payable(owner()).transfer(balance);
    }
    
    function totalSupply() public view returns (uint256) {
        return _tokenIds;
    }
}
```

---

## ขั้นตอนที่ 1387: NFT Service

```typescript
// src/blockchain/nft.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { ethers } from "ethers";
import { create } from "ipfs-http-client";

const NFT_ABI = [
  "function ownerMint(address to, string tokenURI) returns (uint256)",
  "function publicMint(string tokenURI) payable returns (uint256)",
  "function tokenURI(uint256 tokenId) view returns (string)",
  "function ownerOf(uint256 tokenId) view returns (address)",
  "function balanceOf(address owner) view returns (uint256)",
  "function totalSupply() view returns (uint256)",
  "function mintPrice() view returns (uint256)",
  "event NFTMinted(address indexed recipient, uint256 tokenId, string tokenURI)"
];

@Injectable()
export class NFTService {
  private readonly logger = new Logger(NFTService.name);
  private contract: ethers.Contract;
  private signer: ethers.Wallet;
  private ipfsClient: any;

  constructor() {
    const provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
    
    this.signer = new ethers.Wallet(
      process.env.PRIVATE_KEY ?? "",
      provider
    );
    
    this.contract = new ethers.Contract(
      process.env.NFT_CONTRACT_ADDRESS ?? "",
      NFT_ABI,
      this.signer
    );

    // IPFS client for metadata storage
    this.ipfsClient = create({
      host: "ipfs.infura.io",
      port: 5001,
      protocol: "https",
      headers: {
        authorization: `Bearer ${process.env.IPFS_PROJECT_SECRET}`
      }
    });
  }

  async mintNFT(
    toAddress: string,
    name: string,
    description: string,
    imageBuffer: Buffer
  ): Promise<{ tokenId: number; txHash: string; metadataUrl: string }> {
    // Upload image to IPFS
    const imageResult = await this.ipfsClient.add(imageBuffer);
    const imageUrl = `ipfs://${imageResult.cid}`;

    // Create and upload metadata
    const metadata = {
      name,
      description,
      image: imageUrl,
      attributes: [],
      created_at: new Date().toISOString()
    };

    const metadataResult = await this.ipfsClient.add(JSON.stringify(metadata));
    const metadataUrl = `ipfs://${metadataResult.cid}`;

    // Mint NFT
    const tx = await this.contract.ownerMint(toAddress, metadataUrl);
    const receipt = await tx.wait(1);
    
    // Extract token ID from event logs
    const event = receipt.logs.find(
      (log: any) => log.eventName === "NFTMinted"
    );
    const tokenId = Number(event?.args?.[1] ?? 0);

    this.logger.log(`Minted NFT #${tokenId} to ${toAddress}, tx: ${tx.hash}`);

    return { tokenId, txHash: tx.hash, metadataUrl };
  }

  async getNFTMetadata(tokenId: number): Promise<any> {
    const tokenURI = await this.contract.tokenURI(tokenId);
    const owner = await this.contract.ownerOf(tokenId);
    
    // Fetch metadata from IPFS
    let metadata = null;
    if (tokenURI.startsWith("ipfs://")) {
      const hash = tokenURI.replace("ipfs://", "");
      const response = await fetch(`https://ipfs.io/ipfs/${hash}`);
      metadata = await response.json();
    }

    return {
      tokenId,
      owner,
      tokenURI,
      metadata
    };
  }

  async getCollectionStats() {
    const [totalSupply, mintPrice] = await Promise.all([
      this.contract.totalSupply(),
      this.contract.mintPrice()
    ]);

    return {
      totalMinted: Number(totalSupply),
      mintPrice: ethers.formatEther(mintPrice)
    };
  }
}
```

---

## ขั้นตอนที่ 1388: Marketplace Smart Contract

```solidity
// contracts/Marketplace.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract NFTMarketplace is ReentrancyGuard, Ownable {
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool active;
    }
    
    mapping(uint256 => Listing) public listings;
    uint256 private _listingId;
    
    uint256 public platformFeePercent = 250;  // 2.5% (in basis points)
    uint256 public constant MAX_FEE = 1000;   // 10% maximum
    
    event Listed(
        uint256 indexed listingId,
        address indexed seller,
        address nftContract,
        uint256 tokenId,
        uint256 price
    );
    
    event Sold(
        uint256 indexed listingId,
        address indexed buyer,
        address indexed seller,
        uint256 price
    );
    
    event Cancelled(uint256 indexed listingId);

    constructor(address initialOwner) Ownable(initialOwner) {}

    function list(
        address nftContract,
        uint256 tokenId,
        uint256 price
    ) external returns (uint256) {
        require(price > 0, "Price must be > 0");
        
        IERC721 nft = IERC721(nftContract);
        require(nft.ownerOf(tokenId) == msg.sender, "Not NFT owner");
        require(
            nft.isApprovedForAll(msg.sender, address(this)) ||
            nft.getApproved(tokenId) == address(this),
            "Marketplace not approved"
        );
        
        _listingId++;
        listings[_listingId] = Listing({
            seller: msg.sender,
            nftContract: nftContract,
            tokenId: tokenId,
            price: price,
            active: true
        });
        
        emit Listed(_listingId, msg.sender, nftContract, tokenId, price);
        return _listingId;
    }

    function buy(uint256 listingId) external payable nonReentrant {
        Listing storage listing = listings[listingId];
        
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");
        
        listing.active = false;
        
        // Calculate fees
        uint256 fee = (listing.price * platformFeePercent) / 10000;
        uint256 sellerAmount = listing.price - fee;
        
        // Transfer NFT to buyer
        IERC721(listing.nftContract).safeTransferFrom(
            listing.seller,
            msg.sender,
            listing.tokenId
        );
        
        // Pay seller
        payable(listing.seller).transfer(sellerAmount);
        
        // Refund excess payment
        if (msg.value > listing.price) {
            payable(msg.sender).transfer(msg.value - listing.price);
        }
        
        emit Sold(listingId, msg.sender, listing.seller, listing.price);
    }

    function cancel(uint256 listingId) external {
        Listing storage listing = listings[listingId];
        
        require(listing.seller == msg.sender, "Not seller");
        require(listing.active, "Not active");
        
        listing.active = false;
        emit Cancelled(listingId);
    }

    function withdrawFees() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
}
```

---

## ขั้นตอนที่ 1389: Marketplace Service

```typescript
// src/blockchain/marketplace.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { ethers } from "ethers";

const MARKETPLACE_ABI = [
  "function list(address nftContract, uint256 tokenId, uint256 price) returns (uint256)",
  "function buy(uint256 listingId) payable",
  "function cancel(uint256 listingId)",
  "function listings(uint256) view returns (address seller, address nftContract, uint256 tokenId, uint256 price, bool active)",
  "event Listed(uint256 indexed listingId, address indexed seller, address nftContract, uint256 tokenId, uint256 price)",
  "event Sold(uint256 indexed listingId, address indexed buyer, address indexed seller, uint256 price)"
];

@Injectable()
export class MarketplaceService {
  private readonly logger = new Logger(MarketplaceService.name);
  private contract: ethers.Contract;
  private provider: ethers.JsonRpcProvider;
  private signer: ethers.Wallet;

  constructor() {
    this.provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
    
    this.signer = new ethers.Wallet(
      process.env.PRIVATE_KEY ?? "",
      this.provider
    );
    
    this.contract = new ethers.Contract(
      process.env.MARKETPLACE_ADDRESS ?? "",
      MARKETPLACE_ABI,
      this.signer
    );
  }

  async listNFT(
    nftContractAddress: string,
    tokenId: number,
    priceEth: string
  ): Promise<{ listingId: number; txHash: string }> {
    const priceWei = ethers.parseEther(priceEth);
    
    // First approve marketplace to transfer NFT
    const nftAbi = [
      "function approve(address to, uint256 tokenId)",
      "function setApprovalForAll(address operator, bool approved)"
    ];
    
    const nftContract = new ethers.Contract(nftContractAddress, nftAbi, this.signer);
    const approveTx = await nftContract.setApprovalForAll(
      await this.contract.getAddress(),
      true
    );
    await approveTx.wait(1);
    
    // List the NFT
    const tx = await this.contract.list(nftContractAddress, tokenId, priceWei);
    const receipt = await tx.wait(1);
    
    const event = receipt.logs.find(
      (log: any) => log.eventName === "Listed"
    );
    const listingId = Number(event?.args?.[0] ?? 0);
    
    return { listingId, txHash: tx.hash };
  }

  async buyNFT(listingId: number): Promise<{ txHash: string }> {
    const listing = await this.contract.listings(listingId);
    
    if (!listing.active) {
      throw new Error("Listing is not active");
    }
    
    const tx = await this.contract.buy(listingId, {
      value: listing.price
    });
    
    await tx.wait(1);
    
    return { txHash: tx.hash };
  }

  async getListing(listingId: number) {
    const listing = await this.contract.listings(listingId);
    
    return {
      listingId,
      seller: listing.seller,
      nftContract: listing.nftContract,
      tokenId: Number(listing.tokenId),
      price: ethers.formatEther(listing.price),
      active: listing.active
    };
  }

  async subscribeToEvents(
    onSale: (event: any) => void,
    onBuy: (event: any) => void
  ) {
    this.contract.on("Listed", (listingId, seller, nftContract, tokenId, price) => {
      onSale({ listingId: Number(listingId), seller, price: ethers.formatEther(price) });
    });

    this.contract.on("Sold", (listingId, buyer, seller, price) => {
      onBuy({ listingId: Number(listingId), buyer, seller, price: ethers.formatEther(price) });
    });
  }
}
```

---

## ขั้นตอนที่ 1390: Wallet Management

```typescript
// src/blockchain/wallet.service.ts

import { Injectable, Logger } from "@nestjs/common";
import { ethers } from "ethers";

@Injectable()
export class WalletService {
  private readonly logger = new Logger(WalletService.name);

  // Create new wallet
  createWallet(): { address: string; privateKey: string; mnemonic: string } {
    const wallet = ethers.Wallet.createRandom();
    
    return {
      address: wallet.address,
      privateKey: wallet.privateKey,
      mnemonic: wallet.mnemonic?.phrase ?? ""
    };
  }

  // Restore wallet from mnemonic
  restoreWalletFromMnemonic(mnemonic: string): ethers.HDNodeWallet {
    return ethers.Wallet.fromPhrase(mnemonic);
  }

  // Get wallet balance
  async getBalance(address: string): Promise<{
    eth: string;
    wei: bigint;
  }> {
    const provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
    
    const balance = await provider.getBalance(address);
    
    return {
      eth: ethers.formatEther(balance),
      wei: balance
    };
  }

  // Sign a message
  async signMessage(
    privateKey: string,
    message: string
  ): Promise<{ signature: string; address: string }> {
    const wallet = new ethers.Wallet(privateKey);
    const signature = await wallet.signMessage(message);
    
    return {
      signature,
      address: wallet.address
    };
  }

  // Verify a signature
  verifySignature(message: string, signature: string): string {
    // Returns the signer's address
    return ethers.verifyMessage(message, signature);
  }

  // Generate HD wallet (multiple accounts from one mnemonic)
  generateHDWallet(mnemonic: string, count: number = 10): Array<{
    path: string;
    address: string;
    privateKey: string;
  }> {
    const hdNode = ethers.HDNodeWallet.fromPhrase(mnemonic);
    const accounts = [];

    for (let i = 0; i < count; i++) {
      const child = hdNode.derivePath(`m/44'/60'/0'/0/${i}`);
      accounts.push({
        path: `m/44'/60'/0'/0/${i}`,
        address: child.address,
        privateKey: child.privateKey
      });
    }

    return accounts;
  }

  // Estimate gas for transaction
  async estimateGas(
    from: string,
    to: string,
    value: string
  ): Promise<{ gas: string; price: string; total: string }> {
    const provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
    
    const gasLimit = await provider.estimateGas({
      from,
      to,
      value: ethers.parseEther(value)
    });
    
    const feeData = await provider.getFeeData();
    const gasPrice = feeData.gasPrice ?? 0n;
    const totalFee = gasLimit * gasPrice;

    return {
      gas: gasLimit.toString(),
      price: ethers.formatUnits(gasPrice, "gwei") + " gwei",
      total: ethers.formatEther(totalFee) + " ETH"
    };
  }
}
```

---

## ขั้นตอนที่ 1391: Event Indexing

```typescript
// src/blockchain/event-indexer.ts
// Index blockchain events to database for fast queries

import { Injectable, Logger, OnModuleInit } from "@nestjs/common";
import { ethers } from "ethers";
import { DataSource } from "typeorm";

@Injectable()
export class BlockchainEventIndexer implements OnModuleInit {
  private readonly logger = new Logger(BlockchainEventIndexer.name);
  private provider: ethers.JsonRpcProvider;

  constructor(private readonly dataSource: DataSource) {
    this.provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
  }

  async onModuleInit() {
    await this.startIndexing();
  }

  async startIndexing() {
    // Get last indexed block
    const result = await this.dataSource.query(
      "SELECT MAX(block_number) as last_block FROM blockchain_events"
    );
    const lastBlock = result[0]?.last_block ?? 0;
    const currentBlock = await this.provider.getBlockNumber();

    this.logger.log(`Indexing from block ${lastBlock} to ${currentBlock}`);

    // Index historical events
    await this.indexFromBlock(lastBlock + 1, currentBlock);

    // Subscribe to new events
    this.provider.on("block", async (blockNumber) => {
      await this.indexBlock(blockNumber);
    });
  }

  private async indexFromBlock(fromBlock: number, toBlock: number) {
    const batchSize = 1000;
    
    for (let start = fromBlock; start <= toBlock; start += batchSize) {
      const end = Math.min(start + batchSize - 1, toBlock);
      await this.indexBlockRange(start, end);
    }
  }

  private async indexBlockRange(from: number, to: number) {
    const nftAddress = process.env.NFT_CONTRACT_ADDRESS ?? "";
    
    const filter: ethers.Filter = {
      address: nftAddress,
      topics: [
        ethers.id("Transfer(address,address,uint256)")
      ],
      fromBlock: from,
      toBlock: to
    };

    const logs = await this.provider.getLogs(filter);
    
    for (const log of logs) {
      await this.saveEvent(log);
    }
    
    this.logger.debug(`Indexed ${logs.length} events from blocks ${from}-${to}`);
  }

  private async indexBlock(blockNumber: number) {
    await this.indexBlockRange(blockNumber, blockNumber);
  }

  private async saveEvent(log: ethers.Log) {
    const iface = new ethers.Interface([
      "event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)"
    ]);
    
    try {
      const parsed = iface.parseLog(log);
      
      await this.dataSource.query(`
        INSERT INTO blockchain_events (
          block_number, tx_hash, contract_address, event_name,
          from_address, to_address, token_id, created_at
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
        ON CONFLICT (tx_hash, log_index) DO NOTHING
      `, [
        log.blockNumber,
        log.transactionHash,
        log.address.toLowerCase(),
        "Transfer",
        parsed?.args[0].toLowerCase(),
        parsed?.args[1].toLowerCase(),
        parsed?.args[2].toString()
      ]);
    } catch (error) {
      this.logger.error(`Error saving event: ${(error as Error).message}`);
    }
  }
}
```

---

## ขั้นตอนที่ 1392: Hardhat Testing

```typescript
// test/Token.test.ts

import { expect } from "chai";
import { ethers } from "hardhat";
import { MyToken } from "../typechain-types";

describe("MyToken", function () {
  let token: MyToken;
  let owner: any;
  let addr1: any;
  let addr2: any;

  beforeEach(async function () {
    [owner, addr1, addr2] = await ethers.getSigners();
    
    const Token = await ethers.getContractFactory("MyToken");
    token = await Token.deploy(owner.address);
    await token.waitForDeployment();
  });

  it("Should have correct name and symbol", async function () {
    expect(await token.name()).to.equal("MyToken");
    expect(await token.symbol()).to.equal("MTK");
  });

  it("Should mint initial supply to deployer", async function () {
    const balance = await token.balanceOf(owner.address);
    expect(balance).to.equal(ethers.parseUnits("100000", 18));
  });

  it("Should transfer tokens", async function () {
    // Add to whitelist first
    await token.addToWhitelist(addr1.address);
    
    const transferAmount = ethers.parseUnits("1000", 18);
    await token.transfer(addr1.address, transferAmount);
    
    expect(await token.balanceOf(addr1.address)).to.equal(transferAmount);
  });

  it("Should revert if not in whitelist", async function () {
    await expect(
      token.transfer(addr1.address, ethers.parseUnits("100", 18))
    ).to.be.revertedWith("Not in whitelist");
  });

  it("Should allow owner to mint", async function () {
    await token.addToWhitelist(addr1.address);
    await token.mint(addr1.address, ethers.parseUnits("5000", 18));
    
    expect(await token.balanceOf(addr1.address)).to.equal(
      ethers.parseUnits("5000", 18)
    );
  });
});
```

---

## ขั้นตอนที่ 1393: Gas Optimization

```solidity
// Gas optimization tips in Solidity

// ❌ Expensive: Reading from storage in loop
function badLoop(uint256[] storage values) internal view returns (uint256 sum) {
    for (uint256 i = 0; i < values.length; i++) {  // values.length is storage read
        sum += values[i];
    }
}

// ✅ Cheap: Cache storage reads
function goodLoop(uint256[] storage values) internal view returns (uint256 sum) {
    uint256 length = values.length;  // Cache in memory
    for (uint256 i = 0; i < length; i++) {
        sum += values[i];
    }
}

// ❌ Expensive: Using uint8 in struct (padding)
struct BadStruct {
    uint8 a;    // 1 byte
    uint256 b;  // 32 bytes (padded to new slot)
    uint8 c;    // 1 byte (padded to new slot)
}

// ✅ Cheap: Pack small values together
struct GoodStruct {
    uint8 a;   // 1 byte
    uint8 c;   // 1 byte (same slot as a)
    uint256 b; // 32 bytes (new slot)
}

// ❌ Expensive: String comparison
function badCompare(string memory a, string memory b) pure returns (bool) {
    return keccak256(bytes(a)) == keccak256(bytes(b));  // Actually OK
}

// ✅ Better: Use bytes32 for fixed-size strings
mapping(bytes32 => address) public roles;  // bytes32 key is cheaper than string
```

---

## ขั้นตอนที่ 1394: MetaMask Integration

```typescript
// src/web3/metamask.service.ts (Frontend/Browser)
// This code runs in the browser

export class MetaMaskService {
  
  async connect(): Promise<string> {
    if (!window.ethereum) {
      throw new Error("MetaMask not installed");
    }

    const accounts = await window.ethereum.request({
      method: "eth_requestAccounts"
    }) as string[];

    return accounts[0];
  }

  async signMessage(message: string): Promise<string> {
    const accounts = await window.ethereum.request({
      method: "eth_requestAccounts"
    }) as string[];

    const signature = await window.ethereum.request({
      method: "personal_sign",
      params: [message, accounts[0]]
    }) as string;

    return signature;
  }

  // Authentication flow: Sign a challenge message
  async authenticate(userId: string): Promise<{
    address: string;
    signature: string;
    message: string;
  }> {
    const address = await this.connect();
    
    // Get challenge from server
    const challenge = await fetch(`/api/auth/challenge/${address}`).then(r => r.json());
    
    // Sign the challenge
    const signature = await this.signMessage(challenge.message);
    
    return { address, signature, message: challenge.message };
  }
}
```

---

## ขั้นตอนที่ 1395: Blockchain Authentication

```typescript
// src/auth/blockchain-auth.service.ts
// Authenticate users with their Ethereum wallet

import { Injectable, Logger, UnauthorizedException } from "@nestjs/common";
import { ethers } from "ethers";
import { JwtService } from "@nestjs/jwt";
import Redis from "ioredis";
import { v4 as uuidv4 } from "uuid";

@Injectable()
export class BlockchainAuthService {
  private readonly logger = new Logger(BlockchainAuthService.name);

  constructor(
    private readonly jwtService: JwtService,
    private readonly redis: Redis
  ) {}

  async generateChallenge(address: string): Promise<{ message: string; expiresAt: number }> {
    const nonce = uuidv4().slice(0, 8);
    const expiresAt = Date.now() + 5 * 60 * 1000;  // 5 minutes
    
    const message = `Sign this message to authenticate with MyApp\n\nAddress: ${address}\nNonce: ${nonce}\nExpires: ${new Date(expiresAt).toISOString()}`;
    
    // Store challenge
    await this.redis.set(
      `challenge:${address.toLowerCase()}`,
      JSON.stringify({ message, nonce, expiresAt }),
      "EX",
      300  // 5 minutes
    );
    
    return { message, expiresAt };
  }

  async verifySignature(
    address: string,
    signature: string
  ): Promise<{ token: string; address: string }> {
    const challengeKey = `challenge:${address.toLowerCase()}`;
    const challengeData = await this.redis.get(challengeKey);
    
    if (!challengeData) {
      throw new UnauthorizedException("No pending challenge for this address");
    }

    const { message, expiresAt } = JSON.parse(challengeData);
    
    if (Date.now() > expiresAt) {
      await this.redis.del(challengeKey);
      throw new UnauthorizedException("Challenge expired");
    }

    // Verify the signature
    const recoveredAddress = ethers.verifyMessage(message, signature);
    
    if (recoveredAddress.toLowerCase() !== address.toLowerCase()) {
      throw new UnauthorizedException("Invalid signature");
    }

    // Delete used challenge
    await this.redis.del(challengeKey);

    // Issue JWT
    const token = this.jwtService.sign({
      sub: address.toLowerCase(),
      type: "blockchain-auth"
    });

    this.logger.log(`Authenticated wallet: ${address}`);

    return { token, address: address.toLowerCase() };
  }
}
```

---

## ขั้นตอนที่ 1396: DeFi Patterns

```typescript
// src/defi/price-oracle.ts
// Get token prices from Uniswap/Chainlink

import { Injectable, Logger } from "@nestjs/common";
import { ethers } from "ethers";

const CHAINLINK_ABI = [
  "function latestRoundData() view returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt, uint80 answeredInRound)",
  "function decimals() view returns (uint8)"
];

// Chainlink price feed addresses (Mainnet)
const PRICE_FEEDS: Record<string, string> = {
  "ETH/USD": "0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419",
  "BTC/USD": "0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c",
  "LINK/USD": "0x2c1d072e956AFFC0D435Cb7AC38EF18d24d9127c"
};

@Injectable()
export class PriceOracleService {
  private readonly logger = new Logger(PriceOracleService.name);
  private provider: ethers.JsonRpcProvider;

  constructor() {
    this.provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? ""
    );
  }

  async getPrice(pair: string): Promise<{ price: number; updatedAt: Date }> {
    const feedAddress = PRICE_FEEDS[pair];
    if (!feedAddress) throw new Error(`No price feed for ${pair}`);

    const feed = new ethers.Contract(feedAddress, CHAINLINK_ABI, this.provider);
    
    const [roundData, decimals] = await Promise.all([
      feed.latestRoundData(),
      feed.decimals()
    ]);

    const price = Number(roundData.answer) / 10 ** Number(decimals);
    const updatedAt = new Date(Number(roundData.updatedAt) * 1000);

    // Check if price is stale (>1 hour)
    if (Date.now() - updatedAt.getTime() > 3600000) {
      this.logger.warn(`Stale price feed for ${pair}: ${updatedAt.toISOString()}`);
    }

    return { price, updatedAt };
  }

  async getMultiplePrices(pairs: string[]): Promise<Record<string, number>> {
    const results = await Promise.all(
      pairs.map(pair => this.getPrice(pair).catch(() => null))
    );

    return pairs.reduce((acc, pair, i) => {
      if (results[i]) {
        acc[pair] = results[i]!.price;
      }
      return acc;
    }, {} as Record<string, number>);
  }
}
```

---

## ขั้นตอนที่ 1397: Blockchain API Controller

```typescript
// src/blockchain/blockchain.controller.ts

import { Controller, Get, Post, Body, Param, Logger } from "@nestjs/common";
import { TokenService } from "./token.service";
import { NFTService } from "./nft.service";
import { WalletService } from "./wallet.service";

@Controller("blockchain")
export class BlockchainController {
  private readonly logger = new Logger(BlockchainController.name);

  constructor(
    private readonly tokenService: TokenService,
    private readonly nftService: NFTService,
    private readonly walletService: WalletService
  ) {}

  @Get("token/info")
  async getTokenInfo() {
    return this.tokenService.getTokenInfo();
  }

  @Get("token/balance/:address")
  async getBalance(@Param("address") address: string) {
    return {
      address,
      balance: await this.tokenService.getBalance(address)
    };
  }

  @Post("wallet/create")
  createWallet() {
    const wallet = this.walletService.createWallet();
    
    // WARNING: In production, never return private key to client
    // Use a secure storage solution
    return {
      address: wallet.address,
      mnemonic: wallet.mnemonic  // Store this securely!
    };
  }

  @Get("wallet/balance/:address")
  async getWalletBalance(@Param("address") address: string) {
    return this.walletService.getBalance(address);
  }

  @Get("nft/:tokenId")
  async getNFT(@Param("tokenId") tokenId: string) {
    return this.nftService.getNFTMetadata(parseInt(tokenId));
  }

  @Get("nft/collection/stats")
  async getCollectionStats() {
    return this.nftService.getCollectionStats();
  }
}
```

---

## ขั้นตอนที่ 1398: Transaction Monitoring

```typescript
// src/blockchain/transaction-monitor.ts

import { Injectable, Logger } from "@nestjs/common";
import { ethers } from "ethers";

interface PendingTransaction {
  hash: string;
  from: string;
  to: string;
  value: string;
  submittedAt: Date;
  callbacks: {
    onConfirmed: (receipt: any) => void;
    onFailed: (error: Error) => void;
  };
}

@Injectable()
export class TransactionMonitor {
  private readonly logger = new Logger(TransactionMonitor.name);
  private pending = new Map<string, PendingTransaction>();
  private provider: ethers.JsonRpcProvider;

  constructor() {
    this.provider = new ethers.JsonRpcProvider(
      process.env.ETHEREUM_RPC_URL ?? "http://localhost:8545"
    );
    this.startMonitoring();
  }

  async watchTransaction(
    hash: string,
    onConfirmed: (receipt: any) => void,
    onFailed: (error: Error) => void,
    confirmations: number = 1
  ): Promise<void> {
    const tx = await this.provider.getTransaction(hash);
    if (!tx) throw new Error(`Transaction ${hash} not found`);

    this.pending.set(hash, {
      hash,
      from: tx.from,
      to: tx.to ?? "",
      value: ethers.formatEther(tx.value),
      submittedAt: new Date(),
      callbacks: { onConfirmed, onFailed }
    });

    this.logger.log(`Watching transaction: ${hash}`);
  }

  private startMonitoring() {
    this.provider.on("block", async (blockNumber) => {
      for (const [hash, pendingTx] of this.pending.entries()) {
        try {
          const receipt = await this.provider.getTransactionReceipt(hash);
          
          if (!receipt) continue;
          
          if (receipt.status === 0) {
            // Transaction failed
            this.pending.delete(hash);
            pendingTx.callbacks.onFailed(new Error(`Transaction ${hash} reverted`));
          } else if (blockNumber >= receipt.blockNumber) {
            // Transaction confirmed
            this.pending.delete(hash);
            this.logger.log(`Transaction ${hash} confirmed in block ${receipt.blockNumber}`);
            pendingTx.callbacks.onConfirmed(receipt);
          }
        } catch (error) {
          this.logger.error(`Error monitoring ${hash}: ${(error as Error).message}`);
        }
      }
    });
  }
}
```

---

## ขั้นตอนที่ 1399: Blockchain Module

```typescript
// src/blockchain/blockchain.module.ts

import { Module } from "@nestjs/common";
import { JwtModule } from "@nestjs/jwt";
import { TokenService } from "./token.service";
import { NFTService } from "./nft.service";
import { MarketplaceService } from "./marketplace.service";
import { WalletService } from "./wallet.service";
import { TransactionMonitor } from "./transaction-monitor";
import { BlockchainEventIndexer } from "./event-indexer";
import { BlockchainController } from "./blockchain.controller";
import { BlockchainAuthService } from "../auth/blockchain-auth.service";
import { PriceOracleService } from "../defi/price-oracle";

@Module({
  imports: [
    JwtModule.register({
      secret: process.env.JWT_SECRET,
      signOptions: { expiresIn: "7d" }
    })
  ],
  providers: [
    TokenService,
    NFTService,
    MarketplaceService,
    WalletService,
    TransactionMonitor,
    BlockchainEventIndexer,
    BlockchainAuthService,
    PriceOracleService
  ],
  controllers: [BlockchainController],
  exports: [
    TokenService,
    NFTService,
    WalletService,
    BlockchainAuthService,
    PriceOracleService
  ]
})
export class BlockchainModule {}
```

---

## ขั้นตอนที่ 1400: Security Considerations

```typescript
// Security best practices for blockchain apps

export const blockchainSecurityChecklist = {
  smartContracts: [
    "Use ReentrancyGuard for functions that transfer ETH",
    "Use OpenZeppelin battle-tested contracts",
    "Audit before mainnet deployment",
    "Use Slither or Mythril for static analysis",
    "Implement emergency pause mechanism",
    "Timelocks for critical operations",
    "Multi-sig for admin functions"
  ],
  
  privateKeys: [
    "NEVER store private keys in code or env files in production",
    "Use AWS KMS or HashiCorp Vault for key management",
    "Use Hardware Security Modules (HSM) for critical keys",
    "Implement key rotation policies",
    "Use multi-sig wallets for contract admin"
  ],
  
  apis: [
    "Rate limit blockchain API calls",
    "Validate all addresses (checksum format)",
    "Never trust client-provided blockchain data",
    "Verify transaction receipts server-side",
    "Monitor for suspicious transaction patterns"
  ],
  
  dataValidation: [
    "Validate Ethereum addresses: ethers.isAddress(address)",
    "Validate transaction hashes: /^0x[0-9a-f]{64}$/i",
    "Validate amounts: non-negative, within bounds",
    "Verify message signatures on server-side",
    "Check token contract is legitimate (not fake)"
  ]
};
```

---

## 🏋️ แบบฝึกหัด

### แบบฝึกหัดที่ 1: ERC20 Token
1. Deploy ERC20 token ด้วย Hardhat
2. ทดสอบ transfer functions
3. Interact จาก Node.js

### แบบฝึกหัดที่ 2: NFT Minting
1. Deploy ERC721 contract
2. Upload metadata to IPFS
3. Mint NFT จาก backend

### แบบฝึกหัดที่ 3: Wallet Auth
1. implement challenge-response auth
2. Verify MetaMask signature
3. Issue JWT token

---

## 📚 สรุป

ใน Part นี้เราได้เรียนรู้:
- Blockchain fundamentals
- Web3.js และ ethers.js setup
- Smart Contract deployment
- ERC20 Token contract
- ERC721 NFT contract
- NFT Marketplace
- Wallet management
- Event indexing
- DeFi price oracles
- Blockchain authentication
- Transaction monitoring

**Part ถัดไป**: IoT Integration
