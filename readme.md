# Onion Routing in Bitcoin Lightning Notes

- Session key = the private key in an ephemeral key pair; used to derive other ephemeral keys along each hop

- HMAC (Hash-based Message Authentication Code) = a cryptographic hash is used along with the secret key to generate the code. It's only derived by the Sender. Each hop uses this to confirm that there was no tampering. Each HMAC for each hop is present in the onion.

- XOR (Exclusive OR) = Bitwise operation where the result is 1 if only one of the two bits operated on is 1. The result of XORing a packet with itself is 0. If you XOR the 0, the result is the original packet. Using a zero-byte stream does not change the original value. Therefore, XORing the original value with a zero-byte stream results in the original value. This is useful in debugging. However, using a random-byte stream results in encrypted data. Therefore, XORing the random-byte stream with the encrypted data will result in the original value.

- The size of an onion payload is 1300 bytes. However, the size of the complete onion packet is 1366 bytes and it's broken down as follows:

    - Version byte (1 byte)
    - Sender's compressed session public key (33 bytes)
    - Onion payload (1300 bytes)
    - HMAC (32 bytes)

- Pseudo-random byte stream generation uses the ChaCha20 stream cipher, which takes the appropriate key type and a 12-byte zero nonce as inputs to produce the random stream.

## Nature (make up) of onion payloads put in the onion packet

Onion payloads put in the onion packet are constituted as follows:

[ length ][ payload ][ HMAC ]

Where:
- Length = total length of payload
- Payload = contains routing info
- HMAC = 32 bytes; prevents tampering

## Steps in creating the onion payload

(Note: Start from the final recipient and work your way backwards to the beginning)

- Generate a 1300 byte random Chacha20 byte stream using the session key and the pad key(padding)

- Slide the last hop's payload and chop off the trailing section to maintain the onion payload length (1300 bytes). The last hop's HMAC is fake since it doesn't have to be verified by anyone. 

- Use sender's shared secret with recipient (last hop) together with `rho` to generate 1300 random Chacha20 byte stream and use that random byte stream to encrypt the whole onion payload (XORing it). 

- If you're working on the last hop, you'll need to include the filler as part of the onion payload.

- The HMAC for this leg of the onion payload (not the aforementioned "fake" HMAC of the recipient) is derived using the `mu` key (which itself was derived from the `mu` constant and the shared secret) and the packet contents. The result will be used as the next hop's HMAC.

- The process for the next hops follows the same steps as above but this time the shared secret used is the one between the sender and the next hop.

- The process continues in the same way until you get to the beginning. The final onion payload is sandwiched between the packet version byte and the sender's first ephemeral public key and the last HMAC that was earlier derived.

## How is the shared secret calculated?

The shared secret is derived using the Elliptic Curve Diffie-Hellman key exchange. Hops in a transaction are able to use their private keys along with the peer node/hop's public key to derive a common shared secret that would be used to generate other cryptographic keys (eg. mu, rho, pad, um, ammag). This allows the nodes/hops to keep their private keys confidential. I like to think of it like this:

`shared_secret = SHA256(ECDH(Ephemeral_key * node_public_key))`

## What are `rho` and `mu`?

- `Rho` and `mu` are constants that are used for encryption and authentication, respectively. 

- The derivation of the `rho` and `mu` keys is done for each hop.

## Peeling (reading) the onion packet

(Note: We're working our way from the beginning [ Sender ] to the receipient)

- Validate the HMAC at the end using the `mu` (that's derived from the shared secret of the sender and the hop) and the onion package contents. Validation is through a hash and not XOR.

- The shared secret is used to derive the `rho`, which is used to generate a 2600 random byte stream. 

- 1300 zero byte stream is appended to the end of the onion payload.

- The whole 2600 random byte stream is XOR'd with the padded payload. This works because the first 1300 bytes out of the 2600 decrypts the encrypted payload (resulting in 1) while the XOR of the last 1300 bytes of the random stream with the 1300 zero byte stream leaves it as is.