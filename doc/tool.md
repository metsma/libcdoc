# libcdoc Tool Usage

The **libcdoc** library includes a command-line tool, **cdoc-tool** (or **cdoc-tool.exe** on Windows), which can be used to encrypt and decrypt files, as well as view the locks in an encrypted container.

---

## Encryption

The general syntax for encrypting one or more files for one or more recipients is as follows:

```bash
cdoc-tool encrypt --rcpt RECIPIENT [--rcpt...] [-v1] [--genlabel]
    [--library PKCS11_LIBRARY]
    [--server ID URL(s)]
    [--accept SERVER_CERT_FILENAME]
    --out OUTPUTFILE
    FILE1 [FILE2 FILE3... FILEn]
```

To re-encrypt a file by adding new recipients, use the **re-encrypt** switch instead of **encrypt**. All other options remain the same.

### Options

- `-v1`  
  Generates a CDOC version 1 format container instead of the default CDOC version 2 container. This option is only valid when encrypting with a public-key certificate (see [Recipients](#recipients)). Using this option with other encryption methods will result in an error.

- `--genlabel`  
  Generates a machine-readable label for the lock instead of using the label provided with the `--rcpt` option. The generated label follows the format described in the [KeyLabel Recommendations](https://open-eid.github.io/CDOC2/1.1/02_protocol_and_cryptography_spec/ch03_container_format/#keylabel-recommendations) section of the *CDOC2 Container Format* specification.

- `--library PKCS11_LIBRARY`  
  Specifies the path to the PKCS11 library for handling smart-card operations. This option is required on macOS and Linux when using the `p11sk` or `p11pk` encryption methods (see [Recipients](#recipients)). On Windows, the tool uses the native Windows API, and this option is ignored.

- `--server ID URL(s)`  
  Specifies a key or share server. The recipient key will be stored on the server instead of in the container. For a key server, the URL is either the fetch or send URL. For a share server, it is a comma-separated list of share server URLs.

- `--accept SERVER_CERT_FILENAME`  
  Specifies the path to the server's TLS certificate file in DER format. This option is required if a key or share server is used (i.e., when the `--server` option is specified).

- `--out OUTPUTFILE`  
  Specifies the name of the output CDOC file. This option is mandatory. If no path is provided, the file is created in the current working directory.

- `FILE`  
  Specifies one or more files to be encrypted. At least one file must be provided.

### Recipients

One or more recipients must be specified, each with its own encryption method. At least one recipient is required.

| Form | Description |
| --- | --- |
| `[label]:cert:CERTIFICATE_HEX` | Encrypts using a public key from a certificate provided as a hex-encoded string. |
| `[label]:skey:SECRET_KEY_HEX` | Encrypts using a symmetric AES key provided as a hex-encoded string. |
| `[label]:pkey:SECRET_KEY_HEX` | Encrypts using a public key provided as a hex-encoded string. |
| `[label]:pfkey:PUB_KEY_FILE` | Encrypts using a public key from a DER file (EC **secp384r1** curve). |
| `[label]:pw:PASSWORD` | Encrypts using a password-derived key (PWBKDF). |
| `[label]:p11sk:SLOT:[PIN]:[PKCS11 ID]:[PKCS11 LABEL]` | Encrypts using an AES key from a PKCS11 module. |
| `[label]:p11pk:SLOT:[PIN]:[PKCS11 ID]:[PKCS11 LABEL]` | Encrypts using a public key from a PKCS11 module. |
| `[label]:share:ID` | Encrypts using a key share server with the specified ID (e.g., a personal code). |

If the `label` is omitted, the `--genlabel` option must be specified. Otherwise, the tool will generate an error. If both `label` and `--genlabel` are provided, the behavior depends on the encryption method. Refer to [Appendix D](https://open-eid.github.io/CDOC2/1.1/02_protocol_and_cryptography_spec/appendix_d_keylabel/) of the *CDOC2 Container Format* specification for examples.

### Examples

1. **Encrypt a file with a password**  
   Encrypt the file `abc.txt` with the password `Test123`. The resulting container is `abc.txt-pw.cdoc`.

   ```bash
   ./cdoc-tool encrypt --rcpt Test:pw:Test123 --out abc.txt-pw.cdoc ~/Documents/abc.txt
   ```

2. **Encrypt a file with a public key from an ID card**  
   Encrypt the file `abc.txt` using a public key from an Estonian ID card. The resulting container is `abc.txt-p11pk.cdoc`.

   ```bash
   ./cdoc-tool encrypt --rcpt :p11pk:0:::Isikutuvastus --genlabel --out abc.txt-p11pk.cdoc --library /opt/homebrew/lib/opensc-pkcs11.so ~/Documents/abc.txt
   ```

3. **Encrypt a file with a public key from a file**  
   Encrypt the file `abc.txt` using a public key from the file `ec-secp384r1-pub.der`. The resulting container is `abc.txt-pfkey.cdoc`.

   ```bash
   ./cdoc-tool encrypt --rcpt :pfkey:ec-secp384r1-pub.der --genlabel --out abc.txt-pfkey.cdoc ~/Documents/abc.txt
   ```

4. **Encrypt a file with an AES key**  
   Encrypt the file `abc.txt` using an AES key provided via the command line. The resulting container is `abc.txt-aes.cdoc`.

   ```bash
   ./cdoc-tool encrypt --rcpt Test:skey:E165475C6D8B9DD0B696EE2A37D7176DFDF4D7B510406648E70BAE8E80493E5E --genlabel --out abc.txt-aes.cdoc ~/Documents/abc.txt
   ```

5. **Encrypt a file with a public key from a key server**  
   Encrypt the file `abc.txt` using a public key from the RIA test key server. **A VPN connection to RIA must be established!** The resulting container is `abc.txt-ks.cdoc`.

   ```bash
   ./cdoc-tool encrypt --rcpt Test:p11pk:0:::Isikutuvastus --library /opt/homebrew/lib/opensc-pkcs11.so --server 00000000-0000-0000-0000-000000000000 https://cdoc2-keyserver.test.riaint.ee:8443 --accept keyserver-cert.der --out abc.txt-ks.cdoc ~/Documents/abc.txt
   ```

---

## Decryption

The syntax for decrypting an encrypted file is as follows:

```bash
cdoc-tool decrypt OPTIONS FILE [OUTPUT_DIR]
```

### Options

- `--label LABEL`  
  Specifies the lock label of the CDOC container. Either the label or the label's index must be provided.

- `--label_idx INDEX`  
  Specifies the 1-based index of the lock label in the CDOC container. Either the label or the label's index must be provided.

- `--slot SLOT`  
  Specifies the PKCS11 slot number (usually `0`).

- `--password PASSWORD`  
  Specifies the password if the file was encrypted with a password.

- `--secret SECRET`  
  Specifies the secret phrase (AES key) if the file was encrypted with a symmetric key.

- `--pin PIN`  
  Specifies the PKCS11 (smart card) PIN code.

- `--key-id`  
  Specifies the PKCS11 key ID.

- `--key-label`  
  Specifies the PKCS11 key label.

- `--library PKCS11_LIBRARY`  
  Specifies the path to the PKCS11 library. This is the same as in the encryption case.

- `--server ID URL(s)`  
  Specifies a key or share server. This is the same as in the encryption case.

- `--accept SERVER_CERT_FILENAME`  
  Specifies the path to the server's TLS certificate file. This is the same as in the encryption case.

- `FILE`  
  Specifies the encrypted file to be decrypted.

- `OUTPUT_DIR`  
  Specifies the output directory where the decrypted files will be saved. If not specified, the current working directory is used. Existing files with the same name will be overwritten.

### Examples

1. **Decrypt a file with a password**  
   Decrypt the file `abc.txt-pw.cdoc` using the key label `Test` and password `Test123`.

   ```bash
   ./cdoc-tool decrypt --label Test --password Test123 abc.txt-pw.cdoc
   ```

2. **Decrypt a file with an ID card**  
   Decrypt the file `abc.txt-p11pk.cdoc` using the key label index `1` and an Estonian ID card with PIN code `1234`.

   ```bash
   ./cdoc-tool decrypt --label_idx 1 --pin 1234 --slot 0 --key-label Isikutuvastus --library /opt/homebrew/lib/opensc-pkcs11.so abc.txt-p11pk.cdoc
   ```

3. **Decrypt a file with a key server**  
   Decrypt the file `abc.txt-ks.cdoc` using the key label `Test` and a private key from the RIA test key server. **A VPN connection to RIA must be established!**

   ```bash
   ./cdoc-tool decrypt --library /opt/homebrew/lib/opensc-pkcs11.so --server 00000000-0000-0000-0000-000000000000 https://cdoc2-keyserver.test.riaint.ee:8444 --accept keyserver-cert.der --label Test --slot 0 --pin 1234 --key-label Isikutuvastus abc.txt-ks.cdoc out
   ```

---

## Viewing Locks

To view the locks in a container, use the following syntax:

```bash
cdoc-tool locks FILE
```

This command does not have any options. The only argument is the encrypted container file whose locks will be displayed.

### Example

Display the locks of the file `abc.txt-aes.cdoc`:

```bash
./cdoc-tool locks abc.txt-aes.cdoc
```
