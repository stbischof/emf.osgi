# Fennec EMF OSGi Specification

## Chapter 1: Overview

### 1.1 Introduction

This document specifies the Fennec EMF OSGi service definitions. The Fennec EMF OSGi specification defines a standard way for bundles to use the Eclipse Modeling Framework (EMF) in a pure OSGi environment, embracing the dynamic and service-oriented nature of OSGi.

This specification enables the replacement of EMF's static, global registries with a dynamic, service-based approach for managing the lifecycle of EPackages, ResourceFactories, and ResourceSets.

### 1.2 The Problem

The Eclipse Modeling Framework (EMF) is a powerful framework for creating structured data models and generating code from them. However, its core architecture was designed around a static, centralized registry model and an extension-point system tied to the Eclipse IDE. Key components like `EPackage.Registry.INSTANCE` and `Resource.Factory.Registry.INSTANCE` are global singletons.

This static nature presents significant challenges in a dynamic OSGi environment:
*   **Lifecycle Mismatch:** EMF's static registries do not align with the dynamic lifecycle of OSGi bundles. When a bundle providing an EMF model is stopped or uninstalled, its registrations remain in the global registries, potentially leading to stale references and class loading issues.
*   **Lack of Modularity:** The global registries create tight coupling between bundles, hindering modularity and versioning. It is difficult to run multiple versions of the same model side-by-side.
*   **Dependency on Eclipse:** The traditional mechanism for registering EMF models relies on the `org.eclipse.emf.ecore.generated_package` extension point, which is specific to the Eclipse Equinox runtime and requires a dependency on the Eclipse Platform.

### 1.3 The Solution

The Fennec EMF OSGi specification defines a set of services and patterns that solve these problems by integrating EMF with the OSGi service model. This approach allows for the dynamic discovery and lifecycle management of EMF models and related artifacts.

The core principles of the solution are:
*   **Service-Oriented Registries:** EMF's global registries are replaced by OSGi services that act as registries. These services are scoped and managed by the OSGi framework.
*   **Whiteboard Pattern:** Bundles can register their EMF models (EPackages) and ResourceFactories as OSGi services using well-defined service properties. Registry services discover and aggregate these services as they appear and disappear.
*   **Capability-Based Discovery:** Consumers can look up `ResourceSet` or `ResourceSetFactory` services that are pre-configured with the specific models they need by using filters on OSGi service properties. This decouples consumers from providers.
*   **Pure OSGi:** The entire framework operates in any OSGi R7+ compliant framework, with no dependencies on the Eclipse IDE or its extension point mechanism.

This specification details the service interfaces, service properties, and interaction patterns required for bundles to provide, discover, and consume EMF models in a dynamic and modular fashion.

## Chapter 2: Glossary

This specification uses terminology from both OSGi and EMF.

*   **Bundle:** The OSGi unit of modularity and deployment. A bundle is a JAR file containing code, resources, and a manifest describing its contents and dependencies.
*   **Service Registry:** The central component of the OSGi service model. It provides a dynamic and collaborative environment where bundles can publish, find, and bind to services.
*   **Whiteboard Pattern:** An OSGi design pattern where bundles publish services with specific properties, and a central "tracker" service discovers them and acts upon them. This specification uses the whiteboard pattern to discover model and factory providers.
*   **EPackage:** An EMF object that represents the structure of a model (a metamodel). It contains classifiers like classes (EClass) and data types (EDataType).
*   **Resource:** An EMF object that represents a persistent document (like a file). It contains the instances of a model (the EObjects).
*   **ResourceSet:** An EMF object that manages a collection of related `Resource` instances. It provides a scope for resolving references between models and contains a local `EPackage.Registry` and `Resource.Factory.Registry`.

## Chapter 3: Core Concepts

### 2.1 From Static to Dynamic Registries

The fundamental change introduced by this specification is the move away from static, global registries towards dynamic, service-based registries.

In standard EMF, a consumer would load a model into a `ResourceSet` and the `EPackage` would be added to the `ResourceSet`'s local `EPackage.Registry`, which in turn delegates to the global `EPackage.Registry.INSTANCE`.

