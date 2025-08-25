# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Commands for Development

### Building PowerShell

**Primary build system**: Use PowerShell's `build.psm1` module functions.

```powershell
# Import build module
Import-Module ./build.psm1

# Build debug version (default)
Start-PSBuild

# Build release version
Start-PSBuild -Configuration Release

# Build for specific runtime
Start-PSBuild -Runtime linux-x64
Start-PSBuild -Runtime win7-x64
Start-PSBuild -Runtime osx-x64

# Clean build
Start-PSBuild -Clean

# Quick build of System.Management.Automation only
Start-PSBuild -SMAOnly

# Build with resource generation and type generation
Start-PSBuild -ResGen -TypeGen
```

### Testing

```powershell
# Run Pester tests (PowerShell tests)
Start-PSPester

# Run specific test file
Start-PSPester -Path "./test/powershell/engine"

# Run tests with specific tags
Start-PSPester -Tag @("CI","Feature")

# Run xUnit tests (C# unit tests) 
Start-PSxUnit

# Build and publish test tools
Publish-PSTestTools
```

### Development Setup

```powershell
# Bootstrap development environment (install .NET SDK and dependencies)
Start-PSBootstrap

# Install dependencies only
Start-PSBootstrap -Scenario DotNet

# Restore NuGet packages
Restore-PSPackage -Force
```

### Running a Single Test

```powershell
# PowerShell/Pester tests - run specific describe block
Start-PSPester -Path "./test/powershell/engine/basic" -Tag "CI"

# xUnit tests - use dotnet CLI directly from test/xUnit directory
dotnet test --filter "DisplayName~TestName"
```

## High-Level Architecture

### Core Components

**System.Management.Automation** (`src/System.Management.Automation/`): The PowerShell engine containing the core automation framework. This is the fundamental component that everything else builds upon.

**Host Applications**: Multiple entry points for different platforms:
- `src/powershell-win-core/`: Windows PowerShell host (`pwsh.exe`)  
- `src/powershell-unix/`: Unix/Linux/macOS PowerShell host (`pwsh`)
- `src/Microsoft.PowerShell.ConsoleHost/`: Console host implementation

**Command Modules** (`src/Microsoft.PowerShell.*`): Each major functional area has its own module:
- `Commands.Management`: File system, process, service management
- `Commands.Utility`: Data manipulation, formatting, basic utilities  
- `Commands.Diagnostics`: Event logs, performance counters
- `Security`: Execution policy, security cmdlets
- `Microsoft.WSMan.Management`: WinRM and remoting

**Core Engine Classes**:
- `ExecutionContext`: Central execution context and engine state
- `InitialSessionState`: Defines what's available in a PowerShell session
- `PowerShell`: Main API class for creating and executing PowerShell commands
- `ScriptBlock`: Represents compiled PowerShell scripts
- `SessionState`: Manages variables, functions, aliases, providers

### Module System Architecture

PowerShell uses two main module loading approaches:

1. **Engine Modules** (`EngineModules` in `InitialSessionState.cs`): Core modules like `Microsoft.PowerShell.Utility`, `Microsoft.PowerShell.Management` that provide fundamental cmdlets.

2. **Binary Modules**: C# assemblies that expose cmdlets through `[Cmdlet]` attributes in the `Microsoft.PowerShell.Commands.*` namespaces.

The `InitialSessionState` class orchestrates loading these components into a PowerShell session, with separate mappings between public module names and their underlying implementation assemblies.

### Build System Architecture

The build is orchestrated by `build.psm1` PowerShell module:

1. **Dependency Management**: Uses .NET SDK specified in `global.json` and `DotnetRuntimeMetadata.json`
2. **Multi-Platform**: Supports different runtimes (Windows, Linux, macOS, ARM)
3. **Modular**: Each `src/` component is a separate C# project (`.csproj`)
4. **Resource Generation**: Generates C# code from `.resx` files (`Start-ResGen`)
5. **Type Generation**: Creates type catalogs for runtime optimization (`Start-TypeGen`)

### Testing Architecture

**Two Testing Frameworks**:
1. **Pester**: PowerShell-based tests in `test/powershell/` for cmdlet functionality and integration testing
2. **xUnit**: C# unit tests in `test/xUnit/` for engine component testing

**Test Organization**:
- Tests are tagged with `CI`, `FEATURE`, `SCENARIO` for different test runs
- Platform-specific tests use tags like `RequireAdminOnWindows`, `RequireSudoOnUnix`
- Test tools in `test/tools/` provide supporting executables and utilities

## Development Patterns

### PowerShell Cmdlet Development

When creating new cmdlets, follow the established patterns:

- Inherit from `PSCmdlet` or `Cmdlet` base classes
- Use `[Cmdlet]` attribute with approved Verb-Noun naming
- Place in appropriate `Microsoft.PowerShell.Commands.*` namespace
- Add to corresponding engine module in `InitialSessionState.cs`

### Error Handling

PowerShell has sophisticated error handling:
- Use `ErrorRecord` objects for structured error information  
- Cmdlets should call `WriteError()` rather than throwing exceptions for non-terminating errors
- The `$Error` automatic variable tracks all errors in the session

### Session State Management

Understanding session state is crucial:
- Variables, functions, aliases are managed by `SessionStateInternal`
- Scoping follows PowerShell rules (Global, Script, Local, Private)
- Modules get their own session state scope
- `ExecutionContext` maintains the current execution environment

### Resource Management

- Use `using` statements for disposable resources
- Be mindful of PowerShell's object pipeline - avoid holding references longer than necessary
- Follow .NET garbage collection best practices

The codebase follows .NET coding standards and uses extensive XML documentation comments. When making changes, ensure compatibility across all supported platforms (Windows, Linux, macOS) and test with both PowerShell 7+ scenarios.
