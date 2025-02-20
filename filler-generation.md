# Filler Generation in Onion Packet Construction

![The Muppet Show: Rowlf - "I Never Harmed An Onion"](https://i.makeagif.com/media/2-19-2025/9hx5OB.gif)

*I never harmed an onion, so why should they make me cry? - Rowlf the Dog*

Here's my take on explaining the filler generation because it's the part of the onion packet that makes people cry.

## Why do we need the filler in the first place?

The importance of the filler is more apparent when you're unwrapping the onion packet.

In lightning onion packet construction, it's important for the onion payload to maintain its size of 1300 bytes. However, this becomes a problem when layers (read: hops) of the payload are being removed from the payload as the onion is being peeled/read and so you end up with an onion payload that is `1300 - hop_payload_length` in length as you go on.

To deal with this, a zero-byte chunk that is 1300 bytes long is appended to the onion payload before decryption and peeling. At this point, the onion payload is `onion payload + 1300 chunk of zero bytes`. Therefore, a random byte stream that is 2600 bytes long is used to decrypted this. After the decryption, the relevant onion payload is peeled off and the end of the onion payload is chopped off to bring it back to 1300-bytes long.

Without the filler, there would be a problem decrypting the onion payload because the encryption of the onion payload would not be uniform.

## How is the filler generated?

*Note: You need to keep track of the length of the hop payloads that make up the onion payload and disregard the final hop.*

- Initialise a filler that is filled with zero bytes. Its length should match that of the hop payloads that make up the onion payload EXCEPT the final hop.

- Keep a record of the hop payload lengths that **have been processed** during the filler generation process.

- Loop through each hop payload EXCEPT the final hop payload and:
    
    - Generate a random byte stream with the appropriate key and shared secret that is 2600 bytes long (this is to match what it would be like during the decryption process).
    
    - Keep track of what part of the filler needs to be encrypted (the target range). 
    
    *But where are we starting from?*
    
    Imagine taking a few steps back before running forward and then taking a few more steps back from your starting point before running again. That's what we'll be doing here.
    
    We move backwards along the byte stream, starting from the middle of the byte stream *(remember, the byte stream is 2600 bytes long)*. The starting point for encryption is identified by moving back from the middle by the lengths of the hop payloads that have already been accounted for in the filler generation process. Therefore, if it's the first iteration of the loop, that means we haven't processed any hop payloads yet so the starting point is at 1300.

    *But at what point are we going to stop?*

    This part is easy. We are stopping at 1300 + the current payload length.

    - Next, we encrypt the target range of the filler byte from the start point we identified to the stop point.

    - After encryption, we can now say we have processed that hop payload length so we add it to the record of the lengths we have processed.

When the loop ends, we've got our filler.
