# Overview of libcdoc

The libcdoc library is a modular and extensible C++ library designed for handling CDoc containers. It provides robust support for encryption, decryption, and secure document workflows, making it a critical component of the Estonian eID ecosystem.

## Key Features

- **Cross-Platform Support:** The library is compatible with Windows, macOS, and Linux for desktop environments, as well as iOS and Android for mobile platforms.
- **Support for CDoc Versions:** Handles CDoc 1.x and 2.0 formats, ensuring compatibility with legacy systems and modern security standards.
- **Encryption and Decryption:** Provides secure workflows for encrypting and decrypting documents, including multi-recipient support.
- **Integration with eID Systems:** Works seamlessly with Estonian ID-card infrastructure and other digital identity systems.
- **Multi-Language Support:** Offers Java and C# bindings using SWIG for cross-platform development.
- **Command-Line Tool:** Includes the `cdoc-tool` utility for managing CDoc containers via the command line, refer to the [Tool](tool.md) document.

For detailed workflows, refer to the [Usage](usage.md) document.

## Library Architecture

The libcdoc library is structured around modular components that handle various aspects of CDoc container processing. The design emphasizes maintainability and flexibility to accommodate evolving standards and security requirements.

### Core Components

- **CDocReader/CDocWriter:** Responsible for reading and writing CDoc 1.x and 2.0 containers. It supports parsing XML-based metadata and managing encrypted payloads.
- **Encryption Engine:** Handles encryption and decryption using cryptographic libraries such as OpenSSL. It supports a variety of schemes, including public key encryption (smartcards, ID-cards, Mobile-ID, Smart-ID), symmetric password-based encryption, and hybrid encryption models.
- **Recipient Management:** Implements recipient-specific key wrapping and metadata handling for multi-recipient encrypted documents.
- **Workflow Support:** Enables both online and offline workflows, supporting scenarios where keys are available via connected tokens or remote signing/encryption services.
- **Standards Abstraction:** Provides an abstraction layer for handling differences between CDoc versions and upcoming specification changes.

### Extensibility

The architecture is designed to allow easy integration of new encryption backends and recipient schemes. For instance, support for emerging mobile and cloud-based identity systems can be added without restructuring core functionality.

### Multi-Language Support with SWIG

The libcdoc library uses **SWIG (Simplified Wrapper and Interface Generator)** to generate bindings for Java and C#. This allows developers to use the library in applications written in these languages without needing to write custom wrappers manually.

- **Java Wrappers:** SWIG generates Java bindings, enabling seamless integration of libcdoc into Java-based applications. This is particularly useful for enterprise systems, Android applications, and cross-platform mobile development.
- **C# Wrappers:** SWIG also generates C# bindings, making libcdoc accessible to .NET developers for use in Windows desktop applications or cross-platform .NET Core projects.

The library's compatibility with iOS and Android platforms ensures that developers can integrate libcdoc into mobile applications, enabling secure document workflows on mobile devices. The use of SWIG ensures that the library's core functionality remains consistent across all supported languages and platforms, reducing maintenance overhead and improving interoperability.

## Interoperability

libcdoc maintains strong compatibility with other components in the Estonian eID ecosystem and adheres closely to official specifications to ensure interoperability across different platforms and implementations.

This architectural foundation ensures that libcdoc remains a reliable and future-ready library for secure document encryption in both governmental and private sector applications.