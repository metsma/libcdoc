# Basic libcdoc Library Usage

This document provides an overview of how to use the **libcdoc** library for encryption and decryption workflows.

---

## Common

Most methods in the library return a `result_t` status value, which can indicate the following:

- **Positive value**: For read and write methods, this indicates success and the number of bytes read or written.
- **OK (= 0)**: Indicates success.
- **END_OF_STREAM (= -1)**: Indicates the end of the file list in a multi-file source.
- **Any error value (< -1)**: Indicates failure.

The `END_OF_STREAM` value is only returned by `nextFile` methods. For all other methods, any negative value always indicates an error.

---

## Encryption

Encryption is managed by the `CDocWriter` object.

### CryptoBackend

To use encryption, you must create or implement a `CryptoBackend` class. The default implementation is sufficient for public key encryption schemes. For symmetric key encryption, you must implement at least one of the following methods:

#### `getSecret`

```cpp
int getSecret(std::vector<uint8_t>& dst, unsigned int idx)
```

This method copies either the password (for PBKDF-based keys) or the plain AES key into the `dst` vector. While simple, this method may expose the password or key in memory.

#### `getKeyMaterial`

```cpp
int getKeyMaterial(std::vector<uint8_t>& dst, const std::vector<uint8_t>& pw_salt, int32_t kdf_iter, unsigned int idx)
```

This method returns the key material for a symmetric key (either password or plain key) derivation. The default implementation calls `getSecret` and performs PBKDF2_SHA256 if the key is password-based.

#### `extractHKDF`

```cpp
result_t extractHKDF(std::vector<uint8_t>& dst, const std::vector<uint8_t>& salt, const std::vector<uint8_t>& pw_salt, int32_t kdf_iter, unsigned int idx)
```

This method calculates the Key Encryption Key (KEK) pre-master from a symmetric key (either password or key-based). The default implementation calls `getKeyMaterial` and performs a local HKDF extract.

---

### NetworkBackend

If you intend to use a key server, you must subclass `NetworkBackend` and implement the following methods:

#### `getClientTLSCertificate`

```cpp
int getClientTLSCertificate(std::vector<uint8_t>& dst)
```

Returns the client's TLS certificate for authentication with the key server.

#### `getPeerTLSCertificates`

```cpp
int getPeerTLSCertificates(std::vector<std::vector<uint8_t>>& dst)
```

Returns the list of acceptable peer certificates for the key server.

#### `signTLS`

```cpp
int signTLS(std::vector<uint8_t>& dst, CryptoBackend::HashAlgorithm algorithm, const std::vector<uint8_t>& digest)
```

Signs the provided digest for TLS authentication.

In addition to implementing `NetworkBackend`, you must also instantiate a `Configuration` subclass.

---

### Configuration

The `Configuration` class is required to retrieve key server parameters. You must subclass it and implement the following method:

#### `getValue`

```cpp
std::string getValue(std::string_view domain, std::string_view param)
```

Returns the configuration value for a given domain/parameter combination. For key servers:

- **Domain**: The key server ID.
- **Param**: `KEYSERVER_SEND_URL`.

---

### CDocWriter

The `CDocWriter` object is created using one of the following static methods:

```cpp
CDocWriter* createWriter(int version, DataConsumer* dst, bool take_ownership, Configuration* conf, CryptoBackend* crypto, NetworkBackend* network);
CDocWriter* createWriter(int version, std::ostream& ofs, Configuration* conf, CryptoBackend* crypto, NetworkBackend* network);
CDocWriter* createWriter(int version, const std::string& path, Configuration* conf, CryptoBackend* crypto, NetworkBackend* network);
```

- **`version`**: Specifies the file format (1 for CDOC version 1, 2 for CDOC version 2).
- **`crypto`**: A `CryptoBackend` instance (required).
- **`network`**: A `NetworkBackend` instance (optional, for key server use).
- **`conf`**: A `Configuration` instance (optional, for key server use).

The `CDocWriter` does not take ownership of these objects, so you must delete them manually.

#### Workflow

1. **Add Recipients**  
   Add one or more recipients using:

   ```cpp
   int addRecipient(const Recipient& rcpt);
   ```

2. **Begin Encryption**  
   Start the encryption process:

   ```cpp
   int beginEncryption();
   ```

