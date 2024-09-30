# 

### **Step 1: Solidity Smart Contract for Parking Tokens**

This smart contract will allow users to purchase tokens that represent parking spots, and they can redeem these tokens for parking time.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ParkingToken {
    string public name = "ParkingToken";
    string public symbol = "PARK";
    uint256 public totalSupply;
    address public owner;

    mapping(address => uint256) public balanceOf;
    
    event TokenPurchased(address indexed buyer, uint256 amount);
    event TokenRedeemed(address indexed user, uint256 amount);

    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply;
        balanceOf[msg.sender] = _initialSupply;
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not contract owner");
        _;
    }

    function purchaseTokens() public payable {
        uint256 tokensToBuy = msg.value;
        require(balanceOf[owner] >= tokensToBuy, "Not enough tokens available");

        balanceOf[msg.sender] += tokensToBuy;
        balanceOf[owner] -= tokensToBuy;

        emit TokenPurchased(msg.sender, tokensToBuy);
    }

    function redeemTokens(uint256 _amount) public {
    
    
```

Let's implement the requested changes to your Solidity contract. Here's an overview of the tasks:

1. **Token Purchase (10 tokens per Ether)**: Modify the `purchaseTokens` function to allow users to buy 10 tokens per Ether.
2. **Redeem Tokens as NFT for Parking**: The `redeemTokens` function should allow the user to scan a parking spot and redeem hours of parking based on the value of the token. The user can also sell the NFT at a higher exchange rate for future parking.
3. **Parking Price Calculation**: Parking price should be determined based on the value of Ether at $5,000 per parking spot, divided by 13 passengers.
4. **Transfer Tokens at a Higher Exchange Rate**: Users should be able to transfer tokens at a higher exchange rate, driving up the price of parking or gifting in the form of an NFT.
5. **Contract Owner Ether Withdrawal**: Add the ability for the contract owner to withdraw the Ether that has been paid by users for tokens.

### Updated Solidity Smart Contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ParkingToken {
    string public name = "ParkingToken";
    string public symbol = "PARK";
    uint256 public totalSupply;
    uint256 public tokenPrice = 1 ether / 10; // 10 tokens per 1 Ether
    uint256 public parkingPriceInEther = 5000 ether / 13; // $5,000 per spot divided by 13 passengers

    address public owner;
    mapping(address => uint256) public balanceOf;

    // For tracking NFTs (Non-Fungible Value)
    struct ParkingSpotNFT {
        address owner;
        uint256 valueInTokens; // The value of the NFT in tokens
        bool isActive;
    }

    mapping(uint256 => ParkingSpotNFT) public parkingNFTs; // Mapping of NFT ID to ParkingSpotNFT details
    uint256 public nextNFTId = 1; // ID tracker for NFTs

    event TokenPurchased(address indexed buyer, uint256 amount);
    event TokenRedeemed(address indexed user, uint256 amount, uint256 parkingHours);
    event NFTCreated(address indexed owner, uint256 nftId, uint256 tokenValue);
    event NFTSold(address indexed seller, address indexed buyer, uint256 nftId, uint256 salePrice);
    event TokensTransferred(address indexed from, address indexed to, uint256 amount);
    event OwnerWithdrawal(uint256 amount);

    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply;
        balanceOf[msg.sender] = _initialSupply;
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not contract owner");
        _;
    }

    // Purchase tokens at 10 tokens per Ether
    function purchaseTokens() public payable {
        uint256 tokensToBuy = (msg.value / tokenPrice) * 10; // 10 tokens per Ether
        require(balanceOf[owner] >= tokensToBuy, "Not enough tokens available");

        balanceOf[msg.sender] += tokensToBuy;
        balanceOf[owner] -= tokensToBuy;

        emit TokenPurchased(msg.sender, tokensToBuy);
    }

    // Redeem tokens as an NFT to scan a parking spot or sell at a higher exchange rate
    function redeemTokensForNFT(uint256 _amount) public {
        require(balanceOf[msg.sender] >= _amount, "Not enough tokens to redeem");

        // Create a new NFT for the user
        ParkingSpotNFT memory newNFT = ParkingSpotNFT({
            owner: msg.sender,
            valueInTokens: _amount,
            isActive: true
        });

        parkingNFTs[nextNFTId] = newNFT;
        emit NFTCreated(msg.sender, nextNFTId, _amount);

        nextNFTId++; // Increment the NFT ID for the next creation

        // Deduct tokens from the user
        balanceOf[msg.sender] -= _amount;
    }

    // Calculate parking hours based on token redemption
    function scanParkingSpot(uint256 nftId) public {
        require(parkingNFTs[nftId].owner == msg.sender, "You do not own this NFT");
        require(parkingNFTs[nftId].isActive, "This NFT has already been used");

        // Calculate hours of parking based on the token value
        uint256 parkingHours = parkingNFTs[nftId].valueInTokens / (parkingPriceInEther / 1 ether);
        parkingNFTs[nftId].isActive = false; // Mark the NFT as redeemed

        emit TokenRedeemed(msg.sender, parkingNFTs[nftId].valueInTokens, parkingHours);
    }

    // Sell an NFT at a higher exchange rate
    function sellNFT(uint256 nftId, address to, uint256 salePrice) public {
        require(parkingNFTs[nftId].owner == msg.sender, "You do not own this NFT");

        // Transfer ownership of the NFT
        parkingNFTs[nftId].owner = to;

        // Transfer tokens based on sale price
        require(balanceOf[to] >= salePrice, "Buyer does not have enough tokens");
        balanceOf[to] -= salePrice;
        balanceOf[msg.sender] += salePrice;

        emit NFTSold(msg.sender, to, nftId, salePrice);
    }

    // Transfer tokens at a higher exchange rate (optional: for gifting or selling tokens)
    function transferTokens(address to, uint256 amount, uint256 exchangeRate) public {
        uint256 finalAmount = amount * exchangeRate; // Adjust amount based on exchange rate
        require(balanceOf[msg.sender] >= finalAmount, "Not enough tokens to transfer");

        balanceOf[msg.sender] -= finalAmount;
        balanceOf[to] += finalAmount;

        emit TokensTransferred(msg.sender, to, finalAmount);
    }

    // Contract owner can withdraw Ether
    function withdrawFunds() public onlyOwner {
        uint256 amount = address(this).balance;
        payable(owner).transfer(amount);

        emit OwnerWithdrawal(amount);
    }

    // Fallback function for receiving Ether and purchasing tokens
    receive() external payable {
        purchaseTokens(); // Automatically trigger token purchase if Ether is sent
    }
}
```

### **Key Changes and Features Explained:**

1. **Token Purchase (10 tokens per Ether)**:
   - The `purchaseTokens` function is modified to allow users to buy **10 tokens per Ether**.
   - Each Ether gets divided by `tokenPrice = 1 ether / 10;` to determine how many tokens a user gets.

2. **Redeem Tokens as NFT for Parking**:
   - The `redeemTokensForNFT` function allows users to redeem tokens to create an **NFT** for a parking spot.
   - **NFTs** are represented by the `ParkingSpotNFT` struct, which stores the owner, token value, and whether the NFT is active.
   - Users can then "scan" their parking spot by calling `scanParkingSpot`, which calculates **parking hours** based on token value and marks the NFT as redeemed.

3. **Parking Price Calculation**:
   - The **parking price** is determined by the value of Ether at $5,000 divided by 13 passengers: 
     ```solidity
     uint256 parkingPriceInEther = 5000 ether / 13;
     ```
   - This is used in the `scanParkingSpot` function to determine how many **hours of parking** the user gets for their redeemed tokens.

4. **Sell or Gift NFTs at a Higher Exchange Rate**:
   - The `sellNFT` function allows users to sell an NFT at a higher exchange rate (i.e., more tokens than it was originally worth) and transfer ownership.
   - The `transferTokens` function allows for **gifting** or transferring tokens at a higher exchange rate, which could drive up the price of parking or be used for rewards.

5. **Owner Withdrawal of Ether**:
   - The `withdrawFunds` function allows the **contract owner** to withdraw any Ether that users have paid to purchase tokens. The owner can transfer this Ether to their address.

---

This contract allows the sale of parking tokens, the creation of parking spot NFTs, the ability to redeem them for parking hours, sell NFTs at a higher exchange rate, and lets the contract owner withdraw Ether. This forms the basis of an interactive NFT-based parking solution!

## Updated Contract 

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ParkingToken {
    // Basic information about the token
    string public name = "ParkingToken"; // Name of the token
    string public symbol = "PARK";       // Symbol of the token
    uint256 public totalSupply;          // Total supply of tokens
    uint256 public tokenPrice = 1 ether; // Price for each token (you can modify this rate)

    address public owner;                // Owner of the contract

    mapping(address => uint256) public balanceOf; // Mapping to store balances of each user
    
    event TokenPurchased(address indexed buyer, uint256 amount); // Event for token purchase
    event TokenRedeemed(address indexed user, uint256 amount);    // Event for token redemption

    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply;          // Initialize total token supply
        balanceOf[msg.sender] = _initialSupply; // Give all initial tokens to contract owner
        owner = msg.sender;                     // Set contract deployer as owner
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not contract owner");
        _;
    }

    // Function to allow users to purchase tokens
    function purchaseTokens() public payable {
        uint256 tokensToBuy = msg.value / tokenPrice; // Adjust this for token pricing
        require(balanceOf[owner] >= tokensToBuy, "Not enough tokens available"); // Ensure owner has enough tokens

        balanceOf[msg.sender] += tokensToBuy; // Increase buyer's balance
        balanceOf[owner] -= tokensToBuy;      // Decrease owner's balance

        emit TokenPurchased(msg.sender, tokensToBuy); // Emit purchase event
    }

    // Function for users to redeem tokens (e.g., pay for parking)
    function redeemTokens(uint256 _amount) public {
        require(balanceOf[msg.sender] >= _amount, "Not enough tokens"); // Ensure user has enough tokens to redeem

        balanceOf[msg.sender] -= _amount;  // Deduct the redeemed tokens from user balance
        balanceOf[owner] += _amount;       // Return tokens to owner (optional)

        emit TokenRedeemed(msg.sender, _amount); // Emit redeem event
        // You can add logic here for parking time assignment, etc.
        // e.g., uint256 parkingTime = _amount * 1 hour;
    }

    // (Optional) Transfer function for users to transfer tokens to others
    function transfer(address _to, uint256 _amount) public {
        require(balanceOf[msg.sender] >= _amount, "Not enough tokens");
        balanceOf[msg.sender] -= _amount;
        balanceOf[_to] += _amount;
    }

    // (Optional) Function for the owner to withdraw Ether collected from token sales
    function withdrawFunds() public onlyOwner {
        payable(owner).transfer(address(this).balance); // Transfer contract balance to the owner
    }

    // Fallback function to handle any Ether sent directly to the contract
    receive() external payable {
        purchaseTokens(); // Automatically trigger token purchase if Ether is sent
    }
}