In Fennec EMF OSGi, the `ResourceSet` itself is obtained as a service. This `ResourceSet` is pre-configured with `EPackage.Registry` and `Resource.Factory.Registry` instances that are also OSGi services. These registry services dynamically track available models and factories from other bundles.

### 2.2 Scoped Registries

To provide fine-grained control over model visibility, Fennec EMF OSGi introduces the concept of registry scopes. When a model is registered, it can target a specific scope, which determines where it will be visible. The primary scopes are:

*   **`static`:** Models registered in this scope are contributed to a service that replaces the global `EPackage.Registry.INSTANCE`. This is useful for foundational models that need to be available globally, but it should be used with caution as it re-introduces a global state.
*   **`resourceset`:** This is the default and recommended scope. Models registered in this scope are available only within `ResourceSet` instances that are specifically configured to include them. This provides strong isolation between different `ResourceSet`s.

This scoping mechanism allows developers to control the visibility of their models and avoid polluting the global namespace.

### 2.3 Capability Advertisement

A central concept is the advertisement of capabilities through OSGi service properties. When a provider bundle registers a model or a factory, it does so with a set of standardized service properties. These properties declare the capabilities of the model, such as its name, namespace URI, and supported file extensions.

For example, a service registered with the property `(emf.nsURI=http://www.example.org/my/model/1.0)` advertises that it provides the EPackage for that specific namespace.

Consumers can then use these properties in OSGi filters to find the exact services they need. A consumer wanting a `ResourceSet` capable of handling the model above can use a target filter like `(emf.nsURI=http://www.example.org/my/model/1.0)`. The Fennec EMF OSGi implementation ensures that the delivered `ResourceSet` is correctly configured with the requested model.

### 2.4 Provider and Consumer Roles

The architecture is divided into two primary roles:

*   **Providers:** Bundles that define EMF models (`.ecore` files) and wish to make them available to other bundles. Providers implement and register `Configurator` services (e.g., `EPackageConfigurator`) to announce their models to the Fennec EMF OSGi framework.
*   **Consumers:** Bundles that need to work with EMF models. Consumers acquire a pre-configured `ResourceSet` or a `ResourceSetFactory` service from the OSGi service registry, often using filters to specify their exact model requirements.

## Chapter 4: Consumer Services

Consumers of EMF models interact with services that provide configured EMF `ResourceSet` instances. This chapter defines the primary services that a consumer bundle will use.

### 4.1 The ResourceSetFactory Service

The `ResourceSetFactory` is the primary service for consumers. It allows for the creation of `ResourceSet` instances that are configured according to the available models and factories in the OSGi service registry.

#### 4.1.1 Service Interface

The service is registered with the interface `org.eclipse.fennec.emf.osgi.api.ResourceSetFactory`.

```java
public interface ResourceSetFactory {
    ResourceSet createResourceSet();
}
```

#### 4.1.2 Service Properties

A `ResourceSetFactory` service advertises the aggregated capabilities of all the models it can configure for a `ResourceSet`. A consumer can use these properties to select a factory that meets its requirements.

For example, a `ResourceSetFactory` that has tracked two models, `modelA` and `modelB`, will have service properties that are a union of the properties from both models. If `modelA` has `(emf.name=modelA)` and `modelB` has `(emf.name=modelB)`, the factory service will have the property `(emf.name=[modelA, modelB])`.

This allows a consumer to get a factory that supports `modelA` by using the filter `(emf.name=modelA)`.

### 4.2 The ResourceSet Service

For convenience, `ResourceSet` instances can be acquired directly from the service registry. These services are registered with a `service.scope` of `prototype`. This means that each bundle that gets the service receives its own unique `ResourceSet` instance, preventing interference between bundles.

#### 4.2.1 Service Interface

The service is registered with the interface `org.eclipse.emf.ecore.resource.ResourceSet`.

#### 4.2.2 Service Properties

Similar to the `ResourceSetFactory`, a `ResourceSet` service is registered with properties that advertise its configured capabilities. A consumer can acquire a `ResourceSet` that is pre-configured with a specific model by using the appropriate filter.

