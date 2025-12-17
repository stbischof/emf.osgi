# Eclipse Fennec EMF OSGi Project - Complete Development Specification

## Project Overview

**Project Name**: Eclipse Fennec EMF OSGi (formerly GeckoEMF)  
**Purpose**: Enable Eclipse Modeling Framework (EMF) to work in pure OSGi environments without Eclipse PDE dependencies  
**Architecture**: Service-based EMF model management with dynamic lifecycle support  

## Build System Requirements

### Core Build Configuration
- **Build Tool**: Gradle + BND Workspace hybrid setup
- **Java Version**: 17 (source and target)
- **BND Version**: 7.1.0+ required
- **Encoding**: UTF-8 for all compilation

### Primary Build Commands
```bash
./gradlew build          # Main build
./gradlew test           # Run all JUnit tests
./gradlew testOSGi       # Run OSGi integration tests
./gradlew clean          # Clean build artifacts
./gradlew codeCoverageReport  # Generate Jacoco coverage
./gradlew sonar          # SonarQube analysis
```

### Repository Configuration
Required Maven repositories in `cnf/build.bnd`:
1. **Maven Central**: `https://repo.maven.apache.org/maven2/`
2. **EMF Snapshots**: `https://repo.eclipse.org/content/repositories/emf-snapshots/`
3. **DIM UML2**: `https://devel.data-in-motion.biz/nexus/repository/maven-releases/`
4. **BND Snapshots**: `https://bndtools.jfrog.io/bndtools/libs-snapshot-local`

## Module Architecture

### 1. API Module (`org.eclipse.fennec.emf.osgi.api`)
**Purpose**: Core interfaces and constants

**Key Components**:
- **Configurator Interfaces**:
  - `EPackageConfigurator` - EPackage registration/unregistration
  - `ResourceSetConfigurator` - ResourceSet customization
  - `ResourceFactoryConfigurator` - Resource factory management
- **Service Interfaces**:
  - `ResourceSetFactory` - ResourceSet creation service
  - `RegistryTrackingService` - Registry property change tracking
- **Constants**: `EMFNamespaces` - All service property constants
- **Annotations**: `@EMFModel`, `@EMFConfigurator` for declarative configuration

**BND Configuration**:
```properties
-buildpath: \
    org.osgi.namespace.extender;version=latest,\
    org.osgi.framework;version=latest,\
    org.eclipse.emf.ecore;version=latest,\
    org.eclipse.emf.common;version=latest

src=${^src},src-gen
-generate: src-version/**.java; output='src-gen/'
```

### 2. Core Implementation (`org.eclipse.fennec.emf.osgi`)
**Purpose**: Main OSGi service implementations

**Three Deployment Variants**:
- **Full Bundle** (`component.bnd`): All components including config/dynamic support
- **Minimal Bundle** (`component.minimal.bnd`): Core components only
- **Config Bundle** (`component.config.bnd`): Configuration-driven components

**Key Components**:
- **Registry Components**:
  - `StaticEPackageRegistryComponent` - Replaces EMF static registry
  - `DefaultEPackageRegistryComponent` - ResourceSet-level registry
  - `DefaultResourceFactoryRegistryComponent` - Resource factory registry
- **Factory Components**:
  - `DefaultResourceSetFactoryComponent` - Main ResourceSet factory
  - `ResourceSetPrototypeFactory` - Prototype service implementation
- **Service Helpers**:
  - `DelegatingEPackageRegistry` - Registry delegation pattern
  - `ServicePropertyContext` - Dynamic property aggregation

### 3. Code Generator (`org.eclipse.fennec.emf.osgi.codegen`)
**Purpose**: BND-based EMF code generator for OSGi compatibility

**Generated Artifacts**:
- `EPackageConfigurator` implementations
- `ConfigurationComponent` OSGi components
- OSGi service registrations for EPackage, EFactory, ResourceFactory
- `Condition` services for lifecycle management

**Templates** (in `templates/model/`):
- `EPackageConfigurator.javajet`
- `ConfigComponent.javajet`
- `PackageClass.javajet`
- `FactoryClass.javajet`
- `ResourceFactoryClass.javajet`

