// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

/**
 * @title RPG Equipment NFT Game
 * @dev A play-to-earn RPG game where players can mint, battle, and trade equipment NFTs
 */
contract RPGGame is ERC721, Ownable, ReentrancyGuard {
    using Counters for Counters.Counter;
    
    Counters.Counter private _tokenIds;
    Counters.Counter private _playerIds;
    
    // Game Token for rewards
    IERC20 public gameToken;
    
    // Equipment types
    enum EquipmentType { SWORD, ARMOR, SHIELD, HELMET, BOOTS }
    enum Rarity { COMMON, RARE, EPIC, LEGENDARY }
    
    // Equipment stats structure
    struct Equipment {
        EquipmentType equipmentType;
        Rarity rarity;
        uint256 attackPower;
        uint256 defensePower;
        uint256 durability;
        uint256 level;
        bool isForSale;
        uint256 price;
    }
    
    // Player structure
    struct Player {
        uint256 playerId;
        uint256 level;
        uint256 experience;
        uint256 wins;
        uint256 losses;
        uint256[] equippedItems; // Array of token IDs
        bool isActive;
    }
    
    // Mappings
    mapping(uint256 => Equipment) public equipments;
    mapping(address => Player) public players;
    mapping(address => bool) public isRegistered;
    
    // Events
    event PlayerRegistered(address indexed player, uint256 playerId);
    event EquipmentMinted(address indexed player, uint256 tokenId, EquipmentType equipmentType, Rarity rarity);
    event BattleCompleted(address indexed winner, address indexed loser, uint256 rewardAmount);
    event EquipmentListed(uint256 indexed tokenId, uint256 price);
    event EquipmentSold(uint256 indexed tokenId, address indexed seller, address indexed buyer, uint256 price);
    
    // Constants
    uint256 public constant BATTLE_REWARD = 100 * 10**18; // 100 tokens
    uint256 public constant MINT_COST = 50 * 10**18; // 50 tokens to mint equipment
    
    constructor(address _gameToken) ERC721("RPG Equipment", "RPGEQ") Ownable(msg.sender) {
        gameToken = IERC20(_gameToken);
    }
    
    /**
     * @dev Core Function 1: Register Player and Mint Starting Equipment
     * @notice Registers a new player and mints their starting equipment
     */
    function registerPlayerAndMintStarterEquipment() external nonReentrant {
        require(!isRegistered[msg.sender], "Player already registered");
        
        _playerIds.increment();
        uint256 newPlayerId = _playerIds.current();
        
        // Register player
        players[msg.sender] = Player({
            playerId: newPlayerId,
            level: 1,
            experience: 0,
            wins: 0,
            losses: 0,
            equippedItems: new uint256[](0),
            isActive: true
        });
        
        isRegistered[msg.sender] = true;
        
        // Mint starter sword (free for new players)
        _mintEquipment(msg.sender, EquipmentType.SWORD, Rarity.COMMON);
        
        emit PlayerRegistered(msg.sender, newPlayerId);
    }
    
    /**
     * @dev Core Function 2: Battle System with Equipment and Rewards
     * @notice Allows two players to battle using their equipped items
     * @param opponent Address of the opponent player
     */
    function battle(address opponent) external nonReentrant {
        require(isRegistered[msg.sender] && isRegistered[opponent], "Both players must be registered");
        require(msg.sender != opponent, "Cannot battle yourself");
        require(players[msg.sender].isActive && players[opponent].isActive, "Both players must be active");
        
        Player storage attacker = players[msg.sender];
        Player storage defender = players[opponent];
        
        // Calculate battle power based on equipped items
        uint256 attackerPower = _calculatePlayerPower(msg.sender);
        uint256 defenderPower = _calculatePlayerPower(opponent);
        
        // Add some randomness (simplified for demo)
        uint256 randomFactor = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender, opponent))) % 20;
        attackerPower += randomFactor;
        
        address winner;
        address loser;
        
        if (attackerPower >= defenderPower) {
            winner = msg.sender;
            loser = opponent;
            attacker.wins++;
            defender.losses++;
        } else {
            winner = opponent;
            loser = msg.sender;
            defender.wins++;
            attacker.losses++;
        }
        
        // Award experience and level up check
        players[winner].experience += 50;
        players[loser].experience += 20; // Consolation experience
        
        _checkLevelUp(winner);
        _checkLevelUp(loser);
        
        // Reward winner with game tokens
        gameToken.transfer(winner, BATTLE_REWARD);
        
        emit BattleCompleted(winner, loser, BATTLE_REWARD);
    }
    
    /**
     * @dev Core Function 3: Equipment Marketplace
     * @notice Allows players to buy/sell equipment NFTs
     * @param tokenId The ID of the equipment to purchase
     */
    function buyEquipment(uint256 tokenId) external nonReentrant {
        require(_ownerOf(tokenId) != address(0), "Equipment does not exist");
        require(equipments[tokenId].isForSale, "Equipment not for sale");
        require(isRegistered[msg.sender], "Must be registered player");
        
        Equipment storage equipment = equipments[tokenId];
        address seller = ownerOf(tokenId);
        uint256 price = equipment.price;
        
        require(msg.sender != seller, "Cannot buy your own equipment");
        require(gameToken.balanceOf(msg.sender) >= price, "Insufficient balance");
        
        // Transfer payment
        gameToken.transferFrom(msg.sender, seller, price);
        
        // Transfer NFT
        _transfer(seller, msg.sender, tokenId);
        
        // Update equipment status
        equipment.isForSale = false;
        equipment.price = 0;
        
        emit EquipmentSold(tokenId, seller, msg.sender, price);
    }
    
    // Helper Functions
    
    function _mintEquipment(address to, EquipmentType equipmentType, Rarity rarity) internal {
        _tokenIds.increment();
        uint256 newTokenId = _tokenIds.current();
        
        // Generate random stats based on rarity
        (uint256 attackPower, uint256 defensePower) = _generateStats(rarity);
        
        equipments[newTokenId] = Equipment({
            equipmentType: equipmentType,
            rarity: rarity,
            attackPower: attackPower,
            defensePower: defensePower,
            durability: 100,
            level: 1,
            isForSale: false,
            price: 0
        });
        
        _safeMint(to, newTokenId);
        
        emit EquipmentMinted(to, newTokenId, equipmentType, rarity);
    }
    
    function _generateStats(Rarity rarity) internal view returns (uint256 attackPower, uint256 defensePower) {
        uint256 baseAttack = 10;
        uint256 baseDefense = 10;
        uint256 multiplier = uint256(rarity) + 1;
        
        uint256 randomBonus = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % 10;
        
        attackPower = baseAttack * multiplier + randomBonus;
        defensePower = baseDefense * multiplier + randomBonus;
    }
    
    function _calculatePlayerPower(address player) internal view returns (uint256) {
        uint256 totalPower = 0;
        uint256[] memory equippedItems = players[player].equippedItems;
        
        for (uint256 i = 0; i < equippedItems.length; i++) {
            Equipment memory equipment = equipments[equippedItems[i]];
            totalPower += equipment.attackPower + equipment.defensePower;
        }
        
        // Add base power and level bonus
        totalPower += 50 + (players[player].level * 10);
        
        return totalPower;
    }
    
    function _checkLevelUp(address player) internal {
        Player storage playerData = players[player];
        uint256 requiredExp = playerData.level * 100; // 100 exp per level
        
        if (playerData.experience >= requiredExp) {
            playerData.level++;
            playerData.experience -= requiredExp;
        }
    }
    
    // Public view functions
    
    function getPlayerStats(address player) external view returns (
        uint256 level,
        uint256 experience,
        uint256 wins,
        uint256 losses,
        uint256 totalPower
    ) {
        require(isRegistered[player], "Player not registered");
        Player memory playerData = players[player];
        
        return (
            playerData.level,
            playerData.experience,
            playerData.wins,
            playerData.losses,
            _calculatePlayerPower(player)
        );
    }
    
    function getEquipmentDetails(uint256 tokenId) external view returns (
        EquipmentType equipmentType,
        Rarity rarity,
        uint256 attackPower,
        uint256 defensePower,
        uint256 durability,
        uint256 level,
        bool isForSale,
        uint256 price
    ) {
        require(_ownerOf(tokenId) != address(0), "Equipment does not exist");
        Equipment memory equipment = equipments[tokenId];
        
        return (
            equipment.equipmentType,
            equipment.rarity,
            equipment.attackPower,
            equipment.defensePower,
            equipment.durability,
            equipment.level,
            equipment.isForSale,
            equipment.price
        );
    }
    
    // Owner functions
    
    function mintEquipmentForPlayer(address player, EquipmentType equipmentType, Rarity rarity) external onlyOwner {
        require(isRegistered[player], "Player not registered");
        _mintEquipment(player, equipmentType, rarity);
    }
    
    function listEquipmentForSale(uint256 tokenId, uint256 price) external {
        require(ownerOf(tokenId) == msg.sender, "Not the owner");
        require(price > 0, "Price must be greater than 0");
        
        equipments[tokenId].isForSale = true;
        equipments[tokenId].price = price;
        
        emit EquipmentListed(tokenId, price);
    }
    
    function removeFromSale(uint256 tokenId) external {
        require(ownerOf(tokenId) == msg.sender, "Not the owner");
        
        equipments[tokenId].isForSale = false;
        equipments[tokenId].price = 0;
    }
}
