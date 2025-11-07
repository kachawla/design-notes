# Terraform to Radius Resource Type Learning CLI Command

- Author: Karishma Chawla (@kachawla)

## Overview

This design proposes a new CLI command `rad resource-type learn` that automatically generates Radius resource type definitions from existing Terraform modules. The command parses Terraform module configurations and converts them into Radius-compatible YAML schemas, enabling users to quickly onboard existing Terraform infrastructure as code into the Radius ecosystem.

## Terms and definitions

- **Terraform Module**: A reusable Terraform configuration that defines a set of infrastructure resources
- **Radius Resource Type**: A custom resource definition in Radius that extends the platform's capabilities
- **terraform-config-inspect**: HashiCorp's official Go library for parsing Terraform configurations
- **Resource Type Schema**: YAML-formatted definition that describes the structure and properties of a Radius resource type
- **Namespace**: Logical grouping of related resource types (e.g., `AWS.Data`, `Custom.Network`)

## Objectives

Issue Reference: Enable seamless migration from Terraform to Radius by providing automated tooling to convert Terraform modules into Radius resource types.

### Goals

- Provide a CLI command that accepts Terraform module Git URLs and generates Radius resource type schemas
- Automatically infer meaningful namespaces based on module content and naming patterns
- Generate clean, well-documented schemas with proper default value formatting
- Support both local file output and stdout for integration with CI/CD pipelines
- Handle complex Terraform variable types including objects, arrays, and nested structures

### Non goals

- Support for all Terraform language features (only variables/inputs are converted)
- Runtime validation of generated resource types
- Automatic deployment or registration of generated resource types
- Support for Terraform modules with dynamic or computed configurations

### User scenarios

#### User story 1

As a platform engineer migrating from Terraform to Radius, I want to quickly convert my existing Terraform modules into Radius resource types so that I can leverage my existing infrastructure patterns within the Radius framework.

#### User story 2

As a developer exploring Radius capabilities, I want to experiment with converting public Terraform modules to understand how they would work as Radius resource types before committing to a migration strategy.

## User Experience

Sample Input:
```bash
rad resource-type learn --git-url https://github.com/terraform-aws-modules/terraform-aws-vpc
```

Sample Output:
```yaml
namespace: AWS.Network
types:
  awsVpc:
    apiVersions:
      2025-01-01-preview:
        schema:
          type: object
          properties:
            application:
              type: string
              description: The resource ID of the application.
            environment:
              type: string
              description: The resource ID of the environment.
            cidr:
              type: string
              description: The CIDR block for the VPC (default: 10.0.0.0/16)
            enable_nat_gateway:
              type: boolean
              description: Should be true if you want to provision NAT Gateways (default: false)
          required:
            - application
            - environment
```

## Design

### High Level Design

The `rad resource-type learn` command follows a three-stage pipeline:
1. **Input Processing**: Clone Git repository and locate Terraform module files
2. **Parsing & Analysis**: Use terraform-config-inspect to parse module configuration and extract variables
3. **Schema Generation**: Convert parsed data into Radius-compatible YAML schema with intelligent namespace inference

### Architecture Diagram

```
Git Repository → Clone → Parse (terraform-config-inspect) → Transform → Generate Schema → Output YAML
     ↓              ↓                    ↓                    ↓              ↓
[Remote Module] [Local Files] [TF Variables/Types] [Schema Properties] [Resource Type YAML]
```

### Detailed Design

The implementation consists of three main components:

1. **CLI Command Handler** (`learn.go`): Manages command-line argument parsing, orchestrates the workflow, and handles output formatting
2. **Terraform Parser** (`terraform.go`): Interfaces with terraform-config-inspect library to extract module structure and variable definitions
3. **Schema Generator** (`schema.go`): Converts Terraform variables to JSON Schema properties and infers appropriate namespaces

#### Advantages

- **Official Library Integration**: Uses HashiCorp's terraform-config-inspect for reliable Terraform parsing
- **Intelligent Namespace Inference**: Automatically categorizes modules based on provider and resource type patterns
- **Comprehensive Type Support**: Handles all Terraform variable types including complex objects and arrays
- **Clean Output Format**: Generates human-readable YAML with properly formatted default values

#### Disadvantages

- **Limited Scope**: Only processes variable declarations, not complete Terraform logic
- **External Dependencies**: Requires network access for Git repository cloning
- **Static Analysis**: Cannot handle dynamically computed module configurations