**Trigger**: Set GenModel "OSGi Compatible" to `true`

### 4. Model Extender (`org.eclipse.fennec.emf.osgi.extender`)
**Purpose**: Auto-registration of EMF models from bundle manifests

**Key Features**:
- Tracks bundles with `Require-Capability: osgi.extender; filter:="(osgi.extender=emf.model)"`
- Discovers `.ecore` files in bundle `model/` directories
- Registers discovered models as `EPackage` and `EPackageConfigurator` services
- Supports both single model files and folder scanning

## OSGi Service Architecture

### Registry Whiteboard Pattern
**Pattern**: Registry services track configurators as they appear/disappear

**Scope-Based Targeting**:
- `emf.model.scope=static` → `StaticEPackageRegistryComponent`
- `emf.model.scope=resourceset` → `DefaultEPackageRegistryComponent`
- `emf.model.scope=generated` → Generated model registries

### Dynamic Property Propagation
**Mechanism**: Registry services automatically update properties as configurators change

**Key Classes**:
- `ServicePropertyContext` - Aggregates properties from multiple sources
- `RegistryTrackingService` - Monitors registry property changes
- `RegistryPropertyListener` - Callback interface for property updates

**Flow**:
1. Configurators register with model metadata properties
2. Registry components aggregate configurator properties
3. Registry services update their own properties
4. ResourceSetFactory services react to registry changes
5. Consumers can filter services by capabilities

### Service Property Schema
**Core Properties** (from `EMFNamespaces`):
```java
// Model Identity
EMF_NAME = "emf.name"                    // String[] - model names
EMF_MODEL_NSURI = "emf.nsURI"           // String[] - namespace URIs
EMF_MODEL_VERSION = "emf.version"        // String - model version

// Resource Factory Properties  
EMF_MODEL_FILE_EXT = "emf.fileExtension" // String[] - file extensions
EMF_MODEL_PROTOCOL = "emf.protocol"      // String[] - protocols
EMF_MODEL_CONTENT_TYPE = "emf.contentType" // String[] - content types

// Registry Targeting
EMF_MODEL_SCOPE = "emf.model.scope"      // static|resourceset|generated
EMF_MODEL_REGISTRATION = "emf.registration" // provided|dynamic|extender|internal

// Feature Properties
EMF_MODEL_FEATURE = "emf.feature"        // String[] - feature identifiers
// emf.feature.* → forwarded as custom properties
```

## Dependency Management

### Core EMF Dependencies
```xml
<!-- EMF Core -->
org.eclipse.emf:org.eclipse.emf.ecore:2.40.0
org.eclipse.emf:org.eclipse.emf.common:2.43.0
org.eclipse.emf:org.eclipse.emf.ecore.xmi:2.38.0

<!-- EMF Code Generation (for codegen module) -->
org.eclipse.emf:org.eclipse.emf.codegen.ecore:2.42.0
org.eclipse.emf:org.eclipse.emf.codegen:2.26.0
```

### OSGi Framework Dependencies
```xml
<!-- Core OSGi -->
org.osgi:org.osgi.framework:1.10.0
org.osgi:org.osgi.service.component:1.5.1
org.osgi:org.osgi.service.component.annotations:1.5.1
org.osgi:org.osgi.service.cm:1.6.1
org.osgi:org.osgi.service.condition:1.0.0

<!-- OSGi Utilities -->
org.osgi:org.osgi.util.converter:1.0.9
org.osgi:org.osgi.util.function:1.2.0
org.osgi:org.osgi.util.tracker:1.5.4
org.osgi:org.osgi.util.promise:1.3.0
```

### Runtime Framework (Apache Felix)
```xml
org.apache.felix:org.apache.felix.framework:7.0.5
org.apache.felix:org.apache.felix.scr:2.2.6
org.apache.felix:org.apache.felix.configadmin:1.9.26
org.apache.felix:org.apache.felix.configurator:1.0.18
```

