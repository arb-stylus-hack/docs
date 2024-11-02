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

### 4. Badge NFT Contract
```rust
use stylus_sdk::{prelude::*, erc::erc721::*};

#[derive(Debug, Encode, Decode, TypeInfo)]
pub struct Badge {
    pub badge_type: BadgeType,
    pub metadata_uri: String,
    pub earned_timestamp: u64,
    pub rarity: BadgeRarity,
}

#[derive(Debug, Encode, Decode, TypeInfo)]
pub enum BadgeType {
    Achievement(String),    // Achievement name
    Streak(u32),           // Streak days
    Ranking(String),       // Rank level
    Tournament(String),    // Tournament name
    Special(String),       // Special event name
}

#[derive(Debug, Encode, Decode, TypeInfo)]
pub enum BadgeRarity {
    Common,
    Rare,
    Epic,
    Legendary,
}

#[external]
impl BadgeContract {
    #[storage]
    struct Storage {
        badges: StorageMap<U256, Badge>,
        player_badges: StorageMap<Address, Vec<U256>>,
        next_badge_id: U256,
        badge_uri_base: String,
    }

    pub fn mint_achievement_badge(
        &mut self, 
        recipient: Address, 
        achievement: String,
        rarity: BadgeRarity
    ) -> Result<U256, Error> {
        self.ensure_authorized()?;
        
        let badge_id = self.next_badge_id();
        let badge = Badge {
            badge_type: BadgeType::Achievement(achievement.clone()),
            metadata_uri: self.generate_metadata_uri(badge_id),
            earned_timestamp: block::timestamp(),
            rarity,
        };
        
        self.badges.insert(badge_id, badge);
        
        // Update player badges
        let mut player_badges = self.player_badges
            .get(&recipient)
            .unwrap_or_default();
        player_badges.push(badge_id);
        self.player_badges.insert(recipient, player_badges);
        
        self.emit(BadgeMinted {
            recipient,
            badge_id,
            badge_type: "achievement",
        });
        
        Ok(badge_id)
    }

    pub fn mint_streak_badge(
        &mut self, 
        recipient: Address, 
        streak_days: u32
    ) -> Result<U256, Error> {
        self.ensure_authorized()?;
        
        // Determine rarity based on streak length
        let rarity = match streak_days {
            0..=7 => BadgeRarity::Common,
            8..=30 => BadgeRarity::Rare,
            31..=90 => BadgeRarity::Epic,
            _ => BadgeRarity::Legendary,
        };
        
        let badge_id = self.next_badge_id();
        let badge = Badge {
            badge_type: BadgeType::Streak(streak_days),
            metadata_uri: self.generate_metadata_uri(badge_id),
            earned_timestamp: block::timestamp(),
            rarity,
        };
        
        self.mint_badge(recipient, badge_id, badge)
    }

    pub fn mint_tournament_badge(
        &mut self, 
        recipient: Address, 
        tournament: String,
        rarity: BadgeRarity
    ) -> Result<U256, Error> {
        self.ensure_authorized()?;
        
        let badge_id = self.next_badge_id();
        let badge = Badge {
            badge_type: BadgeType::Tournament(tournament),
            metadata_uri: self.generate_metadata_uri(badge_id),
            earned_timestamp: block::timestamp(),
            rarity,
        };
        
        self.mint_badge(recipient, badge_id, badge)
    }

    // Internal helper functions
    fn mint_badge(
        &mut self,
        recipient: Address,
        badge_id: U256,
        badge: Badge
    ) -> Result<U256, Error> {
        self.badges.insert(badge_id, badge);
        
        let mut player_badges = self.player_badges
            .get(&recipient)
            .unwrap_or_default();
        player_badges.push(badge_id);
        self.player_badges.insert(recipient, player_badges);
        
        self._mint(recipient, badge_id)?;
        
        Ok(badge_id)
    }

    fn generate_metadata_uri(&self, badge_id: U256) -> String {
        format!("{}/{}", self.badge_uri_base, badge_id)
    }

    // View functions
    pub fn get_player_badges(&self, player: Address) -> Vec<U256> {
        self.player_badges.get(&player).unwrap_or_default()
    }

    pub fn get_badge_details(&self, badge_id: U256) -> Result<Badge, Error> {
        self.badges.get(&badge_id).ok_or(Error::BadgeNotFound)
    }
}
```