#### Proposed Option

Implement as a new subcommand under the existing `rad resource-type` command group to maintain CLI consistency and discoverability.

### CLI Design

```
rad resource-type learn [OPTIONS]

Options:
  --git-url string      Git repository URL containing the Terraform module (required)
  --output string       Output file path (default: stdout)
  --namespace string    Override the automatically inferred namespace
  --resource-type string Override the automatically generated resource type name
  --help               Show help for the command
```

### Implementation Details

#### Core Components

**CLI Command Structure**:
- Integrates with existing Radius CLI framework using cobra command library
- Validates Git URL format and accessibility
- Provides clear error messages for common failure scenarios

**Git Repository Management**:
- Creates temporary directories for repository cloning
- Supports both public and authenticated Git repositories
- Implements cleanup logic to remove temporary files

**Terraform Module Parsing**:
- Uses terraform-config-inspect library for AST-based parsing
- Extracts variable definitions with types, descriptions, and default values
- Handles complex nested types through recursive processing

**Schema Generation**:
- Maps Terraform types to JSON Schema equivalents
- Implements intelligent namespace inference algorithm
- Formats default values in human-readable format within property descriptions

### Error Handling

- **Network Errors**: Clear messaging when Git repositories are inaccessible
- **Parse Errors**: Detailed error reporting when Terraform modules are malformed
- **File System Errors**: Graceful handling of permission and disk space issues
- **Invalid Configurations**: Validation of command-line arguments with helpful suggestions

## Test plan

- **Unit Tests**: Comprehensive test coverage for all parsing and generation functions
- **Integration Tests**: End-to-end testing with real Terraform modules from public repositories
- **Edge Case Testing**: Validation of complex nested objects, empty modules, and malformed configurations
- **CLI Testing**: Command-line interface testing with various argument combinations

Test files implemented:
- `terraform_test.go`: Tests Terraform parsing logic with real module files
- `schema_test.go`: Tests schema generation and namespace inference algorithms

## Security

- **Repository Access**: Users are responsible for ensuring they have appropriate access to Git repositories
- **Temporary File Management**: Temporary directories are created with restricted permissions and cleaned up after use
- **Input Validation**: All Git URLs are validated to prevent injection attacks
- **No Secrets Storage**: The tool does not persist any authentication credentials

## Compatibility

- **Terraform Versions**: Compatible with Terraform configurations up to version 1.x
- **Git Providers**: Supports GitHub, GitLab, Bitbucket, and other standard Git hosting services
- **Operating Systems**: Cross-platform compatibility (Windows, macOS, Linux)
- **Radius Versions**: Generates schemas compatible with current Radius API versions

## Monitoring and Logging

- **Structured Logging**: Uses Radius logging framework for consistent log formatting
- **Progress Indicators**: Provides user feedback during long-running operations (Git clone, parsing)
- **Debug Mode**: Verbose logging option for troubleshooting parsing issues
- **Error Tracking**: Clear error categorization for common failure scenarios

## Development plan

1. **Phase 1**: Implement basic CLI command structure and Git repository cloning
2. **Phase 2**: Integrate terraform-config-inspect library and implement variable parsing
3. **Phase 3**: Develop schema generation logic with namespace inference algorithms
4. **Phase 4**: Add comprehensive test coverage and error handling
5. **Phase 5**: Documentation and user experience improvements

## Open Questions

- Should the command support local Terraform modules in addition to Git repositories?
- How should we handle Terraform modules with multiple sub-modules?
- Should we provide validation of generated schemas against Radius API requirements?

## Alternatives considered

1. **Manual Regex Parsing**: Initially implemented custom parsing logic but replaced with terraform-config-inspect for better reliability
2. **Terraform Plan Analysis**: Considered analyzing Terraform plan output but rejected due to complexity and requirement for valid provider credentials
3. **HCL Library Direct Usage**: Evaluated using HCL parser directly but terraform-config-inspect provides higher-level abstractions that better suit our needs

## Design Review Notes

- Feedback incorporated regarding default value formatting in property descriptions rather than schema structure
- Namespace inference algorithm enhanced based on review of common Terraform module patterns
- Error handling improved to provide more actionable user guidance

## Additional Links

- [terraform-config-inspect Documentation](https://github.com/hashicorp/terraform-config-inspect)
- [Radius Resource Types Documentation](https://docs.radapp.io/concepts/resources/)
- [JSON Schema Specification](https://json-schema.org/)