---
title: "Why encrypted protocols need three layers to stay in sync"
date: 2026-04-09
draft: false
tags: ["wire", "protocol", "bytes", "frame-detection", "byte-stuffing", "encryption"]
summary: "How two peers exchanging encrypted frames can synchronize with each other — and why byte stuffing exists."
---

Inventing new ad-hoc protocols can be very fun: making two remote
machines communicate over a channel, exchanging meaningful messages with each
other.

Professional tides recently brought me to build a raw custom protocol
from scratch, also ensuring that the traffic flowing in the wire was
encrypted end-to-end.

When I tackled the problem at first, I naively thought that what counts overall
is the protocol part itself — clearly defining the exchanged message structures,
the error codes in structured enums, and ultimately having the protocol
handshakes well designed with all related callbacks in the application layer.

The encryption part would "just" make the flowing data opaque to any reader in
the middle. And for that part, I told myself, I won't invent anything — I'll
integrate the best in the art for asymmetric key exchange. We don't rewrite
crypto; we use it from established standards.

I was not so wrong… but neither was I totally right. I learned it's not as
simple as it reads.

Let's step back to a clear-text data transfer first.

## A clear-text framing scenario

### The naive delimiter approach

When two peers share knowledge of the protocol, they can simply agree on one
special character (or a sequence of characters) to send on the wire to separate
messages from each other, and make the receiving peer split the stream
accordingly.

For example, many line-oriented protocols use `\n` (newline) or `\r\n` as a
frame delimiter — HTTP/1.1 headers, SMTP, Redis RESP. Others use a dedicated
control byte, like `0x7E` in HDLC or PPP.

The receiver accumulates bytes into a buffer, and every time the delimiter
appears it slices off the buffer up to that point as a complete frame. Another
component — usually called the protocol handler — then deserializes each byte
chunk into a message structure recognized by the protocol and passes it to the
application callback for action. If the chunk is unrecognized or malformed, it
purges it and takes other side actions (error notification, silent ignore, and
so on).

Readable and easy to debug with a packet sniffer... but..

### The problem: delimiter collision in the payload

There is a catch. What if the payload itself contains the delimiter byte?

Say both peers agree that `0x00` (null byte) terminates every frame. The sender
builds a frame and writes it to the wire followed by `0x00`. The receiver reads
bytes until it sees `0x00` and calls that a frame. All good — until one frame's
payload legitimately contains a `0x00` byte somewhere in the middle.

The receiver sees that byte, thinks the frame is done, and slices the buffer
prematurely. Downstream, the parsing crashes... data is malformed.
The two peers are now out of sync and the protocol results broken.

This is not a theoretical edge case! It artives all the time for binary
payloads: with files, numerical data or serialized structs, zero bytes
are common. You cannot just "pick a byte value supposing it won't appear
in the data" without restricting what data you want transmit.

With encrypted data this heuristic disappears entirely. Good encryption
produces output statistically indistinguishable from a uniform distribution —
every byte value from `0x00` to `0xFF` is equally likely. A multi-byte
delimiter only lends your parser a bit more time before crashing: a 4-byte
sequence has a 1-in-4-billion chance of appearing at any given offset, but
across millions of frames that probability accumulates toward certainty.

### Byte stuffing solves it

The standard fix is **byte stuffing** (also called byte escaping). The idea is:

1. Pick two special byte values: a **frame delimiter** (e.g. `0xC0`) and an
   **escape byte** (e.g. `0xDB`).
2. Before sending to the wire, scan the payload. Every time the delimiter byte
   appears, replace it with the two-byte sequence `[ESC, FLAG]`.
   Every time the escape byte itself appears, replace it with `[ESC, ESC]`.
3. Then append the real delimiter.
4. On the receiving end, reverse the substitution as bytes arrive: after an
   `ESC`, the next byte is all the time `FLAG` or `ESC`.

Now the delimiter byte can only ever mean "end of frame", any occurrence of
the delimiter value inside the payload has been transformed into something else
before hitting the wire (i.e. `[ESC, FLAG]`). The two peers stay in sync
regardless of payload content.

This is how PPP (Point-to-Point Protocol) works, defined in RFC 1662.

## Why encrypted data makes this harder

### Ciphertext looks like random bytes

Good symmetric encryption — AES in CTR mode, ChaCha20, anything AEAD — produces
output that is computationally indistinguishable from uniformly random bytes.
This is not an accident; it is a hard security requirement. If ciphertext had
detectable structure, an attacker could exploit that structure to leak
information about the plaintext.