**Example: Acquiring a `ResourceSet` for `modelA`**
```java
@Reference(target = "(emf.name=modelA)")
private ResourceSet resourceSet;
```
When this service is injected, the consumer receives a `ResourceSet` instance whose `EPackage.Registry` and `Resource.Factory.Registry` are already populated with `modelA` and any associated resource factories.

## Chapter 5: Provider Services (The Whiteboard Pattern)

Providers of EMF models and resource factories make them available to the framework by registering services that the Fennec EMF OSGi implementation tracks using the Whiteboard Pattern. This chapter defines the service contracts for providers.

### 5.1 The EPackageConfigurator Service

A bundle that provides one or more EMF models (EPackages) must register an `EPackageConfigurator` service for each model.

#### 5.1.1 Service Interface

The service is registered with the interface `org.eclipse.fennec.emf.osgi.api.EPackageConfigurator`.

```java
public interface EPackageConfigurator {
    void configure(EPackage.Registry registry);
    void unconfigure(EPackage.Registry registry);
}
```

*   `configure(EPackage.Registry registry)`: This method is called by the framework to add the provider's `EPackage`(s) to the given registry.
*   `unconfigure(EPackage.Registry registry)`: This method is called by the framework to remove the provider's `EPackage`(s) from the given registry when the provider service is going away.

#### 5.1.2 Service Properties and Example

This service is the primary mechanism for advertising model capabilities. It MUST be registered with properties identifying the model. The following example shows how a provider could manually register a configurator for a "Library" model.

```java
// In a bundle activator or DS component

Dictionary<String, Object> props = new Hashtable<>();
props.put(EMFNamespaces.EMF_NAME, "library");
props.put(EMFNamespaces.EMF_MODEL_NSURI, "http://example.com/library");

EPackageConfigurator configurator = new EPackageConfigurator() {
    public void configure(EPackage.Registry registry) {
        registry.put("http://example.com/library", LibraryPackage.eINSTANCE);
    }
    public void unconfigure(EPackage.Registry registry) {
        registry.remove("http://example.com/library");
    }
};

context.registerService(EPackageConfigurator.class, configurator, props);
```

### 5.2 Resource Factory Providers

A bundle that provides a `Resource.Factory` for a specific serialization format can register it in one of two ways.

#### 5.2.1 Direct Resource.Factory Registration

The most direct method is to register an implementation of `org.eclipse.emf.ecore.resource.Resource.Factory` as an OSGi service. The framework will automatically track any service registered with this interface that also has the required `emf.*` properties.

#### 5.2.2 The ResourceFactoryConfigurator Service

Alternatively, a bundle can register a `ResourceFactoryConfigurator` service. This provides more control over the registration and unregistration logic.

*   **Service Interface**: `org.eclipse.fennec.emf.osgi.api.ResourceFactoryConfigurator`

#### 5.2.3 Service Properties

Whether registering a `Resource.Factory` directly or via a configurator, the service MUST be registered with properties that describe the type of resource it can handle. At least one of the following is required:

*   `emf.fileExtension` (String or String[]): The file extension(s) this factory handles (e.g., `xmi`, `library`).
*   `emf.protocol` (String or String[]): The URI protocol(s) this factory handles (e.g., `http`).
*   `emf.contentType` (String or String[]): The content type identifier(s) this factory handles.

## Chapter 6: Service Properties

Service properties are the cornerstone of the Fennec EMF OSGi specification. They form the public contract that enables discovery and capability matching. All properties are defined in the `org.eclipse.fennec.emf.osgi.api.EMFNamespaces` interface.

### 6.1 Property Propagation

The Fennec EMF OSGi implementation automatically propagates properties from provider services (`EPackageConfigurator`, `Resource.Factory`, etc.) to consumer services (`ResourceSetFactory`, `ResourceSet`). When multiple providers are aggregated, their properties are merged. For properties that accept a String array, the resulting value is a union of all provider values.

### 6.2 Model Identification Properties

These properties are used on `EPackageConfigurator` services to describe the model.

