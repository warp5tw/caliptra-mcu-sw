# Flash Module Architecture Analysis

## Overview

This document provides a comprehensive analysis of the flash module architecture in the Caliptra MCU Software repository. The requested file path `npcm400-main/platforms/nuvoton/npcm400/rom/src/flash/mod.rs` does not currently exist in the repository. However, this analysis covers the existing flash module structure under `platforms/emulator/rom/src/flash/` which can serve as a reference implementation for creating similar platform-specific flash modules, including a potential npcm400 platform.

## Repository Structure

The flash-related code is organized across multiple directories:

```
caliptra-mcu-sw/
├── platforms/emulator/rom/src/flash/          # Platform-specific flash implementation
│   ├── mod.rs                                  # Module exports
│   ├── flash_drv.rs                           # Emulated flash controller driver
│   ├── flash_boot_cfg.rs                      # Flash boot configuration
│   └── flash_test.rs                          # Flash testing utilities
├── rom/src/flash/                             # Common flash abstractions
│   ├── mod.rs                                 # Module exports
│   ├── hil.rs                                 # Hardware Interface Layer traits
│   └── flash_partition.rs                     # Partition management
└── runtime/kernel/drivers/flash/              # Runtime flash drivers
    ├── src/lib.rs
    ├── src/flash_ctrl.rs
    ├── src/hil.rs
    └── src/flash_storage_to_pages.rs
```

## Module Components Analysis

### 1. Flash Module Exports (`platforms/emulator/rom/src/flash/mod.rs`)

**Location:** `platforms/emulator/rom/src/flash/mod.rs`

**Purpose:** Module organization and selective export of flash components.

**Content:**
```rust
// Licensed under the Apache-2.0 license
pub mod flash_boot_cfg;
pub mod flash_drv;

#[cfg(any(
    feature = "test-mcu-rom-flash-access",
    feature = "test-flash-based-boot"
))]
pub mod flash_test;
```

**Key Points:**
- Exports the core flash driver (`flash_drv`)
- Exports boot configuration management (`flash_boot_cfg`)
- Conditionally includes test utilities based on feature flags
- Simple, clean module structure

### 2. Flash Hardware Interface Layer (`rom/src/flash/hil.rs`)

**Location:** `rom/src/flash/hil.rs`

**Purpose:** Define generic traits and error types for flash storage operations.

**Key Components:**

#### FlashStorage Trait
```rust
pub trait FlashStorage {
    fn read(&self, buffer: &mut [u8], address: usize) -> Result<(), FlashDrvError>;
    fn write(&self, buffer: &[u8], address: usize) -> Result<(), FlashDrvError>;
    fn erase(&self, address: usize, length: usize) -> Result<(), FlashDrvError>;
    fn capacity(&self) -> usize;
}
```

**Design Principles:**
- **Generic interface:** Works with any flash storage implementation
- **Simple API:** Four basic operations (read, write, erase, capacity)
- **Error handling:** Uses custom `FlashDrvError` enum
- **Flexibility:** Can be implemented by different hardware platforms

#### FlashDrvError Enum
Comprehensive error type covering 13 different error conditions:
- `FAIL`: Generic failure
- `BUSY`: System busy, retry needed
- `ALREADY`: Requested state already set
- `OFF`: Component powered down
- `RESERVE`: Reservation required
- `INVAL`: Invalid parameter
- `SIZE`: Parameter too large
- `CANCEL`: Operation canceled
- `NOMEM`: Memory not available
- `NOSUPPORT`: Operation not supported
- `NODEVICE`: Device not available
- `UNINSTALLED`: Device not physically installed
- `NOACK`: Transmission not acknowledged

**Key Features:**
- Comprehensive error coverage
- Conversion traits to/from `Result<(), FlashDrvError>`
- Numeric representation for FFI compatibility
- Clear semantic meaning for each error

### 3. Flash Partition Management (`rom/src/flash/flash_partition.rs`)

**Location:** `rom/src/flash/flash_partition.rs`

**Purpose:** Provide partition-based access to flash storage with bounds checking.

**Architecture:**

```rust
pub struct FlashPartition<'a> {
    driver: &'a dyn FlashStorage,
    name: &'static str,
    base_offset: usize,
    length: usize,
}
```

**Key Features:**
1. **Bounds checking:** All operations verify they stay within partition boundaries
2. **Named partitions:** Each partition has a debug-friendly name
3. **Offset translation:** Automatically translates partition-relative addresses to absolute flash addresses
4. **Safe abstraction:** Prevents accidental writes outside partition boundaries

**API Methods:**
- `new()`: Create a partition with validation
- `read()`: Read from partition with bounds checking
- `write()`: Write to partition with bounds checking
- `erase()`: Erase within partition with bounds checking
- `len()`: Get partition size
- `is_empty()`: Check if partition has zero length
- `name()`: Get partition name

**Safety Features:**
- Creation-time validation ensures partition fits in flash
- Runtime bounds checking on every operation
- Returns `FlashDrvError::SIZE` for out-of-bounds operations

### 4. Emulated Flash Controller Driver (`platforms/emulator/rom/src/flash/flash_drv.rs`)