The practical consequence is that in the encrypted stream, every byte value from `0x00`
to `0xFF` appears with roughly equal probability. Over a session with many
frames, any specific byte value will appear in ciphertext constantly.

### The framing layer must operate on ciphertext

Framing and encryption are two independent concerns, but they **interact**:

- Encryption makes the message content opaque.
- Framing tells the receiver *where* a message starts and ends.

The receiver must be able to find frame boundaries before it can decrypt
anything. So the framing mechanism has to work at the ciphertext level, on bytes
that look random.

## Two approaches to encrypted framing

### Length-prefix framing

The simpler and more widely used approach: prepend each frame with a fixed-size
field stating the exact byte length of the payload that follows.

```
[ length : 4 bytes | ciphertext : N bytes ]
```

The receiver reads exactly 4 bytes to know the length `N`, then reads exactly `N` bytes
for the payload ciphertext. No delimiter, no scanning, no stuffing needed. The
structure is self-delimiting.

This is what **TLS 1.3** does: the TLS record layer wraps each encrypted
fragment with a 3-byte header that includes a 2-byte length field. The receiver
reads the header, learns how many bytes to expect, and reads them at once without any scanning.

**Signal** follows the same principle, but delegates framing to the WebSocket
layer rather than implementing it at the application level. Encrypted payloads
are sent as binary WebSocket frames; WebSocket's own frame header (RFC 6455)
carries the length, so the receiver always knows exactly how many bytes to read
before handing the blob to the decryption layer. The application code in
libsignal never writes a length prefix explicitly, it encrypts with the Noise
protocol or the Double Ratchet, then passes the ciphertext to the WebSocket
client and this one handles the rest.

**WireGuard** does the same: each UDP packet carries an encrypted payload whose
length is implicit from the UDP datagram length — the IP/UDP stack provides the
framing for free at the network layer, and WireGuard just decrypts the contents.