3. **Write Files**  
   Add files and write their data:

   ```cpp
   int addFile(const std::string& name, size_t size);
   int64_t writeData(const uint8_t* src, size_t size);
   ```

4. **Finish Encryption**  
   Finalize the encryption process:

   ```cpp
   int finishEncryption();
   ```

---

### Implementation Example

```cpp
struct MyBackend : public libcdoc::CryptoBackend {
    int getSecret(std::vector<uint8_t>& dst, unsigned int idx) override final {
        /* Write secret to dst */
    }
};

/* In the data processing method */
MyBackend crypto;

CDocWriter* writer = createWriter(version, cdoc_filename, nullptr, &crypto, nullptr);

/* Add recipients */
writer->addRecipient(myrcpt);
writer->beginEncryption();

/* Add files */
writer->addFile(filename, -1);
writer->writeData(data, data_size);

/* Finalize encryption */
writer->finishEncryption();

delete writer;
```

---

## Decryption

Decryption is managed by the `CDocReader` object.

### CryptoBackend

To decrypt all lock types, you must implement the following methods in a `CryptoBackend` subclass:

#### `deriveECDH1`

```cpp
int deriveECDH1(std::vector<uint8_t>& dst, const std::vector<uint8_t>& public_key, unsigned int idx);
```

Derives a shared secret using the ECDH1 algorithm.

#### `decryptRSA`

```cpp
int decryptRSA(std::vector<uint8_t>& dst, const std::vector<uint8_t>& data, bool oaep, unsigned int idx);
```

Decrypts data using an RSA private key.

You must also implement one of the symmetric key methods listed in the encryption section for symmetric key support.

---

### CDocReader

To determine whether a file or `DataSource` is a valid CDOC container, use:

```cpp
int getCDocFileVersion(const std::string& path);
int getCDocFileVersion(DataSource* src);
```

Both methods return the container version (1 or 2) or a negative value if the file/source is invalid.

Create a `CDocReader` instance using one of the following static constructors:

```cpp
CDocReader* createReader(DataSource* src, bool take_ownership, Configuration* conf, CryptoBackend* crypto, NetworkBackend* network);
CDocReader* createReader(const std::string& path, Configuration* conf, CryptoBackend* crypto, NetworkBackend* network);
CDocReader* createReader(std::istream& ifs, Configuration* conf, CryptoBackend* crypto, NetworkBackend* network);
```

#### Workflow

1. **Get Locks**  
   Retrieve the list of locks in the container:

   ```cpp
   const std::vector<Lock>& getLocks();
   ```

2. **Decrypt FMK**  
   Obtain the File Master Key (FMK) for the container:

   ```cpp
   result_t getFMK(std::vector<uint8_t>& fmk, unsigned int lock_idx);
   ```

3. **Begin Decryption**  
   Start the decryption process:

   ```cpp
   result_t beginDecryption(const std::vector<uint8_t>& fmk);
   ```

4. **Read Files**  
   Read individual files:

   ```cpp
   result_t nextFile(std::string& name, int64_t& size);
   result_t readData(uint8_t* dst, size_t size);
   ```

5. **Finish Decryption**  
   Finalize the decryption process:

   ```cpp
   result_t finishDecryption();
   ```

---

### Implementation Example

```cpp
struct MyBackend : public libcdoc::CryptoBackend {
    result_t deriveECDH1(std::vector<uint8_t>& dst, const std::vector<uint8_t>& public_key, unsigned int idx) override final {
        /* Derive shared secret */
    }
    result_t decryptRSA(std::vector<uint8_t>& dst, const std::vector<uint8_t>& data, bool oaep, unsigned int idx) override final {
        /* Decrypt data */
    }
};

MyBackend crypto;

CDocReader* reader = createReader(cdoc_filename, nullptr, &crypto, nullptr);

/* Get locks */
auto locks = reader->getLocks();

/* Decrypt FMK */
std::vector<uint8_t> fmk;
reader->getFMK(fmk, lock_idx);

/* Start decryption */
reader->beginDecryption(fmk);
std::string name;
int64_t size;
while (reader->nextFile(name, size) == libcdoc::OK) {
    /* Read data */
    reader->readData(buffer, size);
}

/* Finalize decryption */
reader->finishDecryption();

delete reader;
```

---