**Location:** `platforms/emulator/rom/src/flash/flash_drv.rs`

**Purpose:** Platform-specific implementation of flash controller for the emulator.

**Architecture:**

#### Constants
```rust
const PAGE_SIZE: usize = 256;
const FLASH_MAX_PAGES: usize = 64 * 1024 * 1024 / PAGE_SIZE;  // 64MB
```

#### Flash Operations
```rust
pub enum FlashOperation {
    ReadPage = 1,
    WritePage = 2,
    ErasePage = 3,
}
```

#### EmulatedFlashCtrl Structure
```rust
pub struct EmulatedFlashCtrl {
    registers: StaticRef<PrimaryFlashCtrl>,
}
```

**Key Features:**

1. **Page-based operations:** All operations work on 256-byte pages
2. **Memory-mapped registers:** Uses hardware register interface
3. **Interrupt-based completion:** Polls interrupt status for operation completion
4. **Automatic page alignment:** Handles partial page reads/writes automatically

**Implementation Details:**

##### Read Operation
- Supports arbitrary length reads
- Automatically handles page boundaries
- Copies data from multiple pages as needed
- Efficient buffer management

##### Write Operation
- Supports arbitrary length writes
- Read-modify-write for partial pages
- Maintains data outside the write region
- Ensures data integrity

##### Erase Operation
- Calculates start and end pages
- Erases all pages in the range
- Handles partial page erasure

**Hardware Interaction:**
- Uses memory-mapped registers (`PrimaryFlashCtrl`)
- Interrupt enable/disable for error and event handling
- Status polling for synchronous completion
- Register-based control flow

**Polling Mechanism:**
```rust
fn poll_for_completion(&self) -> Result<(), FlashDrvError> {
    loop {
        // Check event interrupt (success)
        // Check error interrupt (failure)
        // Clear status and return
    }
}
```

### 5. Flash Boot Configuration (`platforms/emulator/rom/src/flash/flash_boot_cfg.rs`)

**Location:** `platforms/emulator/rom/src/flash/flash_boot_cfg.rs`

**Purpose:** Manage boot configuration stored in flash, including partition selection and status.

**Architecture:**

```rust
pub struct FlashBootCfg<'a> {
    flash_driver: &'a mut FlashPartition<'a>,
}
```

**Key Features:**

1. **Partition Table Management:**
   - Read partition table from flash
   - Verify checksum integrity
   - Update partition table safely

2. **Boot Configuration API** (implements `BootConfig` trait):
   - `get_active_partition()`: Determine which partition to boot from
   - `set_active_partition()`: Set the active boot partition
   - `increment_boot_count()`: Track boot attempts
   - `get_boot_count()`: Retrieve boot attempt count
   - `set_rollback_enable()`: Enable/disable rollback functionality
   - `set_partition_status()`: Update partition status (Valid/Invalid/Testing)
   - `get_partition_status()`: Query partition status
   - `is_rollback_enabled()`: Check rollback configuration

3. **Data Integrity:**
   - Checksum verification on reads
   - Checksum calculation on writes
   - Uses `StandAloneChecksumCalculator`

4. **Partition Support:**
   - Dual partition support (A and B)
   - Per-partition boot counts
   - Per-partition status tracking

**Usage Pattern:**
```rust
let mut partition_table_driver = FlashPartition::new(...)?;
let boot_cfg = FlashBootCfg::new(&mut partition_table_driver);
let active = boot_cfg.get_active_partition()?;
```

### 6. Flash Testing Utilities (`platforms/emulator/rom/src/flash/flash_test.rs`)

**Location:** `platforms/emulator/rom/src/flash/flash_test.rs`

**Purpose:** Comprehensive testing of flash operations.

**Test Strategy:**

```rust
pub fn test_rom_flash_access(partition: &FlashPartition) {
    const TEST_DATA_SIZE: usize = 1024;
    const NUM_ITER: usize = 4;
    const ADDR_STEP: usize = 0x1000;
    
    // Test erase -> write -> read -> verify cycle
    // Multiple iterations with different addresses
}
```

**Test Coverage:**
- Erase operations
- Write operations with test patterns
- Read operations
- Data verification
- Multiple iterations at different addresses
- Error handling and reporting

## Integration in ROM

The flash modules are integrated into the ROM as shown in `platforms/emulator/rom/src/riscv.rs`:

### Initialization Sequence

```rust
// 1. Initialize flash controllers
let primary_flash_ctrl = EmulatedFlashCtrl::initialize_flash_ctrl(PRIMARY_FLASH_CTRL_BASE);
let secondary_flash_ctrl = EmulatedFlashCtrl::initialize_flash_ctrl(SECONDARY_FLASH_CTRL_BASE);

// 2. Create partition table partition
let partition_table_driver = FlashPartition::new(
    &primary_flash_ctrl,
    "Partition Table",
    PARTITION_TABLE.offset,
    PARTITION_TABLE.size,
)?;

// 3. Initialize boot configuration
let boot_cfg = FlashBootCfg::new(&mut partition_table_driver);

// 4. Determine active partition
let active_partition = boot_cfg.get_active_partition()?;

// 5. Create image partitions
let partition_a = FlashPartition::new(...)?;
let partition_b = FlashPartition::new(...)?;

// 6. Select appropriate partition and pass to ROM
let flash_image_partition_driver = match active_partition {
    PartitionId::A => partition_a,
    PartitionId::B => partition_b,
    _ => fatal_error(...),
};

mcu_rom_common::rom_start(RomParameters {
    flash_partition_driver: Some(&mut flash_image_partition_driver),
    ..Default::default()
});
```