```

Key Points to Fill In or Modify:
Token Price (tokenPrice):

Right now, each token is priced at 1 Ether (uint256 public tokenPrice = 1 ether;). You can modify the tokenPrice to a different value if needed.
redeemTokens Logic:

The redeemTokens function is mostly complete but you need to define how users redeem tokens for parking. For example:
How many tokens equal an hour of parking?
What happens when they redeem tokens (maybe triggering parking rights)?

```
uint256 parkingTime = _amount * 1 hours;

```

-   **Fallback Function:**
    
    -   The `receive()` function allows the contract to automatically sell tokens when someone sends Ether directly to the contract. You don't need to change this unless you have specific requirements.
-   **Transfer Logic (Optional):**
    
    -   The `transfer()` function allows users to transfer tokens to other users. If your project doesn't need token transfers, you can remove this.
-   **Withdraw Funds (Optional):**
    
    -   The `withdrawFunds()` function allows the contract owner to withdraw Ether collected from token purchases. This is useful if you want to withdraw the funds for operational purposes.

### **Next Steps:**

-   **JavaScript (Web3.js or Ethers.js):** Once this contract is deployed, the front-end (HTML/CSS) will need JavaScript (using **Web3.js** or **Ethers.js**) to interact with the contract.
    
    -   Use **Web3.js** to call the contract functions (`purchaseTokens`, `redeemTokens`) from your frontend.
    -   You'll link the smart contract ABI and address to the frontend in JavaScript.
-   **Deployment:** You'll need to deploy this smart contract on a test network (e.g., Rinkeby) or the main Ethereum network using tools like **Remix** or **Truffle/Hardhat**.

Let's implement the requested changes to your Solidity contract. Here's an overview of the tasks:

1.  **Token Purchase (10 tokens per Ether)**: Modify the `purchaseTokens` function to allow users to buy 10 tokens per Ether.
2.  **Redeem Tokens as NFT for Parking**: The `redeemTokens` function should allow the user to scan a parking spot and redeem hours of parking based on the value of the token. The user can also sell the NFT at a higher exchange rate for future parking.
3.  **Parking Price Calculation**: Parking price should be determined based on the value of Ether at $5,000 per parking spot, divided by 13 passengers.
4.  **Transfer Tokens at a Higher Exchange Rate**: Users should be able to transfer tokens at a higher exchange rate, driving up the price of parking or gifting in the form of an NFT.
5.  **Contract Owner Ether Withdrawal**: Add the ability for the contract owner to withdraw the Ether that has been paid by users for tokens.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ParkingToken {
    string public name = "ParkingToken";
    string public symbol = "PARK";
    uint256 public totalSupply;
    uint256 public tokenPrice = 1 ether / 10; // 10 tokens per 1 Ether
    uint256 public parkingPriceInEther = 5000 ether / 13; // $5,000 per spot divided by 13 passengers

    address public owner;
    mapping(address => uint256) public balanceOf;

    // For tracking NFTs (Non-Fungible Value)
    struct ParkingSpotNFT {
        address owner;
        uint256 valueInTokens; // The value of the NFT in tokens
        bool isActive;
    }

    mapping(uint256 => ParkingSpotNFT) public parkingNFTs; // Mapping of NFT ID to ParkingSpotNFT details
    uint256 public nextNFTId = 1; // ID tracker for NFTs

    event TokenPurchased(address indexed buyer, uint256 amount);
    event TokenRedeemed(address indexed user, uint256 amount, uint256 parkingHours);
    event NFTCreated(address indexed owner, uint256 nftId, uint256 tokenValue);
    event NFTSold(address indexed seller, address indexed buyer, uint256 nftId, uint256 salePrice);
    event TokensTransferred(address indexed from, address indexed to, uint256 amount);
    event OwnerWithdrawal(uint256 amount);

    constructor(uint256 _initialSupply) {
        totalSupply = _initialSupply;
        balanceOf[msg.sender] = _initialSupply;
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Not contract owner");
        _;
    }

    // Purchase tokens at 10 tokens per Ether
    function purchaseTokens() public payable {
        uint256 tokensToBuy = (msg.value / tokenPrice) * 10; // 10 tokens per Ether
        require(balanceOf[owner] >= tokensToBuy, "Not enough tokens available");

        balanceOf[msg.sender] += tokensToBuy;
        balanceOf[owner] -= tokensToBuy;

        emit TokenPurchased(msg.sender, tokensToBuy);
    }

    // Redeem tokens as an NFT to scan a parking spot or sell at a higher exchange rate
    function redeemTokensForNFT(uint256 _amount) public {
        require(balanceOf[msg.sender] >= _amount, "Not enough tokens to redeem");

        // Create a new NFT for the user
        ParkingSpotNFT memory newNFT = ParkingSpotNFT({
            owner: msg.sender,
            valueInTokens: _amount,
            isActive: true
        });

        parkingNFTs[nextNFTId] = newNFT;
        emit NFTCreated(msg.sender, nextNFTId, _amount);

        nextNFTId++; // Increment the NFT ID for the next creation

        // Deduct tokens from the user
        balanceOf[msg.sender] -= _amount;
    }

    // Calculate parking hours based on token redemption
    function scanParkingSpot(uint256 nftId) public {
        require(parkingNFTs[nftId].owner == msg.sender, "You do not own this NFT");
        require(parkingNFTs[nftId].isActive, "This NFT has already been used");

        // Calculate hours of parking based on the token value
        uint256 parkingHours = parkingNFTs[nftId].valueInTokens / (parkingPriceInEther / 1 ether);
        parkingNFTs[nftId].isActive = false; // Mark the NFT as redeemed

        emit TokenRedeemed(msg.sender, parkingNFTs[nftId].valueInTokens, parkingHours);
    }

    // Sell an NFT at a higher exchange rate
    function sellNFT(uint256 nftId, address to, uint256 salePrice) public {
        require(parkingNFTs[nftId].owner == msg.sender, "You do not own this NFT");

        // Transfer ownership of the NFT
        parkingNFTs[nftId].owner = to;

        // Transfer tokens based on sale price
        require(balanceOf[to] >= salePrice, "Buyer does not have enough tokens");
        balanceOf[to] -= salePrice;
        balanceOf[msg.sender] += salePrice;

        emit NFTSold(msg.sender, to, nftId, salePrice);
    }

    // Transfer tokens at a higher exchange rate (optional: for gifting or selling tokens)
    function transferTokens(address to, uint256 amount, uint256 exchangeRate) public {
        uint256 finalAmount = amount * exchangeRate; // Adjust amount based on exchange rate
        require(balanceOf[msg.sender] >= finalAmount, "Not enough tokens to transfer");

        balanceOf[msg.sender] -= finalAmount;
        balanceOf[to] += finalAmount;

        emit TokensTransferred(msg.sender, to, finalAmount);
    }

    // Contract owner can withdraw Ether
    function withdrawFunds() public onlyOwner {
        uint256 amount = address(this).balance;
        payable(owner).transfer(amount);

        emit OwnerWithdrawal(amount);
    }

    // Fallback function for receiving Ether and purchasing tokens
    receive() external payable {
        purchaseTokens(); // Automatically trigger token purchase if Ether is sent
    }
}

```

