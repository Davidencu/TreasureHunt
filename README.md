# Treasure Hunt Management System

A comprehensive treasure hunt management system implemented in C, featuring multi-process communication, file I/O operations, and signal handling. This system allows users to create hunts, manage treasures, and calculate scores for participating users.

## Overview

The Treasure Hunt Management System is a command-line based application that organizes and manages treasure hunts with multiple treasures per hunt. Each treasure contains metadata including an ID, username of the creator, a clue, geographic coordinates, and a point value. The system uses a hub-and-spoke architecture with a central manager process coordinating operations across multiple child processes.

## Project Structure

### Executables

The project compiles into three main executables:

#### 1. **treasure_manager** (`treasure_manager.c`)
The core treasure management utility that handles all data operations on treasures and hunts.

**Supported Commands:**
- `add <hunt_id>` - Add a new treasure to a hunt. If the hunt doesn't exist, it will be created automatically.
- `list <hunt_id>` - List all treasures within a specific hunt, including file metadata and modification times.
- `view <hunt_id> <treasure_id>` - Display details of a specific treasure by ID.
- `remove_treasure <hunt_id> <treasure_id>` - Remove a specific treasure from a hunt.
- `remove_hunt <hunt_id>` - Delete an entire hunt directory and its contents.
- `list_hunts` - Display all available hunts with their treasure counts.

**Treasure Structure:**
```c
typedef struct treasure {
    char id[8];              // Unique treasure identifier
    char username[16];       // Creator's username
    char clue[64];           // Location clue
    coordinates c;           // GPS coordinates (x, y as floats)
    int value;               // Point value for scoring
} treasure;
```

**Data Storage:**
- Each hunt is stored in a directory named with the hunt ID
- Treasures are stored in binary format in `<hunt_id>/file.bin`
- A symbolic link `logged_hunt-<hunt_id>` points to a log file that tracks all operations on that hunt

#### 2. **treasure_hub** (`treasure_hub.c`)
The main user interface and process orchestrator. Manages the interactive command loop and controls the monitor process.

**User Commands:**
- `start_monitor` - Launch a background monitor process to handle operations.
- `list_hunts` - List all available hunts.
- `list_treasures <hunt_id>` - Display all treasures in a hunt.
- `view_treasure <hunt_id> <treasure_id>` - View a specific treasure.
- `stop_monitor` - Stop the monitor process (with 15-second sleep before termination).
- `calculate_scores <hunt_id>` - Calculate and display scores for all users who found treasures in a hunt.
- `exit` - Exit the application.

**Key Features:**
- Implements a **monitor process** that handles background operations via signal handling (SIGUSR1, SIGUSR2)
- Uses **inter-process communication (IPC)** via pipes to transmit command results
- Tracks monitor state (running/sleeping) and prevents invalid commands
- Creates `cmd.txt` file to communicate commands to the monitor process
- Validates user input and provides error messaging

#### 3. **calculate_scores** (`calculate_scores.c`)
Computes and displays user scores based on treasure values found in a specific hunt.

**Functionality:**
- Takes a hunt ID as a command-line argument
- Reads the binary treasure file from `<hunt_id>/file.bin`
- Aggregates treasure values by username
- Outputs results in CSV format: `username,total_score`
- Handles up to 10 unique users per hunt

## Build & Compilation

### Makefile

The project includes a `makefile` that automates compilation:

```bash
make           # Compile all three executables
make clean     # Remove all compiled binaries
```

**Compiler Configuration:**
- Compiler: `gcc`
- Flags: `-Wall` (enable all commonly used compiler warnings)

**Generated Binaries:**
- `treasure_manager` - Compiled from `treasure_manager.c`
- `treasure_hub` - Compiled from `treasure_hub.c`
- `calculate_scores` - Compiled from `calculate_scores.c`

## Usage Guide

### Basic Workflow

1. **Start the system:**
   ```bash
   make clean    # Clean any existing binaries
   make          # Compile all programs
   ./treasure_hub # Launch the interactive interface
   ```

2. **Initialize the monitor:**
   ```
   Enter command: start_monitor
   ```

3. **Create and manage hunts:**
   ```
   Enter command: add Hunt001
   Treasure ID: T001
   Username: alice
   Clue: Under the old oak tree
   Coordinate x: 40.7128
   Coordinate y: -74.0060
   Value: 100
   ```

4. **View hunt contents:**
   ```
   Enter command: list_treasures Hunt001
   ```

