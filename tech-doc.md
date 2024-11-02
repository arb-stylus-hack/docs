# GamersDAO Technical Documentation
*Smart Contract Implementation Guide for Arbitrum Stylus*

## Table of Contents
1. [Development Setup](#development-setup)
2. [Contract Architecture](#contract-architecture)
3. [Core Smart Contracts](#core-smart-contracts)
4. [Implementation Guide](#implementation-guide)
5. [Testing Framework](#testing-framework)
6. [Deployment Process](#deployment-process)

## Development Setup

### Prerequisites
```bash
# Install Rust and Cargo
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Install Stylus CLI
cargo install cargo-stylus

# Install Dependencies
cargo add stylus-sdk
cargo add alloy-primitives
cargo add alloy-sol-types
```

### Project Structure
```
gamers-dao/
├── contracts/
│   ├── src/
│   │   ├── profile.rs
│   │   ├── matchmaking.rs
│   │   ├── wagering.rs
│   │   ├── achievements.rs
│   │   └── lib.rs
│   ├── tests/
│   └── Cargo.toml
├── scripts/
└── README.md
```

## Contract Architecture

### Core Components
1. Profile Management
2. Matchmaking System
3. Wagering System
4. Achievement Tracking

### Data Structures

```rust
// Profile Structure
#[derive(Debug, Default, Encode, Decode, TypeInfo)]
pub struct PlayerProfile {
    pub address: Address,
    pub username: String,
    pub gaming_accounts: Vec<GamingAccount>,
    pub reputation: u32,
    pub achievements: Vec<Achievement>,
}

// Matchmaking Structure
#[derive(Debug, Encode, Decode, TypeInfo)]
pub struct Match {
    pub id: U256,
    pub players: Vec<Address>,
    pub wager_amount: U256,
    pub status: MatchStatus,
    pub game_type: GameType,
    pub timestamp: u64,
}

// Wagering Structure
#[derive(Debug, Encode, Decode, TypeInfo)]
pub struct Wager {
    pub match_id: U256,
    pub amount: U256,
    pub token: Address,
    pub status: WagerStatus,
}
```

## Core Smart Contracts

### 1. Profile Contract
```rust
use stylus_sdk::{prelude::*, storage::StorageMap};

#[external]
impl ProfileContract {
    pub fn create_profile(&mut self, username: String) -> Result<(), Error> {
        let sender = msg::sender();
        ensure!(!self.profiles.contains(&sender), Error::ProfileExists);
        
        let profile = PlayerProfile {
            address: sender,
            username,
            gaming_accounts: Vec::new(),
            reputation: 0,
            achievements: Vec::new(),
        };
        
        self.profiles.insert(sender, profile);
        self.emit(ProfileCreated { address: sender });
        Ok(())
    }

    pub fn link_gaming_account(&mut self, platform: String, account_id: String) -> Result<(), Error> {
        let sender = msg::sender();
        let mut profile = self.profiles.get(&sender).ok_or(Error::ProfileNotFound)?;
        
        // Verify account ownership through oracle/API
        self.verify_gaming_account(&platform, &account_id)?;
        
        profile.gaming_accounts.push(GamingAccount { platform, account_id });
        self.profiles.insert(sender, profile);
        Ok(())
    }
}
```

### 2. Matchmaking Contract
```rust
#[external]
impl MatchmakingContract {
    pub fn create_match(&mut self, game_type: GameType, wager_amount: U256) -> Result<U256, Error> {
        let sender = msg::sender();
        ensure!(self.is_profile_valid(&sender), Error::InvalidProfile);
        
        let match_id = self.next_match_id();
        let new_match = Match {
            id: match_id,
            players: vec![sender],
            wager_amount,
            status: MatchStatus::Pending,
            game_type,
            timestamp: block::timestamp(),
        };
        
        self.matches.insert(match_id, new_match);
        self.emit(MatchCreated { id: match_id, creator: sender });
        Ok(match_id)
    }

    pub fn join_match(&mut self, match_id: U256) -> Result<(), Error> {
        let sender = msg::sender();
        let mut game_match = self.matches.get(&match_id).ok_or(Error::MatchNotFound)?;
        
        ensure!(game_match.status == MatchStatus::Pending, Error::InvalidMatchStatus);
        ensure!(!game_match.players.contains(&sender), Error::AlreadyJoined);
        
        game_match.players.push(sender);
        game_match.status = MatchStatus::InProgress;
        
        self.matches.insert(match_id, game_match);
        Ok(())
    }
}
```

### 3. Wagering Contract
```rust
#[external]
impl WageringContract {
    pub fn place_wager(&mut self, match_id: U256, token: Address) -> Result<(), Error> {
        let sender = msg::sender();
        let game_match = self.matches.get(&match_id).ok_or(Error::MatchNotFound)?;
        
        ensure!(game_match.status == MatchStatus::Pending, Error::InvalidMatchStatus);
        
        // Transfer tokens to contract
        let amount = game_match.wager_amount;
        self.transfer_tokens(sender, self.address(), token, amount)?;
        
        let wager = Wager {
            match_id,
            amount,
            token,
            status: WagerStatus::Active,
        };
        
        self.wagers.insert((match_id, sender), wager);
        Ok(())
    }

    pub fn resolve_wager(&mut self, match_id: U256, winner: Address) -> Result<(), Error> {
        // Only callable by oracle or verified game result
        self.ensure_authorized()?;
        
        let game_match = self.matches.get(&match_id).ok_or(Error::MatchNotFound)?;
        let wager = self.wagers.get(&(match_id, winner)).ok_or(Error::WagerNotFound)?;
        
        // Calculate rewards and transfer tokens
        let reward_amount = wager.amount * 2;
        self.transfer_tokens(self.address(), winner, wager.token, reward_amount)?;
        
        Ok(())
    }
}
```

## Implementation Guide

### 1. Setting Up a New Contract
```rust
use stylus_sdk::prelude::*;

#[contract]
mod gamers_dao {
    #[storage]
    struct Storage {
        profiles: StorageMap<Address, PlayerProfile>,
        matches: StorageMap<U256, Match>,
        wagers: StorageMap<(U256, Address), Wager>,
    }

    #[event]
    struct ProfileCreated {
        address: Address,
    }

    #[event]
    struct MatchCreated {
        id: U256,
        creator: Address,
    }
}
```

### 2. Implementing Game Verification
```rust
impl GameVerification {
    fn verify_gaming_account(&self, platform: &str, account_id: &str) -> Result<(), Error> {
        // Implementation will vary based on platform
        match platform {
            "steam" => self.verify_steam_account(account_id),
            "epic" => self.verify_epic_account(account_id),
            "riot" => self.verify_riot_account(account_id),
            _ => Err(Error::UnsupportedPlatform),
        }
    }

    fn verify_game_result(&self, match_id: U256) -> Result<Address, Error> {
        // Implement game result verification logic
        // This could involve oracle calls or direct API integration
        Ok(winner_address)
    }
}
```

## Testing Framework

### Unit Tests
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_create_profile() {
        let mut contract = ProfileContract::new();
        let result = contract.create_profile("testuser".to_string());
        assert!(result.is_ok());
    }

    #[test]
    fn test_matchmaking() {
        let mut contract = MatchmakingContract::new();
        let match_id = contract.create_match(GameType::Competitive, U256::from(100)).unwrap();
        assert!(contract.matches.contains(&match_id));
    }
}
```

## Deployment Process

### 1. Compile Contracts
```bash
cargo stylus check
```

### 2. Deploy to Testnet
```bash
cargo stylus deploy --network arbitrum-stylus-testnet
```

### 3. Verify Contracts
```bash
cargo stylus verify <CONTRACT_ADDRESS>
```

## Security Considerations

1. **Access Control**
   - Implement role-based access for admin functions
   - Verify game results through trusted oracles
   - Secure token handling in wagering system

2. **Data Validation**
   - Validate all user inputs
   - Check game account ownership
   - Verify match participants

3. **Fund Safety**
   - Implement secure escrow for wagers
   - Use SafeERC20 for token transfers
   - Include emergency withdrawal mechanisms

## Best Practices

1. **Gas Optimization**
   - Use appropriate data structures
   - Batch operations when possible
   - Minimize storage operations

2. **Error Handling**
   - Implement comprehensive error types
   - Use Result types for fallible operations
   - Provide clear error messages

3. **Event Emission**
   - Emit events for all state changes
   - Include relevant indexed parameters
   - Document event structures

## Integration Guidelines

### Frontend Integration
```typescript
// Example TypeScript integration
const profileContract = new ethers.Contract(
    PROFILE_CONTRACT_ADDRESS,
    PROFILE_ABI,
    signer
);

async function createProfile(username: string) {
    const tx = await profileContract.create_profile(username);
    await tx.wait();
}

async function createMatch(gameType: number, wagerAmount: BigNumber) {
    const tx = await matchContract.create_match(gameType, wagerAmount);
    await tx.wait();
}
```

### API Integration
```rust
// Example API endpoint integration
pub async fn verify_game_result(match_id: U256) -> Result<Address, Error> {
    let api_response = reqwest::get(&format!(
        "{}/api/match_result/{}", 
        API_BASE_URL, 
        match_id
    )).await?;
    
    // Process API response and return winner
    Ok(winner_address)
}
```

## Maintenance and Upgrades

1. **Contract Upgrades**
   - Use proxy pattern for upgradeability
   - Maintain state compatibility
   - Document upgrade procedures

2. **Monitoring**
   - Implement logging
   - Monitor contract events
   - Track gas usage

3. **Emergency Procedures**
   - Document emergency shutdown procedures
   - Implement circuit breakers
   - Define recovery processes