### **Key Changes and Features Explained:**

1.  **Token Purchase (10 tokens per Ether)**:
    
    -   The `purchaseTokens` function is modified to allow users to buy **10 tokens per Ether**.
    -   Each Ether gets divided by `tokenPrice = 1 ether / 10;` to determine how many tokens a user gets.
2.  **Redeem Tokens as NFT for Parking**:
    
    -   The `redeemTokensForNFT` function allows users to redeem tokens to create an **NFT** for a parking spot.
    -   **NFTs** are represented by the `ParkingSpotNFT` struct, which stores the owner, token value, and whether the NFT is active.
    -   Users can then "scan" their parking spot by calling `scanParkingSpot`, which calculates **parking hours** based on token value and marks the NFT as redeemed.
3.  **Parking Price Calculation**:
    
    -   The **parking price** is determined by the value of Ether at $5,000 divided by 13 passengers:

```
uint256 parkingPriceInEther = 5000 ether / 13;

```

1.  -   This is used in the `scanParkingSpot` function to determine how many **hours of parking** the user gets for their redeemed tokens.
2.  **Sell or Gift NFTs at a Higher Exchange Rate**:
    
    -   The `sellNFT` function allows users to sell an NFT at a higher exchange rate (i.e., more tokens than it was originally worth) and transfer ownership.
    -   The `transferTokens` function allows for **gifting** or transferring tokens at a higher exchange rate, which could drive up the price of parking or be used for rewards.
3.  **Owner Withdrawal of Ether**:
    
    -   The `withdrawFunds` function allows the **contract owner** to withdraw any Ether that users have paid to purchase tokens. The owner can transfer this Ether to their address.

This contract allows the sale of parking tokens, the creation of parking spot NFTs, the ability to redeem them for parking hours, sell NFTs at a higher exchange rate, and lets the contract owner withdraw Ether. This forms the basis of an interactive NFT-based parking solution!

