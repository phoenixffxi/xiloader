# xiloader - Comprehensive AI Reference Guide

This document contains detailed technical analysis of the xiloader project for future AI reference.

## Project Overview

**xiloader** is a C++ boot loader for Final Fantasy XI that bypasses PlayOnline authentication to connect to private FFXI servers.

- **Original Authors**: DarkStar Dev Teams (2010-2015)
- **Current Maintainer**: LandSandBoat Dev Teams (2021-present)
- **License**: GNU GPL v3
- **Repository**: https://github.com/LandSandBoat/xiloader
- **Purpose**: Launch FFXI client without PlayOnline to connect to private servers

## Build System & Requirements

### Critical Build Requirements
- **ONLY 32-bit builds supported** - 64-bit will fail intentionally
- Visual Studio 2022 C++ Redistributable required
- CMake 3.15 minimum
- C++17 standard
- Windows only (uses Windows COM, registry, and DLL injection)

### Build Process
```bash
mkdir build
cmake -S . -B build -A Win32  # -A Win32 is MANDATORY
cmake --build build
```

### Dependencies (via CPM)
- **detours**: Microsoft Detours library for DLL hooking
- **argparse**: Command-line argument parsing
- **mbedtls**: SSL/TLS cryptographic library for secure connections
- **System Libraries**: crypt32, psapi, ws2_32, iphlpapi

## Architecture & Technical Details

### Core Components

1. **main.cpp**: Entry point, argument parsing, COM initialization, game launch
2. **network.cpp/h**: Socket management, proxy servers, SSL connections
3. **functions.cpp/h**: Pattern scanning, registry access, helper utilities  
4. **console.cpp/h**: Colored console output with timestamps
5. **defines.h**: Constants, COM GUIDs, language enumerations

### Network Architecture

**Port Configuration (Defaults)**:
- Auth Server: 54231
- Login Data: 54230  
- Login View: 54001
- Game Server: 51220

**Network Flow**:
1. Detours winsock functions (gethostbyname, send, recv, connect)
2. Creates local proxy servers on configured ports
3. Intercepts FFXI network traffic and redirects to private server
4. Handles XIFF protocol commands for lobby communication
5. Uses mbedtls for SSL/TLS authentication only (Windows cert store provides CA certificates)

### Memory Patching & DLL Injection

**Pattern Scanning Targets**:
- `polcore.dll` / `polcoreeu.dll`: INET mutex and POL connection functions
- `FFXiMain.dll`: Hairpin NAT fix locations

**Key Functions Located**:
- INET Mutex: Pattern `\x8B\x56\x2C\x8B\x46\x28\x8B\x4E\x24\x52\x50\x51`
- POL Connection: Pattern `\x81\xC6\x38\x03\x00\x00\x83\xC4\x04\x81\xFE`

### COM Integration

**PlayOnline COM Objects**:
- POLCoreCom: Different CLSIDs for JP/US/EU
- FFXiEntry: Single CLSID for game launching

**Language Support**:
- Japanese (0): `polcore.dll`
- English (1): `polcore.dll`  
- European (2): `polcoreeu.dll`

## XIFF Protocol Commands

The loader handles these lobby protocol commands:
- `0x07`: Login to character with account/character ID
- `0x14`: Character deletion request
- `0x1F`: Character list request
- `0x21`: Character creation notification (client-side)
- `0x22`: Character creation request (server-side)
- `0x24`: Server name request
- `0x26`: Version information exchange
- `0x28`: Character rename information (GM function)
- `0x2B`: GM character world move

## Command Line Usage

### Primary Options
```bash
xiloader --server <address> --user <username> --pass <password>
```

