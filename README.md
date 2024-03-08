# nethernet

NetherNet is a new WebRTC-based transport layer protocol that aims to be a replacement for RakNet on Minecraft: Bedrock
Edition. It has been in development for the past few years, but is now starting to be rolled out to the public, starting
on LAN and Xbox Live games. It cannot currently be used over direct connections.

The protocol is currently not very well documented, so this covers everything needed to implement it. Keep in mind that
since this is a new protocol, it is subject to change at any time and thus this document may become outdated. All
information here is from reverse engineering `v1.20.50` of the game.

## Data formats

All numbers are encoded little-endian. Strings are length prefixed, with either a `uint8` or `uint32` integer.

## LAN discovery

LAN discovery is done on the `7551` port. Clients send a request packet to the broadcast address of the network. Servers
broadcast back a response packet with their name, game mode, and other information.

Discovery packets are encrypted and are prefixed with a checksum. The encryption algorithm itself is `AES-ECB` with
PKCS5/7 padding. The encryption key is the `SHA-256` hash of `0xdeadbeef` encoded as a 64 bit little-endian integer.
The checksum is a `SHA-256` `HMAC` of the unencrypted packet, using the same key as for encryption.

Each discovery packet starts with the packet length (`uint16`), packet type (`uint16`), and sender ID (`uint64`). After
that, there is an 8-byte padding, followed by the actual packet data.

The sender ID is just a sort of session ID, a random number generated by the client to identify itself.

```go
func encodeDiscoveryPacket(senderID uint64, pk discovery.Packet) ([]byte, error) {
    buf := new(bytes.Buffer)
    _ = binary.Write(buf, binary.LittleEndian, pk.ID())
    _ = binary.Write(buf, binary.LittleEndian, senderID)
    _ = binary.Write(buf, binary.LittleEndian, make([]byte, 8))
    pk.Write(buf)

    length := len(buf.Bytes())
    payload := append([]byte{byte(length), byte(length >> 8)}, buf.Bytes()...)
    data, err := encryptECB(payload)
    if err != nil {
        return nil, fmt.Errorf("error encrypting: %w", err)
    }

    hm := hmac.New(sha256.New, key[:])
    hm.Write(payload)
    data = append(hm.Sum(nil), data...)
    return data, nil
}
```

There are three discovery packets that are currently used:

- `DiscoveryRequestPacket` (ID `0x00`)
- `DiscoveryResponsePacket` (ID `0x01`)
- `DiscoveryMessagePacket` (ID `0x02`)

`DiscoveryRequestPacket` does not have any additional data. It is broadcasted by clients to look for servers on LAN.

`DiscoveryResponsePacket` sends a `ServerData` payload, which is a uint32 prefixed string. The string contains hex
encoded data. The structure of the hex decoded data is as follows:

- Version (`uint8`)
- Server name (`string`, `uint8` prefix)
- Level name (`string`, `uint8` prefix)
- Game type (`int32`)
- Player count (`int32`)
- Max player count (`int32`)
- Editor world (`bool`)
- Transport layer (`int32`)

It is used by servers to respond to discovery requests.

`DiscoveryMessagePacket` is used for negotiating the ICE candidates and everything needed for the connection. It is
sent by both the client and the server until the WebRTC connection has been established. It is structured as follows:

- Recipient ID (`uint64`)
- Data (`string`)

## Xbox Live

This section expects you to have some knowledge of how Xbox Live session directory works. This will not cover how to
connect and authenticate to it properly.

The Xbox Live session directory on latest versions will return a `WebRTCNetworkId` in `SupportedConnections`, like so:

```json
"SupportedConnections": [
  {
    "ConnectionType": 3,
    "HostIpAddress": "",
    "HostPort": 0,
    "WebRTCNetworkId": XXXXXXXXXXXXXXXXXXX
  }
]
```

This WebRTC network ID is effectively the friend's session ID, used to connect to them.

The connecting client takes this sender ID and establishes a WebSocket connection to
`wss://signal.franchise.minecraft-services.net/ws/v1.0/signaling/XXXXXXXXXXXXXXXXXXX`, where the X's are the client's
own session ID. This WebSocket is authenticated with the user's `MCToken`, which can be obtained from PlayFab. Keep in
mind that this token must also have the treatment overrides to use this signaling server.

Once connected, the server immediately sends back credentials for STUN and TURN servers that can be used if needed.
The client is then expected to set up their WebRTC client with these servers and then the WebRTC connection can be
negotiated.

## WebRTC negotiation
Before the WebRTC connection is made, the details are negotiated with the messages sent either over the LAN
`DiscoveryMessagePacket` or over the signaling server.

Each message is structured as follows:
`MESSAGETYPE CONNECTIONID DATA`

There are three message types used for WebRTC negotiation:
- `CONNECTREQUEST`
- `CONNECTRESPONSE`
- `CANDIDATEADD`

The connection ID is a unique ID for each connection. For a given negotiation, all messages will have the same ID.

