# FIDO2 TLS1.3 Extension



**fidoSSL** is a proof of concept implementation of a TLS1.3 extension that integrates [FIDO](https://fidoalliance.org/what-is-fido/) authentication with [openSSL](https://www.openssl.org/). This extension leverages the secure, hardware-based storage of  cryptographic keys through FIDO, offering a robust alternative to  traditional client certificates. Beyond its security advantages, FIDO authentication stands out for its user-friendly and easier-to-manage nature, significantly improving the user experience. Additionally, its phishing-resistant features make it a safer choice, reducing the risk of security breaches.

This project serves as a foundational draft for further studies and development in the realm of secure communications.

### Features

- Implements FIDO authentication within the TLS protocol
- Uses the TLS1.3 extension mechanism ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446#section-4.2))
- fidoSSL is crafted as a shared library, facilitating its addition to  projects as OpenSSL callbacks. This design allows the use of a custom  extension without the need for modifying or patching the OpenSSL source code. However, it necessitates adaptations within the application's source code to integrate the FIDO-based authentication capabilities effectively.

### Prerequisites

- C programming environment (GCC, Clang, etc.)
- Dependencies: 
  - **libssl**, **libcrypto** (openSSL)
  - **libfido2** (high level API for WebAuthn & CTAP)
  - **tinycbor** (CBOR encoder & parser)
  - **sqlite3** (SQLite database)
  - **libjansson** (JSON encoder & parser)
- Basic TLS concepts and how it is used in openSSL [[link1](https://www.openssl.org/docs/man3.2/man7/ossl-guide-introduction.html), [link2](https://www.openssl.org/docs/man3.2/man7/ossl-guide-tls-introduction.html), [link3](https://www.openssl.org/docs/man3.2/man7/ossl-guide-tls-client-non-block.html), [link4](https://www.openssl.org/docs/man3.2/man7/ossl-guide-tls-client-block.html)]

### Installation

This section provides a step-by-step guide to downloading and setting up `fidoSSL` on your system.

#### Dependencies

###### macOS

On macOS, the dependencies can be installed with [homebrew](https://brew.sh/)

```bash
# Update homebrew in order to get the newest formulas
brew update

# Install dependencies
brew install openssl libfido2 sqlite jansson
```

###### ubuntu

On Ubuntu, dependencies can be installed with [aptitude](https://wiki.ubuntuusers.de/aptitude/)

```sh
# Update apt in order to get the newest packages sources
sudo apt update

# Install dependencies
sudo apt install build-essential libssl-dev libfido2-dev libsqlite3-dev  libjansson-dev
```

#### Building the Shared Library

1. **Clone the Repository**: Start by cloning the `fidoSSL` repository to your local machine

   ```bash
   git clone https://github.com/tummetott/fidoSSL.git
   ```

2. **Build the Library**: Navigate into the cloned repository directory and build the shared library

   ```bash
   cd fidoSSL
   make
   ```

3. **Installation** (Optional): After building, you have the option to install the library and header files to standard system locations, simplifying the process of linking against `fidoSSL` for future projects. This step is not mandatory; you can also choose to link against the library manually by specifying its path.
   To install `fidoSSL` to the default locations (`/usr/local/lib` for libraries and `/usr/local/include` for headers)

   ```bash
   sudo make install
   ```


**Note on Manual Linking**: If you prefer not to install `fidoSSL` globally or need to use it in a specific project without affecting the system-wide configuration, you can directly reference the library and header files in your compilation command. The compiled library can be found under `fidoSSL/build/libfidossl.a`.

#### Compiling Your Application with fidoSSL

When compiling an application that uses `fidoSSL`, include the path to the library's headers and link against the library. If `fidoSSL` was installed in the standard locations, you might compile your application as follows:

```bash
gcc -o myapp myapp.c -lfidossl -lssl -lcrypto -lfido2 -ltinycbor -ljansson
```

Replace `myapp.c` with your source file's name and `myapp` with your desired executable name. Adjust compiler flags according to your project needs.

For manual linking or if `fidoSSL` and its dependencies are in non-standard locations, specify the paths explicitly:

```bash
gcc -o myapp myapp.c -I/path/to/fidossl/headers -L/path/to/fidossl/library -lfidossl -lssl -lcrypto -lfido2 -ltinycbor -ljansson
```

- Use `-I/path/to/fidossl/headers` to include the directory containing `fidoSSL`'s header files.
- Use `-L/path/to/fidossl/library` to add the directory containing the `fidoSSL` library file to the linker's search paths.
- Ensure these paths point to where `fidoSSL`'s library and headers reside on your filesystem.

### Usage

This section outlines the steps required to seamlessly incorporate `fidoSSL` into your project. Ensure you've met all [Prerequisites](#Prerequisites), including a basic understanding of OpenSSL in C, before proceeding.

The following comprehensive example demonstrates how to integrate the FIDO authentication extension within a client and server application, utilizing  OpenSSL for the setup. 

**Please note**: This example specifically focuses on illustrating how to add `fidoSSL` to your application. It intentionally skips intermediate steps related to general OpenSSL setup and usage, which should be implemented as per OpenSSL's documentation and best practices.

#### Client

```c
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <fidossl.h>

// Initialize your SSL context (ctx) as per your application's requirements
SSL_CTX *ctx;

// Set the minimum protocol version to TLS 1.3
if (!SSL_CTX_set_min_proto_version(ctx, TLS1_3_VERSION)) {
    // Handle error
}

// Load the client certificate and private key
if (SSL_CTX_use_certificate_file(ctx, "/path/to/client.crt", SSL_FILETYPE_PEM) <= 0 ||
    SSL_CTX_use_PrivateKey_file(ctx, "/path/to/client.key", SSL_FILETYPE_PEM) <= 0) {
    // Handle error
}

// Prepare FIDOSSL client options
FIDOSSL_CLIENT_OPTS *opts = malloc(sizeof(FIDOSSL_CLIENT_OPTS));
if (opts == NULL) {
    // Handle malloc error
}

// Modes are: FIDOSSL_REGISTER or FIDOSSL_AUTHENTICATE
opts->mode = FIDOSSL_REGISTER;

// Define a user and user display name.
// When in FIDOSSL_AUTHENTICATE mode, this setting is ignored.
opts->user_name = "alice";
opts->user_display_name = "Alice";

// In FIDOSSL_REGISTER mode, new fido keys are only enrolled if the client provides a valid ticket.
// Client and Server must be configured with the same ticket. The ticket is base64 encoded.
// When in FIDOSSL_AUTHENTICATE mode, this option is ignored.
opts->ticket_b64 = "y1v2BsTzi6baajWpU5WSDw6AYorx2MSDO1iVFSQC8VQ=";

// The PIN of the FIDO token.
opts->pin = "1234";

// Optional: Debug levels are: DEBUG_LEVEL_ERROR, DEBUG_LEVEL_VERBOSE, DEBUG_LEVEL_MORE_VERBOSE.
opts->debug_level = DEBUG_LEVEL_MORE_VERBOSE;

// Register the FIDOSSL extension with the SSL context
SSL_CTX_add_custom_ext(
    ctx,
    FIDOSSL_EXT_TYPE,
    FIDOSSL_CONTEXT,
    fidossl_client_add_cb,
    NULL,
    opts,
    fidossl_client_parse_cb,
    NULL
);

// Create an SSL object and proceed with the connection and TLS handshake as usual
// Remember to configure SNI and handle the connection appropriately

// If we want to register new fido keys, a second handshake is necessary.
if (opts->mode == FIDOSSL_REGISTER) {
  
    // Shutdown the first connection
    while (SSL_shutdown(ssl) != 1) {}
  
    // Create new SSL object out of the context object
    SSL_free(ssl);
    ssl = SSL_new(ctx);
  
  	// Connect the SSL object to the socket as usual. The socket should be kept open,
  	// so the new TLS handshake reuses the TCP connection. Don't forget to set the SNI.

    // Connect to server again
    if (SSL_connect(ssl) != 1) {
        // Handle error
    }

    // Shutdown SSL connection. It is not allowed to send data after a key enrollment
    while (SSL_shutdown(ssl) != 1) {}
}

// The caller is responsible for freeing the client options,
// the extension keeps no references, all data is copied.
free(opts);

// Remember to also free SSL and SSL_CTX objects, and close the network socket as usual
```

#### Server

```c
#include <openssl/ssl.h>
#include <openssl/err.h>
#include <fidossl.h>

// Initialize your SSL context (ctx) as per your application's requirements
SSL_CTX *ctx;

// Initialize the FIDOSSL server options structure. This structure configures the FIDO authentication
// parameters, including the relying party identifier, user verification requirements, and debug level.
FIDOSSL_SERVER_OPTS *opts = malloc(sizeof(FIDOSSL_SERVER_OPTS));

// Set the relying party identifier to the servers domain. This is a unique identifier for the FIDO 
// relying party.
opts->rp_id = "mydomain.com";

// Configure the relying party name to a more descriptive human-readable name, for example, 
// "My Domain Example".
opts->rp_name = "My Domain Example";

// Specify the base64 encoded ticket for the FIDO operation. This ticket is crucial for the 
// FIDO key registration process, acting as a shared secret between the client and server.
opts->ticket_b64 = "y1v2BsTzi6baajWpU5WSDw6AYorx2MSDO1iVFSQC8VQ=";

// Values for user verification are: REQUIRED, PREFERRED, DISCOURAGED
opts->user_verification = PREFERRED;

// Resident keys can be: REQUIRED, PREFERRED, DISCOURAGED
opts->resident_key = REQUIRED;

// Authenticator modes: CROSS_PLATFORM, PLATFORM
opts->auth_attach = CROSS_PLATFORM;

// How the authenticator is connected to the client: USB, NFC, BLE, INTERNAL
opts->transport = USB;

// Timeout for CTAP in [ms]
opts->timeout = 60000; // 1 Minute

// Optional: Debug levels are: DEBUG_LEVEL_ERROR, DEBUG_LEVEL_VERBOSE, DEBUG_LEVEL_MORE_VERBOSE.
opts->debug_level = DEBUG_LEVEL_MORE_VERBOSE;

// Register the `libfidossl` extension with the SSL context. This step is critical as it
// integrates FIDO authentication into the TLS handshake process.
SSL_CTX_add_custom_ext(
    ctx,
    FIDOSSL_EXT_TYPE,
    FIDOSSL_CONTEXT,
    fidossl_server_add_cb,
    NULL,
    NULL,
    fidossl_server_parse_cb,
    opts
);

// Configure the SSL context to request a client certificate. This configuration is essential for triggering
// the SSL_EXT_TLS1_3_CERTIFICATE event, which is necessary for the FIDO extension to operate correctly.
SSL_CTX_set_verify(ctx, SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT, no_verify_cb);

// Create and bind a listening socket to an IP and port. Listen for incoming connections on the socket.
// Accept a connection from a client, creating a new socket for this client.
// Associate the client socket with an SSL object and perform the TLS handshake.

// Shut down the SSL connection properly

// The caller is again responsible for freeing the server options,
// the extension keeps no references, all data is copied.
free(opts);
```

### Disclaimer

This codebase, including the `fidoSSL` library and its accompanying utilities, is a **proof of concept** and is primarily intended for **research and development purposes**. It has been made available to the public to foster understanding, experimentation, and further study into integrating FIDO authentication within the TLS 1.3 protocol using OpenSSL.

**It is strongly advised against using this codebase in production environments or in any application where security, stability, and reliability are critical.** While every effort has been made to adhere to best practices and standards, this project has not undergone the rigorous testing and security auditing that production-ready software demands. As such, it may contain vulnerabilities, bugs, or other issues that could potentially compromise security.

Users are encouraged to explore and experiment with the codebase in controlled, secure environments. Contributions, feedback, and discussions around improvements, security enhancements, and potential use cases are welcome, as they contribute to the ongoing development and understanding of secure authentication practices within the TLS ecosystem.

By using or referencing this codebase, you acknowledge and accept that you do so at your own risk, and you bear full responsibility for any consequences arising from its use.

### License

This project is licensed under the [BSD 2-Clause License](https://opensource.org/license/bsd-2-clause).

The BSD 2-Clause License is one of the simplest and most permissive free software licenses available. This license allows for nearly unrestricted freedom with the software, provided that two key conditions are met:

1. **Redistributions of source code must retain the above copyright notice, this list of conditions, and the following disclaimer.**
2. **Redistributions in binary form must reproduce the above copyright notice, this list of conditions, and the following disclaimer in the documentation and/or other materials provided with the distribution.**

This approach encourages wide adoption and utilization of the software while ensuring that credit is maintained with the original authors. It supports both private and commercial use, making the `fidoSSL` project suitable for a broad array of projects, from academic research to industry applications.

By adopting the BSD 2-Clause License, `fidoSSL` aims to foster an environment of collaboration and open innovation, allowing for enhancements, modifications, and integrations in a way that respects the creators' contributions while offering the community significant freedom to leverage and build upon the project.

For further details about the BSD 2-Clause License, you can refer to the official BSD License page.
