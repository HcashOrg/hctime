hctime
=======

[![Build Status](https://travis-ci.org/HcashOrg/hctime.png?branch=master)](https://travis-ci.org/HcashOrg/hctime)
[![ISC License](https://img.shields.io/badge/license-ISC-blue.svg)](http://copyfree.org)
[![GoDoc](https://img.shields.io/badge/godoc-reference-blue.svg)](https://godoc.org/github.com/HcashOrg/hctime)

HcashOrg anchored timestamp client and server.

The hctime stack as as follows:

```
+-------------------------+
|         hctime         |
+-------------------------+
            |
~~~~~~~~ Internet ~~~~~~~~~
            |
+-------------------------+
|    hctimed (proxy)     |
+-------------------------+
            |
~~~~~~~~~~ VPN ~~~~~~~~~~~~
            |
+-------------------------+
|   hctimed (backend)    |
+-------------------------+
            |
+-------------------------+
|        hcwallet        |
+-------------------------+
            |
+-------------------------+
|          hcd           |
+-------------------------+
```

## Components
* hctime - Reference client application
* hctimed
	- Proxy Mode: Forwards requests to the backend server.
	- Backend mode: Manages timestamps and creates HcashOrg transaction that anchors transactions in the blockchain.

## Library and interfaces
* api/v1 - JSON REST API for hctime clients.
* cmd/hctime - Client reference implementation
* cmd/hctime_unflush - Debug backend tool to either delete the flush record or reset the chain timestamp.
* cmd/hctime_timestamp - Tool to convert between various timestamp formats.
* merkle -  Merkle algorithm implementation.
* util - common used miscellaneous utility functions.

## Example setup

### Backend

This example hctimed.conf connects to hcwallet running on localhost using the testnet network.

```
wallethost=localhost
walletcert=../.hcwallet/rpc.cert
walletpassphrase=MySikritPa$$w0ard
testnet=1
```

Start the store.
```
store-server$ hctimed
07:36:09 2017-06-09 [INF] HCT: Version : 0.1.0
07:36:09 2017-06-09 [INF] HCT: Mode    : Store
07:36:09 2017-06-09 [INF] HCT: Network : testnet2
07:36:09 2017-06-09 [INF] HCT: Home dir: /home/user/.hctimed
07:36:09 2017-06-09 [INF] HCT: Generating HTTPS keypair...
07:36:09 2017-06-09 [INF] HCT: HTTPS keypair created...
07:36:09 2017-06-09 [INF] FSBE: Wallet: 127.0.0.1:19111
07:36:09 2017-06-09 [INF] HCT: Start of day
07:36:09 2017-06-09 [INF] HCT: Listen: :59152
07:58:03 2017-06-09 [INF] HCT: Timestamp 203.0.113.4:44331 via 10.0.0.2:57126: rejected 20170609.120000 b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1
07:59:36 2017-06-09 [INF] HCT: Timestamp 203.0.113.4:57138 via 10.0.0.2:32322: accepted 20170609.120000 8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6
08:00:10 2017-06-09 [INF] FSBE: flusher: directories 1 in 788.607578ms
08:15:29 2017-06-09 [INF] HCT: Verify 203.0.113.4:39992 via 10.0.0.2:57144: Timestamps 0 Digests 1
08:16:36 2017-06-09 [INF] HCT: Verify 204.0.113.4:57146 via 10.0.0.2:33881: Timestamps 0 Digests 1
08:16:36 2017-06-09 [INF] FSBE: Flushed anchor timestamp: 4172a560a7035c169c4da60cba2cb1fbac686bd01224e09a1a56ce5e6f31cff0 1497013614
```

### Proxy

hctimed also has a proxy mode.  It is activated by specifying the --storehost and --storecert options.
In this example, we assume the proxy has an internal interface with ip 10.0.0.2 that connects to storehost.example.com at 10.0.0.1.

```
proxy-server$ mkdir ~/.hctimed
proxy-server$ scp storehost.example.com:/home/user/.hctimed/https.cert ~/.hctimed/hctimed.cert
proxy-server$ hctimed --testnet --storehost=storehost.example.com --storecert=~/.hctimed/hctimed.cert
07:50:59 2017-06-09 [WRN] HCT: open /home/user/.hctimed/hctimed.conf: no such file or directory
07:50:59 2017-06-09 [INF] HCT: Version : 0.1.0
07:50:59 2017-06-09 [INF] HCT: Mode    : Proxy
07:50:59 2017-06-09 [INF] HCT: Network : testnet2
07:50:59 2017-06-09 [INF] HCT: Home dir: /home/user/.hctimed
07:50:59 2017-06-09 [INF] HCT: Generating HTTPS keypair...
07:50:59 2017-06-09 [INF] HCT: HTTPS keypair created...
07:50:59 2017-06-09 [INF] HCT: Start of day
07:50:59 2017-06-09 [INF] HCT: Listen: :59152
07:58:03 2017-06-09 [INF] HCT: Timestamp 203.0.113.4:44331: b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1
07:59:36 2017-06-09 [INF] HCT: Timestamp 203.0.113.4:57138: 8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6
08:15:29 2017-06-09 [INF] HCT: Verify 203.0.113.4:39992: Timestamps 0 Digests 1
08:16:36 2017-06-09 [INF] HCT: Verify 204.0.113.4:57146: Timestamps 0 Digests 1
```

Now we test the setup using hctime.  Note that for this example one digest was already known to the system and one was not.  You can spot the difference in the hctimed trace byt the words "accepted" and "rejected".  Accepted means the file digest was unknown to the store and could therefore be added.  Rejected on the other hands means that said digest already exists and therefore can not be added again.  A digest can only be queried once it has been added to the store.

Per the trace above we issue a known digest first:
```
$ hctime -v /bin/ls
b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1 Upload /bin/ls
b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1 Exists /bin/ls
Collection timestamp: 1497013200
```

And now we issue an unknown digest:
```
$ hctime -v myfile.txt
8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6 Upload myfile.txt
8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6 OK     myfile.txt
Collection timestamp: 1497009600
```

In this example we wait a bit until the store hits its scheduled hourly flush regimen.  This can be observed in the store trace.

And now let's ask about these digests:
```
$ hctime -v b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1
b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1 Verify
b1d080f4d09ea21a7b1872d87993079a84718f485de87d0327b6d1da922620e1 OK
  Chain Timestamp: 1496430430
  Merkle Root    : 9788d5d7b85f2b68ec21d26e738dce6cdd367ee0ec58b53ad6bd4d46b0bc3018
  TxID           : 554b27c309ac9a8dab8ae261bb13dcfcdd351aa5f196322c112f04d106e000f3
```

The next digest was anchored but the store did not have the chain timestamp cached yet.  This can be observed in the store trace; just look for "Flushed anchor".
```
$ hctime -v 8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6
8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6 Verify
8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6 OK
  Chain Timestamp: 1497013614
  Merkle Root    : 8496855341883fdc90cc532f8304d1c46a60586fb15d99f07e41bb5ab19c79c6
  TxID           : 4172a560a7035c169c4da60cba2cb1fbac686bd01224e09a1a56ce5e6f31cff0
```

You can find the merkle root using block explorer.  Surf to https://testnet.HcashOrg.org/tx/554b27c309ac9a8dab8ae261bb13dcfcdd351aa5f196322c112f04d106e000f3 and in the transaction you'll find an entry that is as follows:
```
OP_RETURN 9788d5d7b85f2b68ec21d26e738dce6cdd367ee0ec58b53ad6bd4d46b0bc3018
```
The astute reader noticed that this is the Merkle Root the hctime client returned.

Note that this example was run on a single machine but that the listen port bits were removed for clarity.

## License

hctime is licensed under the [copyfree](http://copyfree.org) ISC License.