### All Available Arguments
- `--server`: Server address to connect to
- `--user/--username`: Account username
- `--pass/--password`: Account password  
- `--email`: Email (currently unused)
- `--serverport`: Lobby server port (default: 51220)
- `--dataport`: Login data port (default: 54230)
- `--viewport`: Login view port (default: 54001)
- `--authport`: Login auth port (default: 54231)
- `--lang`: Language JP/US/EU or 0/1/2
- `--hairpin`: Enable hairpin NAT fix for local servers
- `--hide`: Hide console window after FFXI starts

### Helper Tools

**xi_checker.bat**:
- Locates PlayOnline installation via registry
- Verifies DirectPlay is installed
- Launches xiloader with proper paths
- Usage: `xi_checker.bat <server_address>`

## Development Configuration

### Code Style
- Uses clang-format with WebKit base + Allman braces
- 4-space indentation, no tabs
- Column limit disabled for flexibility
- Namespace indentation enabled

### Git Configuration
**Ignored Files**:
- Build directories: `Debug/`, `Release/`, `bin/`, `build/`
- IDE files: `.vs/`, `.vscode/`, `*.user`
- Compiled outputs: `*.exe`, `*.pdb`, `*.ilk`, `*.exe.manifest`

## Security & SSL Implementation

**Important**: SSL/TLS is **only used for authentication**, not for main game traffic.

### Certificate Handling
- Uses **Windows Certificate Store APIs** to extract root certificates
- `CertOpenStore` and `CertEnumCertificatesInStore` extract system certificates
- Certificates are **converted from Windows format to mbedtls format** using `mbedtls_x509_crt_parse`
- mbedtls then uses these certificates for SSL verification during authentication only

### SSL State Management (Auth Server Only)
Global SSL state includes:
- mbedtls_net_context server_fd
- mbedtls_ssl_context ssl  
- mbedtls_ssl_config conf
- mbedtls_x509_crt cacert
- mbedtls_entropy_context entropy
- mbedtls_ctr_drbg_context ctr_drbg

### Network Security Architecture
- **Auth server communication**: SSL/TLS encrypted using mbedtls
- **Login server communication**: Plain TCP sockets
- **Game server communication**: Plain TCP sockets  
- **Lobby proxy servers**: Plain TCP sockets

## Hairpin NAT Fix

For local server deployments that are publicly exposed:

**Implementation**:
1. Scans FFXiMain.dll for hairpin locations
2. Creates code cave with `HairpinFixCave` function
3. Patches memory with JMP instructions
4. Redirects server address resolution

**Pattern Targets**:
- Main hairpin: `\x8B\x82\xFF\xFF\xFF\xFF\x89\x02\x8B\x0D`
- Zone change: `\x8B\x0D\xFF\xFF\xFF\xFF\x89\x01\x8B\x46`

## Registry Integration

**PlayOnline Registry Keys**:
- US: `HKEY_LOCAL_MACHINE\SOFTWARE\PlayOnlineUS`
- EU: `HKEY_LOCAL_MACHINE\SOFTWARE\PlayOnlineEU` 
- JP: `HKEY_LOCAL_MACHINE\SOFTWARE\PlayOnlineJP`
- Wow64: Same paths under `SOFTWARE\Wow6432Node\`

**Registry Values**:
- InstallFolder: PlayOnline installation directory
- Language settings and regional configurations

## Version Information

Current version: 1.1.5 (must match server expectations)
Version format: x.x.x with single digit components
Update locations: globals in main.cpp and xiloader.rc.in

## Development Notes

### Critical Implementation Details
1. Only 32-bit compilation supported - enforced at CMake level
2. Requires specific VC2022 redistributable for end users
3. Pattern scanning is version-dependent and may break with game updates
4. COM object instantiation requires proper registry configuration
5. Memory patching uses naked assembly functions for hairpin fix

### Maintenance Considerations  
- Pattern signatures may need updates with FFXI client patches
- Registry keys vary by PlayOnline region/language
- SSL certificate handling requires Windows-specific APIs
- Thread management for proxy servers needs careful cleanup

---
*Generated by AI analysis for xiloader project reference*