*   **`emf.name`**: (`String` or `String[]`) The logical name of the EPackage. This is typically the value of `EPackage.getName()`.
*   **`emf.nsURI`**: (`String` or `String[]`) The globally unique Namespace URI of the EPackage. This is the primary identifier for a model and corresponds to `EPackage.getNsURI()`.
*   **`emf.version`**: (`String`) The version of the model. To ensure consistency, this should follow the OSGi version format (major.minor.micro.qualifier).

### 6.3 Resource Factory Properties

These properties are used on `ResourceFactoryConfigurator` or `Resource.Factory` services to describe their capabilities.

*   **`emf.fileExtension`**: (`String` or `String[]`) The file extension that the resource factory supports (e.g., "xml", "xmi").
*   **`emf.protocol`**: (`String` or `String[]`) The URI protocol that the resource factory supports (e.g., "http", "ftp").
*   **`emf.contentType`**: (`String` or `String[]`) The content type identifier that the resource factory supports.

### 6.4 Framework Control Properties

These properties are used to control the behavior of the Fennec EMF OSGi framework.

*   **`emf.model.scope`**: (`String`) Used on an `EPackageConfigurator` to specify the target registry scope. Valid values are `static` and `resourceset`. If omitted, `resourceset` is the default.
*   **`emf.configurator.name`**: (`String`) An optional name for a specific `EPackageConfigurator` or `ResourceFactoryConfigurator` implementation, which can be used for debugging or filtering.

### 6.5 Extensibility Properties

These properties provide a mechanism for adding custom metadata.

*   **`emf.feature`**: (`String` or `String[]`) A property to advertise a special feature of a model or factory.
*   **`emf.feature.*`**: Any service property starting with the prefix `emf.feature.` will have this prefix removed and the remaining key-value pair will be propagated to the `ResourceSetFactory` and `ResourceSet` services. For example, a configurator with the property `emf.feature.author=JohnDoe` will cause the factory to have the property `author=JohnDoe`.

## Chapter 7: Model Registration Strategies

While bundles can always manually register `EPackageConfigurator` services, the Fennec EMF OSGi project provides several higher-level mechanisms to simplify the registration of EMF models.

### 7.1 Code Generator

The Fennec EMF OSGi project includes a code generator that integrates with the standard EMF code generation process. By setting the "OSGi Compatible" property to `true` in a `.genmodel` file, the generator will automatically create the necessary `EPackageConfigurator` and `ResourceFactoryConfigurator` implementations for the model and register them as OSGi services with the correct properties.

This is the recommended and most robust method for providers to expose their models.

### 7.2 Extender Pattern

For situations where modifying the model's generation process is not feasible, the `org.eclipse.fennec.emf.osgi.extender` bundle provides a way to register models declaratively. A bundle containing `.ecore` model files can opt-in to this mechanism by adding a requirement to its manifest:

```
Require-Capability: osgi.extender; filter:="(osgi.extender=emf.model)"
```

The extender bundle will scan the bundle for files in the `model/` directory and automatically register `EPackageConfigurator` services for any `.ecore` files it finds. This allows existing EMF model bundles to participate in the Fennec EMF OSGi ecosystem with no code changes.

### 7.3 Dynamic Configuration

Models can also be registered dynamically using the OSGi Configuration Admin service. A component in the Fennec EMF OSGi implementation listens for factory configurations with the PID `DynamicModelConfigurator`. A configuration can specify the location of a model via the `emf.dynamicEcoreUri` property.

The following JSON shows an example of creating an instance of this configuration. The name `DynamicPackageLoader~myModel` is standard OSGi Configuration Admin syntax for creating a factory configuration instance named `myModel` from the factory PID `DynamicPackageLoader`.

```json
{
  "DynamicPackageLoader~myModel": {
    "emf.dynamicEcoreUri": "https://example.org/models/myModel.ecore",
    "emf.feature": ["experimental"]
  }
}
```

The framework will load the model from the specified URI and register it as an `EPackageConfigurator` service with the properties provided in the configuration. This strategy is useful for systems where models are managed and deployed centrally.

