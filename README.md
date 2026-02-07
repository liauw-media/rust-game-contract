# rust-game-contract

Shared types and Protocol Buffer definitions for game engine communication.

## Overview

This crate contains the shared type definitions and gRPC protocol definitions used for communication between the RGS and game engines. It's the Rust equivalent of `@omnitronix/game-engine-contract`.

**Reference Implementation:** [`game-engine-contract/`](https://github.com/liauw-media/omnitronix-monorepo) (TypeScript)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    rust-game-contract                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Protocol Buffers:                                          │
│  ├── game_engine.proto      (GameEngine service)           │
│  ├── rng.proto              (RNG service)                  │
│  └── common.proto           (shared message types)         │
│                                                             │
│  Generated Rust Types:                                      │
│  ├── GameActionCommand                                      │
│  ├── CommandProcessingResult                                │
│  ├── GameEngineInfo                                         │
│  ├── RngOutcome / RngOutcomeRecord                         │
│  ├── TriggeredBonus / BonusProgress                        │
│  └── CommandType (enum)                                     │
│                                                             │
│  Validation:                                                │
│  ├── Command validation                                     │
│  ├── State schema validation                                │
│  └── RNG outcome verification                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Protocol Buffers

### game_engine.proto

```protobuf
syntax = "proto3";
package game_engine.v1;

import "common.proto";

service GameEngineService {
  // Process a game command and return new state
  rpc ProcessCommand(ProcessCommandRequest) returns (ProcessCommandResponse);

  // Get game engine metadata
  rpc GetGameEngineInfo(Empty) returns (GameEngineInfoResponse);

  // Get available symbols for this game
  rpc GetSymbols(Empty) returns (GetSymbolsResponse);
}

message ProcessCommandRequest {
  optional bytes public_state = 1;   // JSON-encoded public state
  optional bytes private_state = 2;  // JSON-encoded private state
  GameActionCommand command = 3;
}

message ProcessCommandResponse {
  bool success = 1;
  bytes public_state = 2;            // JSON-encoded
  bytes private_state = 3;           // JSON-encoded
  optional bytes outcome = 4;        // JSON-encoded game outcome
  optional string message = 5;
  optional RngOutcome rng_outcome = 6;
}

message GameActionCommand {
  string id = 1;
  CommandType command_type = 2;
  optional bytes payload = 3;        // JSON-encoded command-specific payload
}

enum CommandType {
  COMMAND_TYPE_UNSPECIFIED = 0;
  COMMAND_TYPE_SPIN = 1;
  COMMAND_TYPE_GET_SYMBOLS = 2;
  COMMAND_TYPE_START_BONUS_ROUND = 3;
  COMMAND_TYPE_BONUS_SPIN = 4;
  // Debug commands (blocked in production)
  COMMAND_TYPE_DEBUG_TRIGGER_BONUS = 100;
  COMMAND_TYPE_DEBUG_FORCE_WIN = 101;
  COMMAND_TYPE_DEBUG_SET_RTP = 102;
  COMMAND_TYPE_DEBUG_UPDATE_BONUS_METER_PROGRESS = 103;
}

message GameEngineInfoResponse {
  string game_code = 1;
  string version = 2;
  double rtp = 3;
  GameType game_type = 4;
  string game_name = 5;
  string provider = 6;
}

enum GameType {
  GAME_TYPE_UNSPECIFIED = 0;
  GAME_TYPE_SLOT = 1;
  GAME_TYPE_TABLE = 2;
  GAME_TYPE_INSTANT = 3;
}

message GetSymbolsResponse {
  repeated Symbol symbols = 1;
}

message Symbol {
  string id = 1;
  string name = 2;
  string asset_path = 3;
  bool is_wild = 4;
  bool is_scatter = 5;
  bool is_bonus = 6;
}
```

### common.proto

```protobuf
syntax = "proto3";
package common.v1;

message Empty {}

message RngOutcome {
  map<string, RngOutcomeRecord> records = 1;
}

message RngOutcomeRecord {
  int64 result = 1;
  int64 seed = 2;
  int64 min = 3;
  int64 max = 4;
}

message TriggeredBonus {
  string bonus_id = 1;
  BonusType bonus_type = 2;
  string bet_amount = 3;  // Decimal as string
}

enum BonusType {
  BONUS_TYPE_UNSPECIFIED = 0;
  BONUS_TYPE_FREE_SPINS = 1;
  BONUS_TYPE_WHEEL_BONUS = 2;
  BONUS_TYPE_PICK_BONUS = 3;
}

message BonusProgress {
  uint32 completed = 1;
  uint32 remaining = 2;
  uint32 total = 3;
}
```

## Generated Rust Types

The build script generates Rust types from the protobuf definitions:

```rust
// Auto-generated from proto files
pub mod game_engine {
    pub mod v1 {
        tonic::include_proto!("game_engine.v1");
    }
}

pub mod common {
    pub mod v1 {
        tonic::include_proto!("common.v1");
    }
}
```

## Native Rust Types

For convenience, native Rust types are also provided:

```rust
use rust_macros::Decimal;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// Command types
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
pub enum CommandType {
    Spin,
    GetSymbols,
    StartBonusRound,
    BonusSpin,
    // Debug commands
    DebugTriggerBonus,
    DebugForceWin,
    DebugSetRtp,
    DebugUpdateBonusMeterProgress,
}

/// Incoming command from RGS to game engine
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GameActionCommand {
    pub id: String,
    pub command_type: CommandType,
    pub payload: Option<serde_json::Value>,
}

/// Response from game engine back to RGS
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CommandProcessingResult<TPublic, TPrivate, TOutcome> {
    pub success: bool,
    pub public_state: TPublic,
    pub private_state: TPrivate,
    pub outcome: Option<TOutcome>,
    pub message: Option<String>,
    pub rng_outcome: Option<RngOutcome>,
}

/// Game engine metadata
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct GameEngineInfo {
    pub game_code: String,
    pub version: String,
    pub rtp: f64,
    pub game_type: GameType,
    pub game_name: String,
    pub provider: String,
}

/// Game types
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum GameType {
    Slot,
    Table,
    Instant,
}

/// RNG outcome record for audit
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RngOutcomeRecord {
    pub result: i64,
    pub seed: i64,
    pub min: i64,
    pub max: i64,
}

/// Collection of RNG calls from one command
pub type RngOutcome = HashMap<String, RngOutcomeRecord>;

/// Triggered bonus info
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TriggeredBonus {
    pub bonus_id: String,
    pub bonus_type: BonusType,
    pub bet_amount: Decimal,
}

/// Bonus types
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum BonusType {
    FreeSpins,
    WheelBonus,
    PickBonus,
}

/// Bonus progress tracking
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct BonusProgress {
    pub completed: u32,
    pub remaining: u32,
    pub total: u32,
}
```

## Validation

```rust
use rust_game_contract::validation::{validate_command, validate_rng_outcome};

/// Validate a command before processing
pub fn validate_command(cmd: &GameActionCommand) -> Result<(), ValidationError> {
    // Check command ID is valid UUID
    // Check command type is not Debug in production
    // Validate payload schema for command type
}

/// Validate RNG outcome for audit
pub fn validate_rng_outcome(outcome: &RngOutcome) -> Result<(), ValidationError> {
    // Check all records have valid seeds
    // Verify results are within min/max bounds
}

/// Check if command is a debug command
pub fn is_debug_command(cmd_type: CommandType) -> bool {
    matches!(
        cmd_type,
        CommandType::DebugTriggerBonus
            | CommandType::DebugForceWin
            | CommandType::DebugSetRtp
            | CommandType::DebugUpdateBonusMeterProgress
    )
}
```

## Canonicalization

For GLI-19 audit hashing, RNG outcomes must be canonicalized:

```rust
use rust_game_contract::canonicalize::canonicalize_rng_outcome;

/// Canonicalize RNG outcome for deterministic hashing
/// Keys are sorted alphabetically, numbers are formatted consistently
pub fn canonicalize_rng_outcome(outcome: &RngOutcome) -> String {
    let mut sorted: Vec<_> = outcome.iter().collect();
    sorted.sort_by_key(|(k, _)| *k);

    let entries: Vec<String> = sorted
        .iter()
        .map(|(key, record)| {
            format!(
                "\"{}\":{{\"max\":{},\"min\":{},\"result\":{},\"seed\":{}}}",
                key, record.max, record.min, record.result, record.seed
            )
        })
        .collect();

    format!("{{{}}}", entries.join(","))
}
```

## Reference Files

| Rust Module | TypeScript Reference |
|-------------|---------------------|
| `proto/game_engine.proto` | `game-engine-contract/src/core/*.interface.ts` |
| `proto/common.proto` | `game-engine-contract/src/core/rng-outcome.interface.ts` |
| `src/types.rs` | `game-engine-contract/src/core/command.interface.ts` |
| `src/validation.rs` | `game-engine-contract/src/validation/` |
| `src/canonicalize.rs` | `game-engine-contract/src/utils/canonicalize.ts` |

## Usage

```toml
[dependencies]
rust-game-contract = { git = "https://github.com/liauw-media/rust-game-contract" }
```

```rust
use rust_game_contract::{
    CommandType,
    GameActionCommand,
    CommandProcessingResult,
    RngOutcome,
    canonicalize_rng_outcome,
};

// Create a spin command
let command = GameActionCommand {
    id: uuid::Uuid::new_v4().to_string(),
    command_type: CommandType::Spin,
    payload: Some(serde_json::json!({
        "bet_amount": "1.00",
        "bet_lines": 20
    })),
};

// Canonicalize RNG outcome for hashing
let hash_input = canonicalize_rng_outcome(&rng_outcome);
let hash = sha256::digest(hash_input);
```

## Features

- `proto` - Include generated protobuf types (default)
- `native` - Include native Rust types (default)
- `validation` - Include validation functions
- `serde` - Serde serialization support (default)

## License

MIT
