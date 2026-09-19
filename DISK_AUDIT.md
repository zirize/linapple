# Comprehensive Function Audit & Architectural Review: `Disk.cpp`

**Target Source File**: [`src/apple2/peripherals/disk/Disk.cpp`](file:///home/maxolasersquad/code/linapple/src/apple2/peripherals/disk/Disk.cpp)  
**Associated Headers**:
- [`src/apple2/peripherals/disk/Disk.h`](file:///home/maxolasersquad/code/linapple/src/apple2/peripherals/disk/Disk.h)
- [`src/apple2/peripherals/disk/DiskCommands.h`](file:///home/maxolasersquad/code/linapple/src/apple2/peripherals/disk/DiskCommands.h)
- [`src/apple2/peripherals/disk/DiskError.h`](file:///home/maxolasersquad/code/linapple/src/apple2/peripherals/disk/DiskError.h)
- [`src/apple2/peripherals/disk/DiskFormatDriver.h`](file:///home/maxolasersquad/code/linapple/src/apple2/peripherals/disk/DiskFormatDriver.h)

**Reviewing Bodies**:
- **Peripheral Architect**: 7-Pillar Peripheral Audit Protocol (Header Isolation, State Encapsulation, Bus Seam Fidelity, Host Boundary Hygiene, Command/Query ABI, Timing & Rotational Physics, Deterministic Save-State Serialization).
- **Platinum Quality Engineer**: Procedural C-like C++11 conformity, nesting and control-flow structure (guard clauses vs beneficial nesting), type hygiene, safety checklist, and switch statement optimization.

---

## 1. Executive Summary & Quality Scorecard

| Category | Status | Summary Findings |
|---|---|---|
| **7-Pillar Peripheral Compliance** | **Passed (with minor hardening)** | Full state encapsulation inside `DiskPeripheral_t`, zero non-const globals, deterministic cycle pacing in `think`, complete save-state sizing probe. |
| **Procedural C++11 & Style** | **Conformant** | Strict `snake_case` functions, `PascalCase_t` types, trailing return types (`auto -> type`), zero C++14+ features. |
| **Nesting & Control Flow** | **Good** | Clean use of early-return guard clauses throughout. Opportunities exist to flatten minor nested checks and eliminate redundant bounds checks. |
| **Type Hygiene & Safety** | **Good** | Fixed-width types (`uint8_t`, `uint16_t`, `uint32_t`, `int32_t`) used throughout. All stack arrays modernized to `std::array`. Track buffer converted to RAII `std::vector<uint8_t>`. |
| **Switch Statement Architecture** | **Analyzed** | Softswitch address decoding ($C0n0–$C0nF) analyzed for switch vs. array table dispatch. Sparse command/query switches confirmed optimal. |

---

## 2. Dispatch Architecture & Switch Statement Analysis

`Disk.cpp` contains four `switch` statements across I/O decoding and ABI dispatch:

### A. I/O Softswitch Decoding: `disk_io_read` & `disk_io_write`
- **Current Pattern**:
  - `disk_io_read` implements a 16-case `switch (addr & regs::addr_mask)`.
  - `disk_io_write` implements an almost identical 16-case `switch (addr & regs::addr_mask)`.
- **Architectural Analysis**:
  - Addresses are dense and strictly bounded to 16 contiguous values (`0x0` through `0xF`).
  - *Option 1 (Retain `switch`)*: Modern compilers (GCC/Clang with `-O3`) lower dense 16-case switches into a branchless jump table located in `.rodata`. However, the switch labels and dispatch logic are duplicated verbatim across both functions (~90 lines total).
  - *Option 2 (Function Pointer Dispatch Table)*:
    ```cpp
    using DiskIoHandler_t = auto (*)(void* instance, uint16_t pc, uint16_t addr,
                                     uint8_t is_write, uint8_t val,
                                     uint32_t cycles) -> uint8_t;
    constexpr std::array<DiskIoHandler_t, 16> k_io_dispatch_table = {
        disk_io_control_stepper, disk_io_control_stepper, // $C0n0 - $C0n1
        disk_io_control_stepper, disk_io_control_stepper, // $C0n2 - $C0n3
        disk_io_control_stepper, disk_io_control_stepper, // $C0n4 - $C0n5
        disk_io_control_stepper, disk_io_control_stepper, // $C0n6 - $C0n7
        disk_io_control_motor,   disk_io_control_motor,   // $C0n8 - $C0n9
        disk_io_enable_drive,    disk_io_enable_drive,    // $C0nA - $C0nB
        disk_io_read_write,                               // $C0nC
        disk_io_set_latch,                                // $C0nD
        disk_io_set_read_mode,                            // $C0nE
        disk_io_set_write_mode                            // $C0nF
    };
    ```
  - **Verdict**: A static `constexpr` dispatch table is superior for readability, guarantees $O(1)$ dispatch without compiler heuristic dependency, and eliminates over 60 lines of duplicate switch code.

### B. Command & Query ABI Dispatch: `disk_abi_command` & `disk_abi_query`
- **Current Pattern**:
  - `disk_abi_command` switches over `DiskCmd_e` (`0x0001, 0x0002, 0x0003, 0x0004, 0x0006, 0x1001`).
  - `disk_abi_query` switches over `DiskQuery_t` (`0x0001, 0x0002`).
- **Architectural Analysis**:
  - Command IDs are non-contiguous and sparse.
  - Invocation frequency is low (triggered only by user menu actions or disk insertion/ejection).
  - Memory overhead of an indexed array would require a sparse map or large blank table.
- **Verdict**: Flat `switch` statement with `default: return peripheral_incompatible;` is **ALREADY OPTIMAL**. It is standard, clear, type-safe, and zero-allocation.

---

## 3. Comprehensive Function-by-Function Review (All 44 Functions)

---

### 1. `disk_image_metadata_copy_to_buffer`
- **Signature**: `auto disk_image_metadata_copy_to_buffer(const std::string& src, char* dest, size_t capacity) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Procedural C++11 helper in anonymous namespace. Strict `snake_case` and trailing return type.
- **Nesting**: Guard clause at entry: `if (dest == nullptr || capacity == 0) return;`.
- **Type Hygiene**: Uses `std::min(src.size(), capacity - 1)` with `size_t`. Guarantees null-termination: `dest[copy_len] = '\0'`.
- **Recommendations**: None.

---

### 2. `disk_image_metadata_export_name`
- **Signature**: `auto disk_image_metadata_export_name(const DiskImageMetadata_t& meta, char* dest, size_t capacity) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Clean single-responsibility adapter delegating to `disk_image_metadata_copy_to_buffer`.
- **Nesting**: Flat one-liner.
- **Type Hygiene**: Types and sizes match ABI exports.
- **Recommendations**: None.

---

### 3. `disk_image_metadata_export_path`
- **Signature**: `auto disk_image_metadata_export_path(const DiskImageMetadata_t& meta, char* dest, size_t capacity) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Clean single-responsibility adapter delegating to `disk_image_metadata_copy_to_buffer`.
- **Nesting**: Flat one-liner.
- **Type Hygiene**: Correct.
- **Recommendations**: None.

---

### 4. `is_drive_valid`
- **Signature**: `auto is_drive_valid(int drive_index) -> bool`
- **Status**: **OPTIMAL**
- **Code Quality**: Pure function, zero side effects.
- **Nesting**: Single return expression: `return (drive_index >= 0 && drive_index < disk_drive_count);`.
- **Type Hygiene**: Validates integer bounds against named constant `disk_drive_count = 2`.
- **Recommendations**: Can be marked `constexpr noexcept`.

---

### 5. `get_active_drive`
- **Signature**: `auto get_active_drive(DiskPeripheral_t* dp) -> Disk_t&`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Clean accessor returning reference to active drive.
- **Nesting**: Flat one-liner.
- **Type Hygiene**: Uses `.at(static_cast<size_t>(dp->active_drive_index))`.
- **Findings & Risks**: In the event of memory corruption or uninitialized state, `.at()` throws `std::out_of_range`, which will terminate the application because LinApple compiles with noexcept ABI expectations.
- **Recommendation**: Defensive clamping:
  ```cpp
  auto get_active_drive(DiskPeripheral_t* dp) -> Disk_t& {
    const size_t index = (dp->active_drive_index < disk_drive_count)
                             ? static_cast<size_t>(dp->active_drive_index)
                             : 0;
    return dp->drives[index];
  }
  ```

---

### 6. `notify_status_changed`
- **Signature**: `auto notify_status_changed(const DiskPeripheral_t* dp) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Follows Pillar 4 (Host Boundary Hygiene).
- **Nesting**: Beneficial compound guard: `if (dp != nullptr && dp->host != nullptr && dp->host->NotifyStatusChanged != nullptr)`.
- **Type Hygiene**: Correct.
- **Recommendations**: None.

---

### 7. `notify_activity_changed`
- **Signature**: `auto notify_activity_changed(const DiskPeripheral_t* dp, bool active) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Follows Pillar 4. Forwards drive LED/motor activity to host frontend.
- **Nesting**: Beneficial compound check.
- **Type Hygiene**: `bool active` matches host interface signature.
- **Recommendations**: None.

---

### 8. `is_disk_write_protected`
- **Signature**: `auto is_disk_write_protected(const DiskPeripheral_t* disk_peripheral, int drive_index) -> bool`
- **Status**: **OPTIMAL**
- **Code Quality**: Models the 4 physical layers of Apple II write protection:
  1. Notch/User toggle (`is_user_write_protected`).
  2. Host OS file system permissions (`is_os_read_only`).
  3. Format driver static capabilities (`disk_driver_cap_write`).
  4. Format driver internal container flags (`driver->is_write_protected`).
- **Nesting**: Excellent flattening with sequential early-return guard clauses.
- **Type Hygiene**: Safe casting and bounds checking.
- **Recommendations**: None.

---

### 9. `write_track_to_driver`
- **Signature**: `auto write_track_to_driver(DiskPeripheral_t* disk_peripheral, int drive_index) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Correctly flushes nibble track buffer to the underlying format driver.
- **Nesting**: Guard clauses for drive validity, track bounds, and write protection.
- **Type Hygiene**: Vector `.data()` passed to C driver API.
- **Findings & Risks**: Missing null check for `disk_peripheral` at entry. Calling `write_track_to_driver(nullptr, 0)` causes a segmentation fault on dereference.
- **Recommendation**: Add `if (disk_peripheral == nullptr || !is_drive_valid(drive_index)) return;` at entry.

---

### 10. `read_track_from_driver`
- **Signature**: `auto read_track_from_driver(DiskPeripheral_t* disk_peripheral, int drive_index) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Loads physical bitstream track from format driver into `track_buffer`.
- **Nesting**: Guard clauses for drive validity and track range.
- **Type Hygiene**: Allocates `nibbles_per_track` bytes via `.resize()` on demand.
- **Findings & Risks**:
  1. Missing null check for `disk_peripheral` at entry.
  2. `driver->read_track` returns `loaded_nibbles` via pointer. If a third-party or malformed format driver returns a value greater than `nibbles_per_track`, subsequent reads could overrun.
- **Recommendation**:
  - Add `if (disk_peripheral == nullptr || !is_drive_valid(drive_index)) return;`.
  - Clamp `loaded_nibbles` against `nibbles_per_track`:
    `disk_ptr->nibble_count = std::min<uint32_t>(static_cast<uint32_t>(std::max(0, loaded_nibbles)), static_cast<uint32_t>(nibbles_per_track));`.

---

### 11. `close_format_driver`
- **Signature**: `auto close_format_driver(Disk_t* disk_ptr) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Procedural RAII helper ensuring driver instance cleanup.
- **Nesting**: Guard clause `if (disk_ptr == nullptr || disk_ptr->driver == nullptr) return;`.
- **Type Hygiene**: Safely resets driver pointers to `nullptr` after closing.
- **Recommendations**: None.

---

### 12. `eject_disk_from_drive`
- **Signature**: `auto eject_disk_from_drive(DiskPeripheral_t* disk_peripheral, int drive_index) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Complete physical ejection workflow: flushes dirty track, closes driver, clears host config key, notifies status change, and resets `Disk_t` state.
- **Nesting**: Clean guard clauses at entry.
- **Type Hygiene**: Safe string and configuration operations.
- **Recommendations**: None.

---

### 13. `update_disk_metadata`
- **Signature**: `auto update_disk_metadata(Disk_t* disk_ptr, const char* image_path) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Extracts display name from path and cleans formatting.
- **Nesting**: Guard clause at entry.
- **Type Hygiene**: Modernized with `std::string` and bounds-checked `.at()`.
- **Findings & Risks**: Uses `path_str.find_last_of('/')`. If an image is loaded on a system with Windows backslash separators (`\`), directory components will not be stripped.
- **Recommendation**: Use `path_str.find_last_of("/\\")` to handle both Unix and Windows path separators.

---

### 14. `sync_drive_motor_state`
- **Signature**: `auto sync_drive_motor_state(DiskPeripheral_t* disk_peripheral) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Synchronizes physical motor softswitch state with drive spin-up ticks (`spinup_ticks = 20000`).
- **Nesting**: Guard clause at entry. Only notifies host when spinning transition occurs.
- **Type Hygiene**: Correct boolean states.
- **Recommendations**: None.

---

### 15. `insert_disk_into_drive`
- **Signature**: `auto insert_disk_into_drive(DiskPeripheral_t* disk_peripheral, int drive_index, const char* image_path, bool write_protected, bool create_if_necessary) -> DiskError_e`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Loads floppy disk image via `disk_loader_open()`, updates metadata, and syncs configuration.
- **Nesting**: Guard clauses for drive validity and error return.
- **Type Hygiene**: Strict `DiskError_e` status reporting.
- **Findings & Risks**: Missing defensive checks for `disk_peripheral == nullptr` and `image_path == nullptr`. If `image_path` is null, calling `disk_loader_open` or constructing `std::string` causes undefined behavior.
- **Recommendation**: Add `if (disk_peripheral == nullptr || image_path == nullptr || !is_drive_valid(drive_index)) return disk_err_io;` at entry.

---

### 16. `sync_driver_options`
- **Signature**: `auto sync_driver_options(DiskPeripheral_t* disk_peripheral) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Propagates speed enhancement mode to all mounted format drivers via command ABI.
- **Nesting**: Clean guard clause at entry and loop iteration.
- **Type Hygiene**: Proper `sizeof(uint8_t)` payload.
- **Recommendations**: None.

---

### 17. `disk_io_control_motor`
- **Signature**: `auto disk_io_control_motor(void* instance, uint16_t, uint16_t memory_address, uint8_t, uint8_t, uint32_t) -> uint8_t`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Softswitches `$C088` (motor off) and `$C089` (motor on).
- **Nesting**: Clean guard clause.
- **Type Hygiene**: Memory address bit 0 decode: `(memory_address & 0x01) != 0`.
- **Findings & Risks**: Returns floating bus data via direct `extern` call to `mem_return_random_data(0xFF)`. LinApple standard requires peripherals to avoid undeclared memory core dependencies.
- **Recommendation**: Route floating bus reads through `mem_read_floating_bus(cycles)` or host interface.

---

### 18. `step_drive_head`
- **Signature**: `auto step_drive_head(DiskPeripheral_t* disk_peripheral, int phase_delta) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Accurate physical modeling of Disk II stepper motor (80 half-track phases, 40 tracks, 2 phases per track).
- **Nesting**: Guard clauses for null instance and phase no-op.
- **Type Hygiene**: Safe clamping: `std::max<int32_t>(0, std::min<int32_t>(max_disk_phases - 1, ...))`.
- **Recommendations**: None.

---

### 19. `disk_io_control_stepper`
- **Signature**: `auto disk_io_control_stepper(void* instance, uint16_t, uint16_t memory_address, uint8_t, uint8_t, uint32_t) -> uint8_t`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Softswitches `$C080`–`$C087`. Models 4-phase magnetic coil energizing and detent pulling.
- **Nesting**: Clean guard clause at entry.
- **Type Hygiene**: Bitmask manipulations are properly typed.
- **Findings & Risks**: Line 506 checks `(memory_address == regs::stepper_alt) ? physical::floating_bus : mem_return_random_data(...)` where `stepper_alt = 0xE0`. Returning literal `0xFF` for `$C0E0` is an AppleWin legacy quirk that deviates from physical floating bus bitstream behavior.
- **Recommendation**: Document the legacy quirk and normalize return to standard floating bus generator.

---

### 20. `disk_io_enable_drive`
- **Signature**: `auto disk_io_enable_drive(void* instance, uint16_t, uint16_t memory_address, uint8_t, uint8_t, uint32_t) -> uint8_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Softswitches `$C08A` (Drive 1 select) and `$C08B` (Drive 2 select). Flushes dirty buffer of deselected drive before switching.
- **Nesting**: Guard clauses and clean branch on drive change.
- **Type Hygiene**: Exact typing.
- **Recommendations**: None.

---

### 21. `disk_io_read_write`
- **Signature**: `auto disk_io_read_write(void* instance, uint16_t, uint16_t, uint8_t, uint8_t, uint32_t) -> uint8_t`
- **Status**: **NEEDS OPTIMIZATION**
- **Code Quality**: Softswitch `$C08C` (Q6 strobe data). Hot path executed up to 200,000 times/second.
- **Nesting**: Guard clauses at entry; clean write/read branch.
- **Type Hygiene**: Safe wrapping: `if (++drive.current_byte_pos >= drive.nibble_count) drive.current_byte_pos = 0;`.
- **Findings & Risks**: Lines 566 and 571 use `drive.track_buffer.at(drive.current_byte_pos)`. Because `current_byte_pos` is already explicitly clamped within bounds on lines 558–561, `.at()` introduces redundant branching and exception handling overhead on every single cycle of disk I/O.
- **Recommendation**: Use `drive.track_buffer[drive.current_byte_pos]` for hot-path performance without compromising safety.

---

### 22. `disk_io_set_latch`
- **Signature**: `auto disk_io_set_latch(void* instance, uint16_t, uint16_t, uint8_t is_write, uint8_t data_value, uint32_t) -> uint8_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Softswitch `$C08D` (Q6 shift register / write latch).
- **Nesting**: Clean guard clause.
- **Type Hygiene**: Updates `io_latch` when `is_write != 0`, returns latch value.
- **Recommendations**: None.

---

### 23. `disk_io_set_read_mode`
- **Signature**: `auto disk_io_set_read_mode(void* instance, uint16_t, uint16_t, uint8_t, uint8_t, uint32_t) -> uint8_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Softswitch `$C08E` (Q7 read mode).
- **Nesting**: Clean guard clause.
- **Type Hygiene**: Returns `0x80` if write protected, `0x00` if write enabled (authentic physical Disk II behavior).
- **Recommendations**: None.

---

### 24. `disk_io_set_write_mode`
- **Signature**: `auto disk_io_set_write_mode(void* instance, uint16_t, uint16_t, uint8_t, uint8_t, uint32_t) -> uint8_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Softswitch `$C08F` (Q7 write mode). Activates write mode and updates activity indicators.
- **Nesting**: Clean guard clause.
- **Type Hygiene**: Correct.
- **Recommendations**: None.

---

### 25. `update_drive_physics`
- **Signature**: `auto update_drive_physics(DiskPeripheral_t* disk_peripheral, Disk_t* disk_ptr, uint32_t spin_ticks, uint32_t rotation_ticks) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Stepped rotational and motor spindown physics. Requests host precise timing when motor is spinning in authentic speed mode.
- **Nesting**: Good use of early return when inactive or speed enhanced.
- **Type Hygiene**: Safe non-zero modulo prevents division by zero: `(disk_ptr->nibble_count != 0 ? disk_ptr->nibble_count : 1)`.
- **Findings & Risks**: Missing defensive check `if (disk_peripheral == nullptr || disk_ptr == nullptr) return;` at entry.
- **Recommendation**: Add null checks at entry.

---

### 26. `update_physical_disk_state`
- **Signature**: `auto update_physical_disk_state(DiskPeripheral_t* disk_peripheral, uint32_t elapsed_cycles) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Accumulates 6502 clock cycles and updates drive physics.
- **Nesting**: Flat loop iterating over attached drives.
- **Type Hygiene**: Proper bitshifts and bitmasks (`spin_cycle_mask`, `rotation_cycle_mask`).
- **Findings & Risks**: Missing defensive check `if (disk_peripheral == nullptr) return;`.
- **Recommendation**: Add null check at entry.

---

### 27. `swap_drives`
- **Signature**: `auto swap_drives(DiskPeripheral_t* disk_peripheral) -> bool`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Swaps physical drives (Drive 1 ⇆ Drive 2). Refuses to swap if either drive motor is spinning.
- **Nesting**: Guard clauses for null instance and spinning drive.
- **Type Hygiene**: `std::swap(dp->drives[0], dp->drives[1])`.
- **Findings & Risks**: Swaps drive structs in memory and notifies status change, but does not update host configuration keys `Disk Image 1` and `Disk Image 2`. If the emulator is restarted, the swap is lost.
- **Recommendation**: Write swapped paths back to host configuration via `host->SetConfig`.

---

### 28. `initialize_peripheral`
- **Signature**: `auto initialize_peripheral(DiskPeripheral_t* disk_peripheral) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: Flushes dirty tracks, resets controller flip-flops and stepping masks, and leaves mounted disk media intact.
- **Nesting**: Clean guard clause.
- **Type Hygiene**: Proper reset of scalar state.
- **Recommendations**: None.

---

### 29. `get_peripheral_status`
- **Signature**: `auto get_peripheral_status(DiskPeripheral_t* disk_peripheral, DiskStatus_t* status) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Exports drive status, error codes, and paths to frontend.
- **Nesting**: Guard clause at entry.
- **Type Hygiene**: Correct.
- **Findings & Risks**: Does not explicitly zero `status->padding`, potentially leaking uninitialized stack bytes across the C-ABI boundary.
- **Recommendation**: Clear status struct with `*status = DiskStatus_t{};` before populating fields.

---

### 30. `disk_io_read`
- **Signature**: `auto disk_io_read(void* instance, uint16_t program_counter, uint16_t memory_address, uint8_t is_write, uint8_t, uint32_t remaining_cycles) -> uint8_t`
- **Status**: **CANDIDATE FOR OPTIMIZATION**
- **Code Quality**: Primary I/O read callback registered with `HostInterface_t`.
- **Nesting**: Guard clause `if (instance == nullptr || is_write != 0) return mem_return_random_data(...);`.
- **Switch Analysis**: 16-case switch statement.
- **Recommendation**: Refactor to static `constexpr` dispatch table (as detailed in Section 2) to eliminate code duplication and guarantee $O(1)$ dispatch.

---

### 31. `disk_io_write`
- **Signature**: `auto disk_io_write(void* instance, uint16_t program_counter, uint16_t memory_address, uint8_t is_write, uint8_t data_value, uint32_t remaining_cycles) -> uint8_t`
- **Status**: **CANDIDATE FOR OPTIMIZATION**
- **Code Quality**: Primary I/O write callback registered with `HostInterface_t`.
- **Nesting**: Guard clause `if (instance == nullptr || is_write == 0) return 0;`.
- **Switch Analysis**: 16-case switch statement duplicating `disk_io_read`.
- **Recommendation**: Unify with the static dispatch table.

---

### 32. `cmd_handle_insert`
- **Signature**: `auto cmd_handle_insert(DiskPeripheral_t* dp, const void* data, size_t size) -> PeripheralStatus_t`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Command handler for `disk_cmd_insert`.
- **Nesting**: Guard clause validates payload size and checks path null-termination via `memchr`.
- **Type Hygiene**: Safe casting to `const DiskInsertCmd_t*`.
- **Findings & Risks**:
  1. Missing null check `if (dp == nullptr) return peripheral_error;`.
  2. Ignores return error from `insert_disk_into_drive()`, returning `peripheral_ok` even if image loading failed.
- **Recommendation**: Add `dp == nullptr` check, and return `peripheral_error` if `insert_disk_into_drive()` fails.

---

### 33. `cmd_handle_eject`
- **Signature**: `auto cmd_handle_eject(DiskPeripheral_t* dp, const void* data, size_t size) -> PeripheralStatus_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Validates payload size and drive index, delegates to `eject_disk_from_drive()`, and returns `peripheral_ok`.
- **Nesting**: Clean guard clauses.
- **Type Hygiene**: Safe.
- **Recommendations**: None.

---

### 34. `cmd_handle_set_protect`
- **Signature**: `auto cmd_handle_set_protect(DiskPeripheral_t* dp, const void* data, size_t size) -> PeripheralStatus_t`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Command handler for write protect notch manipulation.
- **Nesting**: Guard clauses for size and drive validity.
- **Findings & Risks**: Missing defensive check `if (dp == nullptr) return peripheral_error;`.
- **Recommendation**: Add null check for `dp`.

---

### 35. `cmd_handle_set_speed`
- **Signature**: `auto cmd_handle_set_speed(DiskPeripheral_t* dp, const void* data, size_t size) -> PeripheralStatus_t`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Command handler for fast vs. authentic disk speed.
- **Nesting**: Guard clause for payload size.
- **Findings & Risks**: Missing defensive check `if (dp == nullptr) return peripheral_error;`.
- **Recommendation**: Add null check for `dp`.

---

### 36. `disk_abi_init`
- **Signature**: `auto disk_abi_init(int slot, HostInterface_t* host) -> void*`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: C-ABI lifecycle initialization callback. Registers format drivers, loads initial configuration, registers ROM and I/O callbacks.
- **Nesting**: Guard clauses for null host and missing registration callbacks.
- **Type Hygiene**: Modernized with `std::array` for configuration buffers.
- **Findings & Risks**:
  1. Concurrency: If multiple Disk II cards are instantiated (e.g. Slot 6 and Slot 5), each call invokes `disk_loader_register()`, creating duplicate driver entries in the global registry.
  2. Multi-Card Configuration: Reads keys `Disk Image 1` and `Disk Image 2` regardless of slot.
- **Recommendation**: Deduplicate driver registrations in `disk_loader_register`, and prefix config keys with slot number when `slot != 6`.

---

### 37. `disk_abi_reset`
- **Signature**: `auto disk_abi_reset(void* instance) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: C-ABI reset callback.
- **Nesting**: Guard clause `if (instance == nullptr) return;`.
- **Type Hygiene**: Safe pointer cast to `DiskPeripheral_t*`.
- **Recommendations**: None.

---

### 38. `disk_abi_shutdown`
- **Signature**: `auto disk_abi_shutdown(void* instance) -> void`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: C-ABI destruction callback. Ejects drives and frees `DiskPeripheral_t`.
- **Nesting**: Guard clause at entry.
- **Type Hygiene**: RAII ownership transfer to `std::unique_ptr`.
- **Findings & Risks**: Calls `disk_loader_shutdown()` *before* calling `eject_disk_from_drive()`. If the driver registry refcount drops to zero, format driver pointers are cleared before the drives are flushed and closed.
- **Recommendation**: Eject all drives first, then invoke `disk_loader_shutdown()`.

---

### 39. `disk_abi_think`
- **Signature**: `auto disk_abi_think(void* instance, uint32_t elapsed_cycles) -> void`
- **Status**: **OPTIMAL**
- **Code Quality**: C-ABI cycle stepping callback. Drives motor and rotational physics.
- **Nesting**: Guard clause `if (instance == nullptr || elapsed_cycles == 0) return;`.
- **Type Hygiene**: Correct cycle typing.
- **Recommendations**: None.

---

### 40. `disk_abi_command`
- **Signature**: `auto disk_abi_command(void* instance, uint32_t cmd, const void* data, size_t size) -> PeripheralStatus_t`
- **Status**: **OPTIMAL**
- **Code Quality**: C-ABI command dispatch point.
- **Nesting**: Guard clause at entry.
- **Switch Analysis**: Flat switch on sparse `DiskCmd_e` constants with `default: break;` returning `peripheral_incompatible`. This is the optimal pattern for sparse ABI commands.
- **Recommendations**: None.

---

### 41. `disk_abi_query`
- **Signature**: `auto disk_abi_query(void* instance, uint32_t cmd, void* data, size_t* size) -> PeripheralStatus_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Adheres strictly to the two-pass sizing probe protocol (`data == nullptr`, `*size < required_size`, and population).
- **Nesting**: Guard clause at entry.
- **Switch Analysis**: Flat switch on sparse query constants. Optimal.
- **Type Hygiene**: Uses `supported_extensions_cap = 256` and exact struct sizes.
- **Recommendations**: None.

---

### 42. `disk_abi_save_state`
- **Signature**: `auto disk_abi_save_state(void* instance, void* buffer, size_t* size) -> PeripheralStatus_t`
- **Status**: **NEEDS HARDENING**
- **Code Quality**: Deterministic serialization of controller and drive state.
- **Nesting**: Guard clauses for null pointers and sizing probes.
- **Type Hygiene**: Uses value-initialization `*s = DiskSavedState_t{};`.
- **Findings & Risks**: Line 1118 copies track buffer: `if (!d.track_buffer.empty()) std::copy_n(d.track_buffer.data(), nibbles_per_track, ds.track_buffer);`. If for any reason `d.track_buffer.size() < nibbles_per_track`, this will read past the buffer.
- **Recommendation**: Bound copy length with `std::min(d.track_buffer.size(), static_cast<size_t>(nibbles_per_track))`.

---

### 43. `disk_abi_load_state`
- **Signature**: `auto disk_abi_load_state(void* instance, const void* buffer, size_t size) -> PeripheralStatus_t`
- **Status**: **OPTIMAL**
- **Code Quality**: Save-state deserialization. Enforces version checks, finds null-terminated path safely with `std::find`, and rigorously clamps all restored fields (track, phase, nibble count, current position) to physical hardware boundaries.
- **Nesting**: Clean guard clauses.
- **Type Hygiene**: Correctly logs clamped out-of-bounds state to host log interface.
- **Recommendations**: None.

---

### 44. `disk_get_descriptor`
- **Signature**: `auto disk_get_descriptor() -> Peripheral_t*`
- **Status**: **OPTIMAL**
- **Code Quality**: Identity header export function.
- **Nesting**: Flat one-liner returning `&g_disk_peripheral`.
- **Type Hygiene**: Descriptor structure specifies expansion slot compatibility (`PERIPHERAL_MASK_EXPANSION = 0xFE`), `default_slot = 6`, and C-ABI function pointers.
- **Recommendations**: None.

---

## 4. Prioritized Actionable Improvements & Implementation Status

| Priority | Function(s) | Improvement Description | Implementation Status |
|---|---|---|---|
| **High** | `disk_io_read_write` | Replace `track_buffer.at(...)` with direct indexing `track_buffer[...]` in hot path to eliminate redundant bounds-check branching and exception overhead. | **Implemented & Verified** |
| **High** | `disk_abi_shutdown` | Invert cleanup order: call `eject_disk_from_drive()` for all drives *before* invoking `disk_loader_shutdown()`. | **Implemented & Verified** |
| **High** | `cmd_handle_insert` | Add null check for `dp` and propagate error status from `insert_disk_into_drive()` instead of unconditionally returning `peripheral_ok`. | **Implemented & Verified** |
| **Medium** | `disk_io_read`, `disk_io_write` | Convert duplicate 16-case switches into a unified `constexpr std::array<DiskIoHandler_t, 16>` dispatch table for $O(1)$ dispatch and clean code reuse. | **Implemented & Verified** |
| **Medium** | `write_track_to_driver`, `read_track_from_driver`, `insert_disk_into_drive`, `update_drive_physics`, `update_physical_disk_state`, `cmd_handle_set_protect`, `cmd_handle_set_speed` | Add missing defensive null pointer guard clauses at function entries. | **Implemented & Verified** |
| **Medium** | `disk_abi_save_state` | Bound `std::copy_n` with `std::min(d.track_buffer.size(), static_cast<size_t>(nibbles_per_track))` to prevent potential buffer overread. | **Implemented & Verified** |
| **Medium** | `update_disk_metadata` | Use `path_str.find_last_of("/\\")` to support both Unix and Windows path separators. | **Implemented & Verified** |
| **Medium** | `disk_io_control_stepper` | Eliminate artificial legacy `$C0E0` quirk (`stepper_alt`) returning static 0xFF; return authentic `mem_return_random_data(physical::floating_bus)` unconditionally. | **Implemented & Verified** |
| **Low** | `get_peripheral_status` | Zero status struct with `*status = DiskStatus_t{};` before populating to avoid leaking uninitialized padding bytes across the ABI. | **Implemented & Verified** |
| **Low** | `swap_drives` | Persist drive path swaps to host configuration keys via `host->SetConfig` so drive swaps survive emulator restarts. | **Implemented & Verified** |
| **Low** | `is_drive_valid` | Harden as `constexpr noexcept`. | **Implemented & Verified** |
| **Low** | `get_active_drive` | Add defensive index clamping before indexing drive array. | **Implemented & Verified** |

---

## 5. Verification & Test Results

All 44 functions have been audited, modernized, and verified against the LinApple coding and safety conventions:

1. **Compilation**: Clean build with zero warnings or errors.
2. **CTest Suite**: 20/20 disk and storage test suites passed (100% pass rate):
   - `test-disk-abi`
   - `test-disk-gcr`
   - `test-disk-drivers`
   - `test-disk-smoke`
   - `test-disk-config-startup`
   - `test-disk-config-runtime-insert`
   - `test-disk-config-runtime-eject`
   - `test-disk-ui`
   - `test-tui-disk-select`
   - `test-disk-motor`
   - `test-disk-prot`
   - `test-disk-woz`
   - `test-disk-savestate`
   - `test-disk-errors`
   - `test-disk-stepper`
   - `test-disk-io`
   - `test-disk-compression`
   - `test-harddisk-peripheral`
   - `test-harddisk-exhaustive`
   - `test-formal-disk`
3. **Static Analysis**: `clang-tidy src/apple2/peripherals/disk/Disk.cpp -p build` executed cleanly with 0 unsuppressed violations.
4. **Code Formatting**: Fully formatted with `clang-format`.

