# RaiBlocks application : Common Technical Specifications

## About

This specification describes the APDU messages interface to communicate with the RaiBlocks application.

## Wallet usage APDUs

### Get Wallet Public Key

#### Description

This command returns the public key and the encoded address for the given BIP 32 path.

#### Coding

**Command**

| *CLA* | *INS*  | *P1*                             | *P2* | *Lc* | *Le* |
|-------|--------|----------------------------------|------|------|------|
|   E0  |   40   |  00 : do not display the address |  00  |      |      |
|       |        |  01 : display the address        |      |      |      |

**Input data*

| *Description*                                                                     | *Length*     |
|-----------------------------------------------------------------------------------|--------------|
| Number of BIP 32 derivations to perform (max 10)                                  | 1            |
| First derivation index (big endian)                                               | 4            |
| ...                                                                               | 4            |
| Last derivation index (big endian)                                                | 4            |

**Output data**

| *Description*                                                                     | *Length*     |
|-----------------------------------------------------------------------------------|--------------|
| Public Key length                                                                 | 1            |
| Uncompressed Public Key                                                           | var          |
| RaiBlocks address length                                                          | 1            |
| RaiBlocks address                                                                 | var          |
| BIP32 Chain code                                                                  | 32           |

## Test and utility APDUs

### Get Firmware Version

#### Description

This command returns the firmware version of the dongle and additional features supported

#### Coding

**Command**

| *CLA* | *INS*  | *P1* | *P2* | *Lc* | *Le* |
|-------|--------|------|------|------|------|
|   E0  |   C4   |  00  |  00  |  00  |  05  |

**Input data**

None

**Output data**

| *Description*                                                                        | *Length* |
|--------------------------------------------------------------------------------------|----------|
| Features flag                                                                        |    01    |
|   0x02 : implementation running with screen + buttons handled by the Secure Element  |          |
|   0x04 : implementation running with screen + buttons handled externally             |          |
|   0x08 : NFC transport and payment extensions supported                              |          |
|   0x10 : BLE transport and low power extensions supported                            |          |
|   0x20 : implementation running on a Trusted Execution Environment                   |          |
| Architecture                                                                         |    01    |
| Firmware major version                                                               |    01    |
| Firmware minor version                                                               |    01    |
| Firmware patch version                                                               |    01    |

**Availability**

This function is always available.

## Transport protocol

### General transport description

Ledger APDUs requests and responses are encapsulated using a flexible protocol allowing to fragment large payloads over different underlying transport mechanisms. 

The common transport header is defined as follows:

| *Description*                                                                     | *Length* |
|-----------------------------------------------------------------------------------|----------|
| Communication channel ID (big endian)                                             | 2        |
| Command tag                                                                       | 1        |
| Packet sequence index (big endian)                                                | 2        |
| Payload                                                                           | var      |

The Communication channel ID allows commands multiplexing over the same physical link. It is not used for the time being, and should be set to 0101 to avoid compatibility issues with implementations ignoring a leading 00 byte.

The Command tag describes the message content. Use TAG_APDU (0x05) for standard APDU payloads, or TAG_PING (0x02) for a simple link test.

The Packet sequence index describes the current sequence for fragmented payloads. The first fragment index is 0x00.

### APDU Command payload encoding

APDU Command payloads are encoded as follows:

| *Description*                                                                     | *Length* |
|-----------------------------------------------------------------------------------|----------|
| APDU length (big endian)                                                          | 2        |
| APDU CLA                                                                          | 1        |
| APDU INS                                                                          | 1        |
| APDU P1                                                                           | 1        |
| APDU P2                                                                           | 1        |
| APDU length                                                                       | 1        |
| Optional APDU data                                                                | var      |

APDU payload is encoded according to the APDU case 

| Case Number  | *Lc* | *Le* | Case description                                          |
|--------------|------|------|-----------------------------------------------------------|
|   1          |  0   |  0   | No data in either direction - L is set to 00              |
|   2          |  0   |  !0  | Input Data present, no Output Data - L is set to Lc       |
|   3          |  !0  |  0   | Output Data present, no Input Data - L is set to Le       |
|   4          |  !0  |  !0  | Both Input and Output Data are present - L is set to Lc   |

### APDU Response payload encoding

APDU Response payloads are encoded as follows:

| *Description*                                                                     | *Length* |
|-----------------------------------------------------------------------------------|----------|
| APDU response length (big endian)                                                 | 2        |
| APDU response data and Status Word                                                | var      |

### USB mapping

Messages are exchanged with the dongle over HID endpoints over interrupt transfers, with each chunk being 64 bytes long. The HID Report ID is ignored.

## Status Words 

The following standard Status Words are returned for all APDUs - some specific Status Words can be used for specific commands and are mentioned in the command description.

**Status Words**

|   *SW*   | *Description*                                                              |
|----------|----------------------------------------------------------------------------|
|   6700   | Incorrect length                                                           |
|   6982   | Security status not satisfied (dongle is locked or invalid access rights)  |
|   6A80   | Invalid data                                                               |
|   6A82   | File not found                                                             |
|   6B00   | Incorrect parameter P1 or P2                                               |
|   6Fxx   | Technical problem (Internal error, please report)                          |
|   9000   | Normal ending of the command                                               |