### BND Build Tools
```xml
biz.aQute.bnd:biz.aQute.bndlib:7.1.0
biz.aQute.bnd:aQute.libg:7.1.0
biz.aQute.bnd:biz.aQute.bnd.annotation:7.1.0
```

## Testing Framework

### Unit Testing
- **Framework**: JUnit 5 with OSGi Test Framework
- **Assertions**: AssertJ with OSGi extensions
- **Coverage**: Jacoco with consolidated reporting
- **Execution**: `./gradlew test`

### OSGi Integration Testing
- **Runtime**: Apache Felix framework
- **Configuration**: `.bndrun` files for test runtime setup
- **Execution**: `./gradlew testOSGi`
- **Coverage**: Separate Jacoco execution data collection

**Example Test Runtime** (`test.bndrun`):
```properties
-library: enableOSGi-Test
-runrequires: bnd.identity;id='org.eclipse.fennec.emf.osgi.itest'
-runfw: org.apache.felix.framework;version='[7.0.5,7.0.5]'
-runee: JavaSE-17
```

## Model Registration Patterns

### 1. Code Generator Registration
**Setup**: Set GenModel "OSGi Compatible" = `true`
**Generated Components**:
- `EPackageConfigurator` implementation
- `ConfigurationComponent` for service registration
- Automatic service property population

### 2. Extender Registration
**Bundle Manifest**:
```
Require-Capability: osgi.extender; filter:="(osgi.extender=emf.model)"
```
**Model Location**: Place `.ecore` files in bundle `model/` directory

### 3. Dynamic Configuration Registration
**Configuration PID**: `DynamicModelConfigurator~<identifier>`
**Example Configuration**:
```json
{
  "DynamicPackageLoader~demo": {
    "emf.dynamicEcoreUri": "https://example.org/model.ecore",
    "emf.feature": ["foo", "bar"],
    "emf.feature.custom": "value"
  }
}
```

## Usage Patterns

### Service Injection
```java
// Basic ResourceSet injection
@Reference
private ResourceSet resourceSet;

// Filtered ResourceSet injection
@Reference(target="(emf.name=mymodel)")
private ResourceSet resourceSet;

// ResourceSetFactory injection
@Reference
private ResourceSetFactory factory;
```

### Model Service Access
```java
// EPackage service injection
@Reference(target="(emf.name=mymodel)")
private EPackage modelPackage;

// ResourceFactory injection
@Reference(target="(emf.fileExtension=myext)")
private Resource.Factory resourceFactory;
```

## Key Implementation Requirements

### 1. Service Registration Lifecycle
- Manual service registration via `BundleContext.registerService()`
- Dynamic property updates using `ServiceRegistration.setProperties()`
- Proper service unregistration in deactivation

### 2. Registry Delegation Pattern
- `DelegatingEPackageRegistry` with parent fallback
- `DelegatingResourceFactoryRegistry` for hierarchical lookup
- Change listener support for property propagation

### 3. Property Aggregation
- `ServicePropertyContext` for multi-source property merging
- Automatic `emf.feature.*` prefix handling
- Array property normalization and deduplication

### 4. Bundle Coordination
- Ensure `org.eclipse.emf.ecore` bundle activation before static registry access
- Proper bundle dependency management
- Exception handling for missing services

## Quality Assurance

### Code Coverage
- **Target**: Consolidated Jacoco reporting across all modules
- **Sources**: Both unit tests (`generated/jacoco/test.exec`) and OSGi tests (`generated/tmp/testOSGi/generated/jacoco.exec`)
- **Report Location**: `build/reports/jacoco/codeCoverageReport/`

### SonarQube Integration
```properties
sonar.projectName=fennec.osgi.emf
sonar.projectKey=eclipse-fennec-fennec..osgi.emf
sonar.organization=eclipse-fennec
sonar.host.url=https://sonarcloud.io
```

### CI/CD
- **Platform**: GitHub Actions
- **Workflow**: `.github/workflows/snapshot.yml`
- **Badge**: Build status in README.md

This specification provides all necessary information for a developer to recreate the Eclipse Fennec EMF OSGi project, including its sophisticated service architecture, build system, and testing framework.