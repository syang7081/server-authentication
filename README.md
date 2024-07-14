# Server Authentication - Certificate Pinning Enhancement
## Method
Server certificate pinning and host name verification are widely used to authenticate remote servers by mobile and desktop applications. This technique requires that an application must contain the server certificate (or its intermediate/root certificate) or its public key hash when the application is released. When the server certificate expires or updated or revoked, certificate pinning will fail and therefore the application cannot trust the server, and is not allowed to continue to work unless a new version of the application is released. In order to make an application continue to work in such scenario, a new method is designed to authenticate the sever. The data flow of this method is described as follows:

(1) The application starts a connection with the remote server through TLS or HTTPS or WebSocket, performs certificate pinning and host name verification during the TLS handshaking process, using a valid server certificate it contains,

(2) after step (1) is finished successfully, the application creates a server authentication code, such as a UUID, and send this server authentication code to the server, together with the application instance ID. The application instance ID should be unique. Client stores this pair of data into its secure storage.

(3) The server receives the server authentication code and the application instance ID, and stores this pair of data into its secure storage, such as a database.

(4) The client closes the connection with the remote server.

(5) If certificate pinning and host name verification fail due to server certificate update or any other reasons, the client will execute the following steps (5a) - (5g) to authenticate the server:

  (5a) The application starts a connection with the remote server with TLS or HTTPS or Websocket,
  
  (5b) On the application side, in the TLS handshaking process, use the TLS extensions to insert client custom data into the ClientHello message, the custom data are: 
  
    - Application instance ID,
    
    - A nonce
  
  (5c) On the server side, the server inserts its custom data into the TLS handshaking ServerHello message. The server will first parse the client custom data it receives from the client, get the server authentication code from its secure storage based on the application instance ID in the client custom data, calculate the hash (SHA256, for example) of the client custom data and the server authentication code. This hash is the server custom data in the ServerHello message to be sent to the client.
  
  (5d) The application receives and parses the server custom data in the TLS ServerHello message, 
  
  (5e) The application calculates the hash of the client custom data plus server authentication code it saved in its storage,
  
  (5f) The application compares the hash calculated in step (5e) with the server custom data. If they match, the server is authenticated together with the next step,
  
  (5g) The application verifies the host name in the TLS handshaking is valid, the same as in the updated certificate.


## NOTES:

1. The protocol defined from (5a) to (5g) only guarantees the server identity is verified and authenticated, in order to protect the client and server communication from Man-in-the-Middle (MiTM) attack, the hash calculated in step (5c) should be signed by the server using the private key of the updated server certificate. This digital signature should be sent to the client in (5d). The client should verify it in step (5f) using the public key of the leaf certificate in the certificate chain interecepted during the TLS handshaking.

2. If a HTTPS library doesn't have APIs to insert/parse TLS extension data during TLS handshaking, we may implement application or HTTP level algorithm to validate the server authentication code as described above. See note 5.

3. For client and server communication using WebSocket, a specific message type can be defined to exchange data to authenticate the server. Right after a Websocket is established, client can authenticate the server by following steps (5a) - (5f), plus digital signature verification described in Note 1. TLS extension support by WebSocket libraries is, therefore, not required.

4. SSO refresh token can be used to authenticate resource servers, see project wiki page: https://github.com/syang7081/server-authentication/wiki.

5. Combining note 1 and steps (5f) - (5g), one can find that if the client can successfully verify the digital signature of the server authentication code, then the server does own the updated certficate and the certificate has the correct private key. The updated server certificate can therefore be trusted and be pinned in all communications after. In fact, this approach tells us that one can create a special server API to verify whether an updated certificate can be promoted as trusted dynamically without releaseing a new version of a client. The data flow of this process is illustrated in the following diagram:
  ![Data flow of Server certificate verification and promotion]( https://github.com/syang7081/server-authentication/blob/main/doc/images/server-certificate-verification-promotion.png)

## Code Example 

The following client and server code based on OpenSSL demonstrates how steps (5a) - (5f) in the above protocol work:

TlsClient.cpp - A simple client supports TLS and TLS custom extension.

TlsServer.cpp - A simple server supports TLS and TLS custom extension.