The main design decision is the size of the length field: 2 bytes limits frames at
65 535 bytes, 4 bytes limits at ~4 GB (size must be chosen accordingly to what
fits best your protocol, and explicitly reject larger frames.

### Delimiter + byte stuffing

On character-oriented or byte-stream channels where a length prefix is
impractical — serial lines, legacy embedded links, some radio protocols (which was my case) — byte
stuffing remains the right approach: I picked a frame delimiter, stuff any
occurrence of that delimiter (and the escape byte) in the ciphertext before
transmitting, then appended the delimiter. Reversing the process on the other end.

The tradeoff compared to length-prefix framing:

| | Length-prefix | Byte stuffing |
|---|---|---|
| Overhead | Fixed (e.g. 4 bytes/frame) | Variable (0–100% of payload) |
| Scanning | None — read exact N bytes | Scan every byte |
| Resync on error | Scan for next valid header | Scan for next delimiter byte |
| Typical use | TCP-based encrypted protocols | Serial / character-oriented links |

For a TCP-based custom protocol, length-prefix is almost always the right
choice. TCP is a byte stream with no message boundaries — a single
`read()` can return half a message or three merged together. The length field
lets the receiver issue two bounded reads (header, then exactly N bytes) with
no scanning. Byte stuffing shines when you are working on a byte-at-a-time
channel where you cannot buffer ahead.

## Byte stuffing in detail

The algorithm uses exactly two reserved byte values:

- `FLAG` (frame delimiter): e.g. `0xC0`
- `ESC` (escape byte): e.g. `0xDB`

Before sending a frame:

1. Write `FLAG` to the wire (marks the frame start).
2. Walk every byte `b` of the payload:
   - If `b == FLAG`: emit `ESC`, then `FLAG`.
   - If `b == ESC`: emit `ESC`, then `ESC`.
   - Otherwise: emit `b` unchanged.
3. Write a final `FLAG` (marks the frame end).

The rule is simple: every `FLAG` or `ESC` that belongs to the payload is
preceded by an `ESC`. A bare `FLAG` — one not preceded by `ESC` — is always
and only a frame boundary.

Example — stuffing the 4-byte payload `[0xC0, 0x41, 0xDB, 0x42]`:

```
Input:   C0  41  DB  42
Stuffed: DB C0  41  DB DB  42
Framed:  C0  DB C0  41  DB DB  42  C0
         ^   ^^^^       ^^^^        ^
         |   ESC+FLAG   ESC+ESC    end FLAG
         start FLAG
```

Worst-case overhead is 2× (every byte in the payload is either `FLAG` or `ESC`).
In practice, for random ciphertext, overhead is around `2/256 ≈ 0.8%` per frame
on average — each of the two reserved values appears roughly once every 128
bytes and costs one extra byte.

## Byte unstuffing

The receiver runs a simple two-state machine as bytes arrive:

- **NORMAL state**: accumulate bytes into the frame buffer.
  - `FLAG` received -> the buffer holds a complete frame; deliver it to the upper-layer and reset.
  - `ESC` received -> transition to the ESCAPE state.
  - Anything else -> append to buffer.
- **ESCAPE state**: the next byte is an escaped value.
  - `FLAG` received -> append `FLAG` to buffer; transition back to the NORMAL state.
  - `ESC` received -> append `ESC` to buffer; transition back to the NORMAL state.
  - Anything else -> protocol error (invalid escape sequence); discard the
    current frame, back to the NORMAL state.

```
         ┌─────────────────────────────────────┐
         │                                     │
         ▼                                     │  other byte -> buffer
    ┌─────────┐   ESC     ┌──────────┐         │
    │ NORMAL  │─────────▶│  ESCAPE  │─────────┘
    └─────────┘           └──────────┘
         │                    │
   FLAG  │               FLAG │ -> buffer FLAG
         │                ESC │ -> buffer ESC
         ▼              other │ -> protocol error
   deliver frame
```

On any error the machine returns to NORMAL and discards the in-progress frame.
The next `FLAG` it encounters, one not preceded by `ESC`, resets the
buffer and starts a fresh frame. That is the resync: no out-of-band signal
needed, the stream **self-corrects** at the next delimiter.

Error conditions worth handling explicitly:

- **Unknown escape sequence**: discard an in-progress frame, log a protocol
  error, remain synchronized — the next `FLAG` will start a fresh frame.
- **Frame too large**: if the buffer grows beyond your declared maximum frame
  size before seeing a `FLAG`, something is wrong (stuffing error or desync).
  Better to discard and reset the communication.
- **Truncated frame at connection close**: if the connection drops in the
  miffle of a frame, discard the partial buffer.

## Frame structure design

Whether you use length-prefix or byte stuffing for framing, the content of each
frame deserves a careful design. A reasonable structure for an encrypted custom
protocol:

```
┌────────────┬────────────┬─────────────────────┬────────────┐
│  magic     │  seq_no    │      ciphertext     │  auth_tag  │
│  2 bytes   │  4 bytes   │       N bytes       │  16 bytes  │
└────────────┴────────────┴─────────────────────┴────────────┘
```

**Magic word** — a fixed 2-byte value (e.g. `0xBEEF`) at the frame header,
for resynchronization in the length-prefix framing scenario. In this
case, indeed, after a desync event, the receiver can
scan the incoming stream for the magic value and attempt to resume parsing from
there.
With byte stuffing, the magic word is largely redundant: FLAG already provides
resync and AEAD authentication will reject any garbage frame.
The magic word is most useful with length-prefix framing instead, where
there is no delimiter to scan for after losing sync.

**Sequence number** — a **monotonically** incrementing counter per sender. The
receiver uses it to detect replayed frames and reordered delivery.
This counter must be incremented on every sent frame; frames must be rejected,
whose sequence number is not strictly greater than the last accepted one.

**Ciphertext** — the encrypted message payload. I used an AEAD cipher
(ChaCha20-Poly1305). The authentication tag it produces covers both
the ciphertext and any associated data you want to authenticate, typically the
sequence number, so replayed frames are caught.

**Auth tag** — 16 bytes produced by the AEAD cipher. The receiver verifies this
before doing anything with the payload. If verification fails, the frame
is **silently** discared (without sending back any detailed error, as
that can leak information to an active attacker).

## Encrypt-then-frame vs frame-then-encrypt

The order of operations matters and was surprisingly easy to get wrong the first
time.

**Frame-then-encrypt**: that's building the full plaintext frame
(header + payload), then encrypt the entire thing.
The problem here is the receiver cannot know the length of the ciphertext before decrypting, so it would need
an outer framing mechanism on the ciphertext.

...instead...

**Encrypt-then-frame**: this means encrypting the payload first, then wrap the ciphertext
in a frame (length prefix, or stuffing + delimiter).
The framing fields (length, magic, sequence number) travel in the clear and are
effectively used to convey communication and resync in case of error.

This is what TLS 1.3, WireGuard,
and Signal all do. It works very well because:

1. The receiver reads the clear-text framing fields to know how many ciphertext
   bytes to expect.
2. It reads exactly that many bytes.
3. It decrypts and verifies the auth tag.
4. It delivers the plaintext to the application.

The clear-text framing fields leaks one piece of information: the length field
reveals the approximate plaintext size, even if the content is opaque. This is a
known aside. TLS 1.3 addresses it with optional record padding — the
sender can pad the plaintext to a round size before encrypting, hiding the true
payload length. Signal pads certain message types for the same reason. If payload
length is sensitive in your protocol, consider doing the same.

## One frame, one message — or more?

So far I've implicitly assumed a 1:1 relationship: one frame carries
exactly one serialized message. The sender never starts a new message
mid-frame;
the receiver dispatches exactly one message per successfully decrypted frame.
This is the simplest design and a perfectly valid choice for a custom protocol.

But it leaks information. Of course, an observer watching the wire cannot read the content,
but they can still see frame sizes and timing. If each frame maps to one message,
frame size correlates directly with message size, and message timing is exposed
in the clear. For many protocols that is acceptable. For a privacy-sensitive one,
it is not.

### Coalescing messages inside the ciphertext

By the way, the encrypt-then-frame ordering opens a useful opportunity: the plaintext that
goes into the AEAD cipher does not have to be a single message. It can be a
batch of several concatenated messages, or a real message padded with dummy
bytes, or both.

```
[ message A | message B | message C | padding ] <- plaintext
                     │
                     ▼
             [ encrypt with AEAD ]
                     │
                     ▼
   [ length | ciphertext + auth tag ]           <- wire frame
```

An observer sees one frame of a certain size. Whether that encodes one large message,
three small ones, or a single real message with noise appended is invisible from
the outside. This technique — sometimes called **message coalescing** or
**batching** — hides both message size and message count within a time window.

Signal does exactly this for attachments: [`PaddingInputStream`](https://github.com/signalapp/Signal-Android/blob/main/lib/libsignal-service/src/main/java/org/whispersystems/signalservice/internal/crypto/PaddingInputStream.java)
wraps the plaintext stream and appends zero bytes up to the next bucket in a
geometric progression (`1.05^n`, minimum 541 bytes) before the stream is passed
to encryption in [`SignalServiceMessageSender`](https://github.com/signalapp/Signal-Android/blob/main/lib/libsignal-service/src/main/java/org/whispersystems/signalservice/api/SignalServiceMessageSender.java#L825).
An observer can tell which bucket an attachment falls in, but not its exact size.

### Inner framing inside the plaintext

The consequence is that the decrypted plaintext is now a stream, not a single
blob. The receiver needs to know where each message starts and ends within it.
That requires a second, inner framing layer — and this one operates on trusted,
already-authenticated data. There is no adversary at this layer, so it can be
simple: for example a small length prefix per message.

```
[ len_A : 2 bytes | message A | len_B : 2 bytes | message B | ... ]
```

Thus, the receiver decrypts the outer frame, verifies the auth tag,
then walks the inner stream parsing messages until the buffer is exhausted.
Padding bytes at the end can be marked with a reserved message type (e.g. type
`0x00` for padding) so the receiver knows to ignore them.

The two framing layers serve different purposes and can be designed
independently:

| Layer | Operates on | Purpose |
|---|---|---|
| Outer (wire) | Ciphertext | Frame boundary detection, resync |
| Inner (app) | Plaintext | Message boundary, batching, padding |

## Error detection and recovery

The primary desync signal is an AEAD authentication failure, i.e. the decrypted
bytes do not authenticate. A secondary indicator is a malformed header: an
unrecognized magic word, a sequence number wildly out of range, or a declared
length exceeding the maximum frame size. Either should trigger a desync
response, not a transient error handler.

For recovery, it's useful maintaining a counter of consecutive desync events.
Below the threshold, attempting local resync (scanning for the next FLAG
or magic word) is valuable. Above
it, better reset and rehandshake, as something sounds systematically wrong (scanning
forward is just burning CPU power, while peers probably talk past each other).

## The whole picture

Putting it all together, a sender's pipeline looks composed by three layers, each with a distinct responsibility:

1- **Outer framing**: find the ciphertext blob on the wire, surviving desynchronization.

2- **Encryption**: authenticate and decrypt, rejecting anything tampered with.

3- **Inner framing**: recover individual messages from the plaintext batch,
  hiding their sizes and counts from the outside.

```
── LAYER 3: inner framing ───────────────────────────────────
message A   message B   message C
    │           │           │
    └─────┬─────┘           │
          │    (+ padding)  │
          ▼                 │
[ prefix each message with its length ]
          │
── LAYER 2: encryption ──────────────────────────────────────
          ▼
[ encrypt plaintext batch with AEAD -> ciphertext + auth tag ]
          │
── LAYER 1: outer framing ───────────────────────────────────
          ▼
[ build outer frame: magic | seq_no | ciphertext | auth_tag ]
          │
          ▼
[ apply length prefix  OR  byte stuff ]
          │
          ▼
         wire
```

And the receiver's pipeline, symmetrically:

```
         wire
          │
── LAYER 1: outer framing ───────────────────────────────────
          ▼
[ read length field + N bytes  OR  scan for FLAG + unstuff ]
          │
          ▼
[ check magic, extract seq_no, reject replays ]
          │
── LAYER 2: encryption ──────────────────────────────────────
          ▼
[ decrypt + verify auth tag -> on failure: discard, count ]
          │
── LAYER 3: inner framing ───────────────────────────────────
          ▼
[ walk inner stream: extract messages by length prefix, discard padding ]
          │
    ┌─────┴─────┐
    ▼           ▼
message A   message B  ...
    │           │
    ▼           ▼
[ dispatch each to protocol handler ]
```

Getting all three in that order, is what finally made a robust encrypted
protocol on the wire for me.

## Summarizing the solution in Rust

The protocol had to run over a serial port.

I instantiated two dedicated threads, one for sending, one for receiving. Each thread owning its half of the port.

Messages flow through `std::sync::mpsc` channels between the application
and the two I/O loops.

Tokio would have been overkill here: the target was a resource-constrained
embedded Linux board, and in thie context two blocking threads with standard
channels keep the dependency minimal.

`impl Read + impl Write` abstracts the wire. In production
the port was backed by a UART-connected transceiver, exposed by the OS
as a serial device.
The same setup would apply to narrow-bandwidth RF links where the radio
module exposes a transparent serial interface (this is common with LoRa
and FSK transceivers for exemple).

The frame layout is `[ seq: 4 bytes | ciphertext + auth_tag ]`. The sequence
number doubles as the AEAD nonce (padded to 12 bytes) and as authenticated
associated data, so a replayed or reordered frame fails both the sequence check
and the auth tag verification.

```rust
use chacha20poly1305::{aead::{Aead, KeyInit, Payload}, ChaCha20Poly1305, Nonce};
use std::{io::{Read, Write}, sync::mpsc};

const FLAG: u8 = 0xC0; // the chosen inter-frame separator
const ESC:  u8 = 0xDB; // the chosen escape caracter

fn byte_stuff(src: &[u8]) -> Vec<u8> {
  let mut out = vec![FLAG];
  for &b in src {
    match b {
      FLAG => out.extend_from_slice(&[ESC, FLAG]),
      ESC  => out.extend_from_slice(&[ESC, ESC]),
      _    => out.push(b),
    }
  }
  out.push(FLAG);
  out
}

fn byte_unstuff(src: &[u8]) -> Option<Vec<u8>> {
  let mut out = Vec::new();
  let mut i = 0;
  while i < src.len() {
    match src[i] {
      ESC => match src.get(i + 1) {
        Some(&b @ (FLAG | ESC)) => { out.push(b); i += 2; }
        _ => return None,
      },
      b => { out.push(b); i += 1; }
    }
  }
  Some(out)
}

fn nonce(seq: u32) -> Nonce {
  let mut n = [0u8; 12]; // ChaCha20-Poly1305 nonce is 96 bits
  n[8..].copy_from_slice(&seq.to_be_bytes()); // seq in the last 4 bytes
  Nonce::from(n)
}

fn decrypt_frame(
  frame: &[u8], cipher: &ChaCha20Poly1305, last_seq: &mut Option<u32>
) -> Option<Vec<u8>> {
  if frame.len() < 4 + 16 { return None; } // 4 seq + 16 AEAD auth tag (minimum)

  let seq = u32::from_be_bytes(
    frame[..4].try_into().ok()?
  ); // first 4 bytes: seq

  if last_seq.is_some_and(|p| seq <= p) { return None; }

  let pt = cipher.decrypt(
    &nonce(seq),
    Payload { msg: &frame[4..], aad: &frame[..4] }
  ).ok()?; // frame[4..] = ciphertext + auth tag  |  frame[..4] = seq as AAD

  *last_seq = Some(seq);
  Some(pt)
}

fn send_loop(mut w: impl Write, rx: mpsc::Receiver<Vec<u8>>, cipher: ChaCha20Poly1305) {
  let mut seq: u32 = 0;
  for msg in rx {
    let aad = seq.to_be_bytes();
    let ct = cipher.encrypt(&nonce(seq), Payload { msg: &msg, aad: &aad }).unwrap();

    let mut frame = aad.to_vec();
    frame.extend_from_slice(&ct);

    w.write_all(&byte_stuff(&frame)).unwrap();
    seq = seq.wrapping_add(1);
  }
}

fn recv_loop(mut r: impl Read, tx: mpsc::Sender<Vec<u8>>, cipher: ChaCha20Poly1305) {
  let (mut buf, mut b) = (Vec::new(), [0u8; 1]);
  let mut last_seq: Option<u32> = None;
  let mut desync_count: u32 = 0; // resync: count consecutive bad frames
  const DESYNC_LIMIT: u32 = 5;

  loop {
    if r.read_exact(&mut b).is_err() { break; }

    match b[0] {
      FLAG if buf.is_empty() => {}
      FLAG => {
        let ok = byte_unstuff(&buf)
          .and_then(|frame| decrypt_frame(&frame, &cipher, &mut last_seq))
          .map(|pt| { let _ = tx.send(pt); })
          .is_some();

        // resync: escalate to rehandshake after repeated failures
        if ok { desync_count = 0; }
        else {
          desync_count += 1;
          if desync_count >= DESYNC_LIMIT { break; }
        }

        buf.clear();
      }
      _ => buf.push(b[0]),
    }
  }
}
```

In `main()`, the two loops are started with `thread::spawn(move || send_loop(...))` and
`thread::spawn(move || recv_loop(...))`, each taking ownership of its port half and
cipher instance via the `move` closure.

## Code refs

### rustls — encrypt-then-frame in TLS 1.3

The clearest entry point is the send pipeline in
[`rustls/src/conn/send.rs`](https://github.com/rustls/rustls/blob/main/rustls/src/conn/send.rs)
(lines 126–128):

```rust
self.sendable_tls.append(
    self.encrypt_state.encrypt_outgoing(m).encode()
);
```

`encrypt_outgoing(m)` seals the plaintext fragment with AEAD and returns an
`EncodedMessage<OutboundOpaque>` — ciphertext only, no header yet. `.encode()`
then writes the 5-byte TLS record header, including the 2-byte big-endian
length of the now-encrypted payload.

That length-writing step lives in
[`rustls/src/crypto/cipher/messages.rs`](https://github.com/rustls/rustls/blob/main/rustls/src/crypto/cipher/messages.rs)
(lines 191–197):

```rust
pub fn encode(self) -> Vec<u8> {
    let length = self.payload.len() as u16;
    let mut encoded_payload = self.payload.0;
    encoded_payload[0] = self.typ.into();
    encoded_payload[1..3].copy_from_slice(&self.version.to_array());
    encoded_payload[3..5].copy_from_slice(&(length).to_be_bytes());
    encoded_payload
}
```

The 5-byte header slot is pre-allocated by `OutboundOpaque::with_capacity()`
(lines 271–275) and stays zeroed until `encode()` fills it in — after
encryption, never before.

### Signal — WebSocket as the framing layer

Signal does not write a length prefix at the application level. Instead it
encrypts with the Noise protocol (for attestation connections) or the Double
Ratchet (for messages), then hands the ciphertext to the WebSocket client as a
binary frame:

```rust
// rust/net/infra/src/ws/attested.rs
pub async fn send_bytes(&mut self, plaintext: &[u8]) -> Result<(), AttestedConnectionError> {
    let message = client_connection.send(plaintext)?;
    Ok(ws_client.write(message).await?)
}
```

([`rust/net/infra/src/ws/attested.rs`](https://github.com/signalapp/libsignal/blob/main/rust/net/infra/src/ws/attested.rs))

The framing is handled by the WebSocket layer itself. Each WebSocket frame
carries a length field in its header per
[RFC 6455 §5.2](https://datatracker.ietf.org/doc/html/rfc6455#section-5.2),
so the receiver always knows how many bytes to read before passing the blob to
decryption. The encrypt-then-frame principle holds — it is just the WebSocket
protocol that provides the "frame" part.