`CONNECTREQUEST` just contains the SDP offer from the client. The server responds with a `CONNECTRESPONSE` containing
the SDP answer. After that, the client sends `CANDIDATEADD` messages with its ICE candidates. Once it has sent around
three, the server will send it's own candidates and the connection will attempt to be established.

In NetherNet, the client connecting will act as the ICE controller, and the server will act as the ICE agent.

Below is an example of a client's SDP offer:

```go
	sdp.SessionDescription{
		Origin:      sdp.Origin{Username: "-", SessionID: rand.Uint64(), SessionVersion: 0x2, NetworkType: "IN", AddressType: "IP4", UnicastAddress: "127.0.0.1"},
		SessionName: "-",
		TimeDescriptions: []sdp.TimeDescription{
			{},
		},
		Attributes: []sdp.Attribute{
			{Key: "group", Value: "BUNDLE 0"},
			{Key: "extmap-allow-mixed", Value: ""},
			{Key: "msid-semantic", Value: " WMS"},
		},
		MediaDescriptions: []*sdp.MediaDescription{
			{
				MediaName: sdp.MediaName{
					Media: "application",
					Port: sdp.RangedPort{
						Value: 9,
					},
					Protos:  []string{"UDP", "DTLS", "SCTP"},
					Formats: []string{"webrtc-datachannel"},
				},
				ConnectionInformation: &sdp.ConnectionInformation{
					NetworkType: "IN",
					AddressType: "IP4",
					Address: &sdp.Address{
						Address: "0.0.0.0",
					},
				},
				Attributes: []sdp.Attribute{
					{Key: "ice-ufrag", Value: iceParams.UsernameFragment},
					{Key: "ice-pwd", Value: iceParams.Password},
					{Key: "ice-options", Value: "trickle"},
					{Key: "fingerprint", Value: fmt.Sprintf("%s %s", fingerprint.Algorithm, fingerprint.Value)},
					{Key: "setup", Value: "actpass"},
					{Key: "mid", Value: "0"},
					{Key: "sctp-port", Value: "5000"},
					{Key: "max-message-size", Value: strconv.Itoa(int(sctpCapabilities.MaxMessageSize))},
				},
			},
		},
	}
```

The following is an actual CONNECTREQUEST sent by a client, some elements replaced with `<size>`:

```
CONNECTREQUEST <random connection ID> v=0
o=- <random ID> 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
a=extmap-allow-mixed
a=msid-semantic: WMS
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:<4 characters>
a=ice-pwd:<24 characters>
a=ice-options:trickle
a=fingerprint:sha-256 DB:23:<28 hex encoded bytes>:A1:D9
a=setup:actpass
a=mid:0
a=sctp-port:5000
a=max-message-size:<integer>
```

Effectively the same thing is done for the SDP answer, except the `setup` attribute is set to `active` instead of
`actpass`.

`CANDIDATEADD`'s data follows the standard ICE candidate string format. An example of one is below:
```
CANDIDATEADD <connection ID> candidate:XXXXXXXXXX 1 udp XXXXXXXXXX 127.0.0.1 12345 typ host generation 0 ufrag XXXX network-id 1 network-cost 10
```

## WebRTC connection
Once the ICE connection is made, the client will also attempt to set up DTLS and then SCTP. Once SCTP is set up, the
client will create two (non-negotiated, defined in-band) data channels:
- `ReliableDataChannel`
- `UnreliableDataChannel`

Packets themselves are encoded the same as they would be over RakNet, except:
- Packets are not batched; they are sent individually.
- If a packet exceeds `10,000` bytes, it is split into multiple packets.

To expand on the second point, each SCTP message is structured in the following way:
- Segment count (`uint8`)
- Packet (this is what encryption and compression is applied on!)
  - Packet length (`varuint32`)
  - Encoded packet data (`[]byte`)

Given that packets are usually under this `10,000` byte limit, you can usually expect zero segments. On the off chance
that a packet does exceed this limit, it is split into multiple segments.

This should be treated a sort of promise. If sent a segment count of `3` for example, you should expect the next SCTP
message to have a remaining segment count of `2`, and so on.

It is unclear how this system works with the `UnreliableDataChannel`, given that packet drops could leave the packet in
an unreconstructed state. As such, it is recommended to only use the `ReliableDataChannel` for now.

```go
role := webrtc.ICERoleControlling
if err = ice.Start(nil, peerIceParams, &role); err != nil {
    panic(err)
}

if err = dtls.Start(peerDTLSParams); err != nil {
    panic(err)
}

if err = sctp.Start(peerSCTPParams); err != nil {
    panic(err)
}

reliableDataChannel, err := api.NewDataChannel(sctp, &webrtc.DataChannelParameters{Label: "ReliableDataChannel"})
if err != nil {
    panic(err)
}

unreliableDataChannel, err := api.NewDataChannel(sctp, &webrtc.DataChannelParameters{Label: "UnreliableDataChannel", Ordered: false})
if err != nil {
    panic(err)
}
```

## Credits
Special thanks to the following people for help reverse engineering the protocol:
- [MCMrARM](https://github.com/MCMrARM)
- [Spajk](https://github.com/Spajker7)
- [TwistedAsylumMC](https://github.com/TwistedAsylumMC)