## Chapter 8: Security Considerations

When running in an OSGi framework with a Java Security Manager enabled, bundles must be granted certain permissions to participate in the Fennec EMF OSGi ecosystem.

### 8.1 Service Permissions

Bundles require `org.osgi.framework.ServicePermission` to interact with the services defined in this specification.

*   **Consumers:** A consumer bundle needs `ServicePermission` with the `GET` action for the services it intends to use. For example, to get a `ResourceSetFactory`:

    ```
    ServicePermission "org.eclipse.fennec.emf.osgi.api.ResourceSetFactory" "get"
    ```

*   **Providers:** A provider bundle needs `ServicePermission` with the `REGISTER` action for the services it provides. For example, to register an `EPackageConfigurator`:

    ```
    ServicePermission "org.eclipse.fennec.emf.osgi.api.EPackageConfigurator" "register"
    ```

### 8.2 Other Permissions

*   **Extender and Dynamic Configuration:** The model registration strategies that load models from URIs or other bundles may require additional permissions. For example, loading a model from a remote URL via the dynamic configuration mechanism would require `java.net.SocketPermission` to connect to the remote host. Loading models from other bundles requires `org.osgi.framework.AdminPermission` with the `RESOURCE` action.

Implementers and deployers should be aware of these requirements and grant bundles only the permissions they need to perform their function.

## Chapter 9: End-to-End Example

This chapter walks through a complete scenario to demonstrate how the different parts of the specification work together.

### 9.1 Scenario

A `library.provider` bundle defines a "Library" model and a custom persistence format with the file extension `.library`. A `library.consumer` bundle wants to use this model to load `.library` files.

*   **Model Namespace URI:** `http://example.com/library`
*   **File Extension:** `library`

### 9.2 The Model Provider

The `library.provider` bundle contains the generated EMF model code for the Library. It registers an `EPackageConfigurator` to make the model available.

```java
// In the provider's BundleActivator or a DS Component

Dictionary<String, Object> props = new Hashtable<>();
props.put(EMFNamespaces.EMF_MODEL_NSURI, LibraryPackage.eNS_URI);

// Registering the EPackageConfigurator
context.registerService(EPackageConfigurator.class, new EPackageConfigurator() {
    @Override
    public void configure(Registry registry) {
        registry.put(LibraryPackage.eNS_URI, LibraryPackage.eINSTANCE);
    }

    @Override
    public void unconfigure(Registry registry) {
        registry.remove(LibraryPackage.eNS_URI);
    }
}, props);
```

### 9.3 The Factory Provider

To handle the `.library` file format, the `library.provider` bundle also registers a `Resource.Factory`.

```java
// In the provider's BundleActivator or a DS Component

Dictionary<String, Object> props = new Hashtable<>();
props.put(EMFNamespaces.EMF_MODEL_FILE_EXT, "library");

// Registering the Resource.Factory directly
context.registerService(Resource.Factory.class, new LibraryResourceFactory(), props);
```

### 9.4 The Consumer

The `library.consumer` bundle needs a `ResourceSet` that is configured with both the Library model and the `.library` resource factory. It can acquire this directly by using a Declarative Services `@Reference` with a target filter that specifies both requirements.

```java
@Component
public class LibraryConsumer {

    @Reference(
        target = "(&("
            + EMFNamespaces.EMF_MODEL_NSURI + "=http://example.com/library)("
            + EMFNamespaces.EMF_MODEL_FILE_EXT + "=library))"
    )
    private ResourceSet resourceSet;

    public void loadBook(URI uri) throws IOException {
        // The injected resourceSet is already configured.
        // It knows about the Library model and the .library factory.
        Resource resource = resourceSet.getResource(uri, true);
        Book book = (Book) resource.getContents().get(0);
        System.out.println("Loaded book: " + book.getTitle());
    }
}
```

This example demonstrates the decoupling of the consumer from the provider. The consumer does not need to know which bundle provides the model or the factory, only the capabilities it requires. The Fennec EMF OSGi framework handles the discovery and wiring.