## Design Patterns and Best Practices

### 1. Layered Architecture
- **HIL (Hardware Interface Layer):** Generic traits
- **Driver Layer:** Platform-specific implementation
- **Partition Layer:** Logical partitioning
- **Boot Configuration Layer:** High-level boot logic

### 2. Error Handling
- Comprehensive error types
- Propagation using `Result<T, E>`
- Early validation and bounds checking
- Clear error semantics

### 3. Safety
- Bounds checking at multiple layers
- Partition isolation
- Checksum verification
- Immutable references where possible

### 4. Modularity
- Clear separation of concerns
- Platform-independent interfaces
- Platform-specific implementations
- Feature flag gating for tests

### 5. Testing
- Conditional compilation for tests
- Comprehensive test coverage
- Error path testing
- Integration testing support

## Memory Map

```
Flash Layout (Example):
┌─────────────────────────┐ 0x00000000
│  Partition Table        │
├─────────────────────────┤
│  ...                    │
├─────────────────────────┤ IMAGE_A_PARTITION.offset
│  Image A (Primary)      │
├─────────────────────────┤
│  ...                    │
├─────────────────────────┤ IMAGE_B_PARTITION.offset
│  Image B (Secondary)    │
├─────────────────────────┤
│  ...                    │
└─────────────────────────┘ 64MB
```

## Dependencies

### Crate Dependencies
- `mcu-config`: Boot configuration traits
- `mcu-config-emulator`: Platform-specific configuration
- `mcu-rom-common`: Common ROM functionality
- `registers-generated`: Hardware register definitions
- `romtime`: Utilities and printing
- `tock-registers`: Register access abstractions
- `zerocopy`: Zero-copy parsing

### Feature Flags
- `test-mcu-rom-flash-access`: Enable flash access tests
- `test-flash-based-boot`: Enable flash-based boot flow
- `hw-2-1`: Hardware revision 2.1 features

## Recommendations for npcm400 Platform Implementation

If implementing a similar flash module for the npcm400 platform at `platforms/nuvoton/npcm400/rom/src/flash/`, consider:

### 1. Directory Structure
```
platforms/nuvoton/npcm400/rom/src/flash/
├── mod.rs                    # Module exports
├── flash_drv.rs             # NPCM400-specific flash controller
├── flash_boot_cfg.rs        # Boot configuration (may reuse emulator version)
└── flash_test.rs            # Platform-specific tests (optional)
```

### 2. Implementation Strategy

**Step 1:** Define hardware specifics
- Page size for NPCM400 flash
- Flash capacity
- Register addresses
- Operation modes

**Step 2:** Implement `FlashStorage` trait
```rust
pub struct Npcm400FlashCtrl {
    registers: StaticRef<Npcm400FlashCtrlRegs>,
}

impl FlashStorage for Npcm400FlashCtrl {
    fn read(&self, buffer: &mut [u8], address: usize) -> Result<(), FlashDrvError> {
        // NPCM400-specific implementation
    }
    // ... other methods
}
```

**Step 3:** Reuse existing abstractions
- Use `FlashPartition` from `rom/src/flash/flash_partition.rs`
- Implement or adapt `FlashBootCfg` based on requirements
- Follow the same initialization pattern as emulator

**Step 4:** Integration
- Update `platforms/nuvoton/npcm400/rom/src/main.rs`
- Add module declaration: `mod flash;`
- Initialize flash in ROM entry point
- Configure memory map for NPCM400

### 3. Key Considerations

**Hardware Differences:**
- NPCM400 may have different page sizes
- Different register interfaces
- Different timing requirements
- Different interrupt handling

**Compatibility:**
- Maintain `FlashStorage` trait compatibility
- Keep partition table format compatible if possible
- Use same boot configuration interface

**Testing:**
- Adapt flash tests for NPCM400 specifics
- Validate against real hardware
- Test error paths thoroughly

**Documentation:**
- Document NPCM400-specific behavior
- Note differences from emulator implementation
- Provide hardware reference links

## Conclusion

The existing flash module architecture provides a well-structured, layered approach to flash management with clear separation between hardware-specific and platform-independent code. The design emphasizes:

1. **Safety:** Comprehensive bounds checking and error handling
2. **Modularity:** Clear interfaces and abstractions
3. **Reusability:** Platform-independent code in common modules
4. **Testability:** Feature-gated tests and utilities
5. **Maintainability:** Clean code structure and documentation

This architecture serves as an excellent template for implementing flash support for new platforms like the NPCM400, requiring primarily platform-specific driver implementation while reusing the robust common infrastructure.