5. **Calculate scores for a hunt:**
   ```
   Enter command: calculate_scores Hunt001
   Hunt001
   alice,100
   bob,250
   ```

6. **Stop the monitor and exit:**
   ```
   Enter command: stop_monitor
   Enter command: exit
   ```

### Example Session

```bash
$ ./treasure_hub
Enter command: start_monitor
Enter command: list_treasures Hunt001
Hunt name: Hunt001
File size: 272 bytes
Last file modification: Wed Mar 12 14:23:45 2026

Treasure 0: T001 - alice - Under the old oak tree - 40.71 - -74.01 - 100
Treasure 1: T002 - bob - By the fountain - 40.75 - -74.00 - 150

Enter command: calculate_scores Hunt001
Hunt001
alice,100
bob,150

Enter command: stop_monitor
Monitor exited with status 0
Enter command: exit
```

## Process Architecture

### Multi-Process Design

The system uses a multi-process model to isolate operations:

1. **Main Process (treasure_hub)**
   - Handles user input in an interactive loop
   - Creates and manages the monitor process
   - Communicates with monitor via SIGUSR1, SIGUSR2 signals
   - Reads results from monitor via pipes

2. **Monitor Process**
   - Runs in the background after `start_monitor`
   - Waits for SIGUSR1 signal to perform operations
   - Forks child processes to execute `treasure_manager` commands
   - Returns results via pipe to main process
   - Can enter sleep mode (15 seconds) via `stop_monitor`

3. **Worker Processes**
   - Created by monitor to execute specific commands
   - Each worker runs an instance of `treasure_manager` or `calculate_scores`
   - Communicates results through standard output/pipes

### Signal Handling

| Signal | Handler | Purpose |
|--------|---------|---------|
| SIGUSR1 | `handle()` (monitor) | Process command from main process |
| SIGUSR1 | `child_is_sleeping()` (main) | Monitor entering sleep mode |
| SIGUSR2 | `child_woke_up()` (main) | Monitor exiting sleep mode |
| SIGCHLD | `monitor_terminated()` (main) | Monitor process terminated |

### Inter-Process Communication (IPC)

- **Pipes:** Used to transmit command results from monitor back to main process
- **Command File:** `cmd.txt` - Temporary file used to pass command arguments to monitor
- **Signals:** Used for synchronization and state notifications between processes

## File System Structure

After running the application, the following directory structure is created:

```
.
├── treasure_hub
├── treasure_manager
├── calculate_scores
├── cmd.txt (created during operation)
├── Hunt001/
│   ├── file.bin (binary treasure storage)
│   └── logged_hunt.txt (operation log)
├── Hunt002/
│   ├── file.bin
│   └── logged_hunt.txt
├── logged_hunt-Hunt001 (symbolic link → Hunt001/logged_hunt.txt)
├── logged_hunt-Hunt002 (symbolic link → Hunt002/logged_hunt.txt)
└── ...
```

## Data Formats

### Binary Treasure File (`file.bin`)

Treasures are stored as binary structures (one treasure per record):

```
[treasure struct][treasure struct][treasure struct]...
Each record: 8 + 16 + 64 + 8 + 4 = 100 bytes
```

### Log File (`logged_hunt.txt`)

Text-based operation log with timestamped entries:

```
add Hunt001
list Hunt001
view Hunt001 T001
remove_treasure Hunt001 T001
```

## Constraints & Limitations

- Maximum treasures per hunt: 100
- Maximum hunts: 30
- Maximum users for score calculation: 10
- Hunt ID max length: 12 characters
- Treasure ID max length: 8 characters
- Username max length: 16 characters
- Clue max length: 64 characters
- Monitor sleep duration: 15 seconds (fixed)

## Error Handling

The system implements comprehensive error handling with `perror()` calls for system-level errors. Common error scenarios:

- Invalid hunt or treasure IDs
- File I/O failures (permission denied, disk full, etc.)
- Process creation failures
- Signal handling failures
- Command parsing errors

## Compilation Requirements

- GCC (GNU C Compiler)
- POSIX-compliant system (Linux, BSD, macOS)
- Standard C libraries: stdio.h, stdlib.h, string.h, unistd.h, fcntl.h
- POSIX APIs: dirent.h, signal.h, sys/stat.h, sys/wait.h

## Future Enhancements

Potential improvements:
- Persistent data validation and recovery
- User authentication and permissions
- Geographic coordinate validation
- Score multipliers and difficulty levels
- Web-based interface
- Concurrent hunt support with locking mechanisms
- Treasure visibility/hints system