### 5. Achievement Tracker Contract
```rust
#[external]
impl AchievementTrackerContract {
    #[storage]
    struct Storage {
        achievements: StorageMap<String, AchievementCriteria>,
        player_progress: StorageMap<(Address, String), u32>,
        badge_contract: Address,
    }

    pub fn register_achievement(&mut self, 
        name: String, 
        criteria: AchievementCriteria
    ) -> Result<(), Error> {
        self.ensure_admin()?;
        self.achievements.insert(name, criteria);
        Ok(())
    }

    pub fn update_progress(&mut self, 
        player: Address, 
        achievement: String, 
        progress: u32
    ) -> Result<(), Error> {
        self.ensure_authorized()?;
        
        let criteria = self.achievements
            .get(&achievement)
            .ok_or(Error::AchievementNotFound)?;
            
        let current_progress = self.player_progress
            .get(&(player, achievement.clone()))
            .unwrap_or_default();
            
        let new_progress = current_progress + progress;
        
        // Check if achievement is completed
        if new_progress >= criteria.required_progress 
            && current_progress < criteria.required_progress {
            // Mint achievement badge
            let badge_contract = BadgeContract::new(self.badge_contract);
            badge_contract.mint_achievement_badge(
                player,
                achievement.clone(),
                criteria.badge_rarity
            )?;
        }
        
        self.player_progress.insert((player, achievement), new_progress);
        Ok(())
    }
}
```

### 6. Streak Tracking Contract
```rust
#[external]
impl StreakTrackerContract {
    #[storage]
    struct Storage {
        player_streaks: StorageMap<Address, PlayerStreak>,
        badge_contract: Address,
    }

    pub fn check_in(&mut self, player: Address) -> Result<(), Error> {
        let current_time = block::timestamp();
        let mut streak = self.player_streaks
            .get(&player)
            .unwrap_or_default();
            
        if self.is_consecutive_day(streak.last_check_in, current_time) {
            streak.current_streak += 1;
            
            // Check if new streak badge should be minted
            if self.should_mint_streak_badge(streak.current_streak) {
                let badge_contract = BadgeContract::new(self.badge_contract);
                badge_contract.mint_streak_badge(
                    player,
                    streak.current_streak
                )?;
            }
        } else {
            streak.current_streak = 1;
        }
        
        streak.last_check_in = current_time;
        self.player_streaks.insert(player, streak);
        Ok(())
    }

    fn should_mint_streak_badge(&self, streak_days: u32) -> bool {
        // Mint badges at specific milestones
        matches!(streak_days, 7 | 30 | 90 | 180 | 365)
    }
}
```

### Integration with Profile Contract
```rust
impl ProfileContract {
    pub fn get_player_achievements(&self, player: Address) -> Vec<Badge> {
        let badge_contract = BadgeContract::new(self.badge_contract);
        let badge_ids = badge_contract.get_player_badges(player);
        
        badge_ids
            .iter()
            .filter_map(|&id| badge_contract.get_badge_details(id).ok())
            .collect()
    }

    pub fn get_player_stats(&self, player: Address) -> PlayerStats {
        let mut stats = PlayerStats::default();
        
        // Get badges
        let badges = self.get_player_achievements(player);
        
        // Calculate stats based on badges
        for badge in badges {
            match badge.badge_type {
                BadgeType::Achievement(_) => stats.achievement_count += 1,
                BadgeType::Streak(days) => stats.longest_streak = stats.longest_streak.max(days),
                BadgeType::Tournament(_) => stats.tournament_badges += 1,
                BadgeType::Ranking(_) => stats.ranking_badges += 1,
                BadgeType::Special(_) => stats.special_badges += 1,
            }
        }
        
        stats
    }
}
```

### Metadata Generation
```rust
impl BadgeContract {
    fn generate_metadata(&self, badge: &Badge) -> String {
        let metadata = json!({
            "name": self.get_badge_name(&badge.badge_type),
            "description": self.get_badge_description(&badge.badge_type),
            "image": self.get_badge_image_uri(&badge.badge_type, &badge.rarity),
            "attributes": [
                {
                    "trait_type": "Badge Type",
                    "value": self.get_badge_type_string(&badge.badge_type)
                },
                {
                    "trait_type": "Rarity",
                    "value": format!("{:?}", badge.rarity)
                },
                {
                    "trait_type": "Earned Date",
                    "value": badge.earned_timestamp
                }
            ]
        });
        
        serde_json::to_string(&metadata).unwrap_or_default()
    }
    
    fn get_badge_image_uri(
        &self,
        badge_type: &BadgeType,
        rarity: &BadgeRarity
    ) -> String {
        format!(
            "{}/images/{}/{}.png",
            self.badge_uri_base,
            self.get_badge_type_string(badge_type).to_lowercase(),
            format!("{:?}", rarity).to_lowercase()
        )
    }
}
```

1. Badge NFT Contract with support for different badge types:
   - Achievement badges
   - Streak badges
   - Tournament badges
   - Ranking badges
   - Special event badges

2. Achievement Tracker Contract for:
   - Tracking progress toward achievements
   - Automatically minting badges when achievements are completed
   - Managing achievement criteria

3. Streak Tracking Contract for:
   - Daily check-ins
   - Streak maintenance
   - Automatic badge minting at milestones

4. Integration with existing Profile Contract to:
   - Display player achievements
   - Calculate stats based on badges
   - Showcase player progression

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