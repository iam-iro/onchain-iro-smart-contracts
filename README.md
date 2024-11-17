# iRO Smart Contract Development Documentation

## Overview
The iRO (Interactive Robot) is a personality-driven NFT project built on Base network. Each iRO token represents a unique robot companion with evolving personality traits based on user interactions.

## Technical Stack
- Solidity: ^0.8.20
- OpenZeppelin Contracts
- Base Network
- ERC721 Standard

## Smart Contract Architecture

### Core Contracts

#### 1. iROPersonality.sol
Main contract handling robot personality and interactions.

```solidity
contract iROPersonality is ERC721, Ownable {
    struct Personality {
        uint256 bondingLevel;      // 0-100
        uint256 emotionalIQ;       // 0-100
        uint256 playfulness;       // 0-100
        uint256 attentiveness;     // 0-100
        uint256 interactions;      // Total interactions count
        uint256 lastInteraction;   // Last interaction timestamp
    }
}
```

### Key Components

#### 1. Personality System
```solidity
// Personality trait mapping
mapping(uint256 => Personality) public personalities;

// Initialization
function _initializePersonality(uint256 tokenId) internal {
    personalities[tokenId] = Personality({
        bondingLevel: 10,
        emotionalIQ: 10,
        playfulness: 10,
        attentiveness: 10,
        interactions: 0,
        lastInteraction: block.timestamp
    });
}
```

#### 2. Interaction System
```solidity
// Interaction types
string constant GENTLE = "gentle";
string constant PLAYFUL = "playful";
string constant LONG_PRESS = "long";

function interact(uint256 tokenId, string memory interactionType) public {
    require(_exists(tokenId), "iRO: Token does not exist");
    require(ownerOf(tokenId) == msg.sender, "iRO: Not the owner");
    
    // Process interaction
    _processInteraction(tokenId, interactionType);
    
    emit InteractionRegistered(tokenId, interactionType);
}
```

## Development Guidelines

### 1. Trait Calculations

Personality traits are calculated based on interaction types:

```solidity
function _calculateTraitIncrease(
    uint256 currentValue,
    uint256 increment
) internal pure returns (uint256) {
    // Max value is 100
    return currentValue + increment > 100 ? 100 : currentValue + increment;
}

function _updateTraits(
    Personality storage personality,
    string memory interactionType
) internal {
    if (keccak256(bytes(interactionType)) == keccak256(bytes(GENTLE))) {
        personality.bondingLevel = _calculateTraitIncrease(personality.bondingLevel, 1);
        personality.emotionalIQ = _calculateTraitIncrease(personality.emotionalIQ, 1);
    } else if (keccak256(bytes(interactionType)) == keccak256(bytes(PLAYFUL))) {
        personality.playfulness = _calculateTraitIncrease(personality.playfulness, 1);
        personality.attentiveness = _calculateTraitIncrease(personality.attentiveness, 1);
    }
}
```

### 2. Events and Logging

```solidity
// Core events
event PersonalityUpdated(
    uint256 indexed tokenId,
    uint256 bondingLevel,
    uint256 emotionalIQ
);

event InteractionRegistered(
    uint256 indexed tokenId,
    string interactionType
);

event TraitIncreased(
    uint256 indexed tokenId,
    string traitName,
    uint256 newValue
);
```

### 3. Security Considerations

1. Access Control
```solidity
modifier onlyTokenOwner(uint256 tokenId) {
    require(ownerOf(tokenId) == msg.sender, "iRO: Not token owner");
    _;
}

modifier validTokenId(uint256 tokenId) {
    require(_exists(tokenId), "iRO: Invalid token ID");
    _;
}
```

2. Rate Limiting
```solidity
mapping(uint256 => uint256) private lastInteractionTime;

modifier rateLimited(uint256 tokenId) {
    require(
        block.timestamp - lastInteractionTime[tokenId] >= 1 hours,
        "iRO: Interaction too frequent"
    );
    _;
}
```

### 4. Upgradability

The contract supports upgradeability through the OpenZeppelin upgrades plugin:

```solidity
// Implementing upgradeable contract
contract iROPersonalityUpgradeable is 
    Initializable,
    ERC721Upgradeable,
    OwnableUpgradeable
{
    function initialize() public initializer {
        __ERC721_init("iRO Robot", "IRO");
        __Ownable_init(msg.sender);
    }
}
```

## Testing Guidelines

### 1. Unit Tests

```javascript
describe("iROPersonality", function() {
    let iRO;
    let owner;
    let user;

    beforeEach(async function() {
        const IRO = await ethers.getContractFactory("iROPersonality");
        [owner, user] = await ethers.getSigners();
        iRO = await IRO.deploy();
        await iRO.deployed();
    });

    it("should initialize with correct traits", async function() {
        await iRO.connect(user).mint();
        const personality = await iRO.getPersonality(0);
        expect(personality.bondingLevel).to.equal(10);
    });
});
```

### 2. Integration Tests

```javascript
describe("Integration", function() {
    it("should properly evolve personality over multiple interactions", async function() {
        await iRO.connect(user).mint();
        await iRO.connect(user).interact(0, "gentle");
        await network.provider.send("evm_increaseTime", [3600]);
        await iRO.connect(user).interact(0, "playful");
        
        const personality = await iRO.getPersonality(0);
        expect(personality.interactions).to.equal(2);
    });
});
```

## Deployment Process

1. Deploy Contract:
```bash
npx hardhat run scripts/deploy.js --network base
```

2. Verify Contract:
```bash
npx hardhat verify --network base DEPLOYED_ADDRESS
```

3. Set Base URI:
```javascript
await iRO.setBaseURI("https://api.iro.ai/metadata/");
```

## Metadata Structure

```json
{
    "name": "iRO #1",
    "description": "Interactive Robot Companion",
    "image": "https://api.iro.ai/images/1.png",
    "attributes": [
        {
            "trait_type": "Bonding Level",
            "value": 45
        },
        {
            "trait_type": "Emotional IQ",
            "value": 32
        },
        {
            "trait_type": "Playfulness",
            "value": 78
        },
        {
            "trait_type": "Attentiveness",
            "value": 56
        }
    ]
}
```

## Gas Optimization Tips

1. Use packed structs:
```solidity
struct PackedPersonality {
    uint64 bondingLevel;
    uint64 emotionalIQ;
    uint64 playfulness;
    uint64 attentiveness;
}
```

2. Batch operations when possible:
```solidity
function batchInteract(uint256[] calldata tokenIds, string[] calldata types) external {
    require(tokenIds.length == types.length, "Length mismatch");
    for(uint i = 0; i < tokenIds.length; i++) {
        _processInteraction(tokenIds[i], types[i]);
    }
}
```

## Error Handling

```solidity
error InvalidInteraction(string reason);
error UnauthorizedAccess(address caller, uint256 tokenId);
error RateLimitExceeded(uint256 tokenId, uint256 nextValidTime);

function interact(uint256 tokenId, string memory interactionType) public {
    if (!_exists(tokenId)) revert InvalidInteraction("Token does not exist");
    if (ownerOf(tokenId) != msg.sender) revert UnauthorizedAccess(msg.sender, tokenId);
    // ... rest of the function
}
```
