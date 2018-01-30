# Using the Lightning Network

First let's start with a brief, high-level explanation of what the Lightning Network is and how it works. From the [community documentation](http://dev.lightning.community/overview/#lightning-network):

> The Lightning Network scales blockchains and enables trustless instant payments by keeping most transactions off-chain and leveraging the security of the underlying blockchain as an arbitration layer.

> This is accomplished primarily through “payment-channels”, wherein two parties commit funds and pay each other by updating the balance redeemable by either party in the channel. This process is instant and saves users from having to wait for block confirmations before they can render goods or services.

And the key word to remember is __network__. Payment channels by themselves are not so interesting. But when a transaction can be passed along from one peer to the next until it reaches its recipient - without the need for a direct connection between the payer and payee - now that is interesting.

In this workshop, we are going to setup our own private lightning network between three different peers (users). We will call these three users "Alice", "Bob", and "Charlie". All three users will connect to our private bitcoin network via `btcd`. Here is a diagram to illustrate:
```
   (1)                        (1)                         (1)
+ ----- +                   + --- +                   + ------- +
| Alice | <--- channel ---> | Bob | <--- channel ---> | Charlie |
+ ----- +                   + --- +                   + ------- +
    |                          |                           |
    |                          |                           |
    + - - - -  - - - - - - - - + - - - - - - - - - - - - - +
                               |
                        + ----------- +
                        | BTC network | <--- (2)
                        + ----------- +
```
This setup will allow all of the users to send payments to the each other:
* Alice to Bob
* Bob to Alice
* Alice to Charlie
* and so on..


## Ready-to-go development environment

For this workshop, you will each connect to a virtual private server with a lightning network development environment which has already been prepared.

You will need to get the IP address and password for the virtual private server (VPS) from me via the IRC channel we are using during the workshop:
* In your browser go to [webchat.freenode.net](https://webchat.freenode.net/)
* In the "Channels" field enter `#ln-workshop-pp`

Once you have the IP address and password, run the following command in a terminal window:
```bash
ssh lnd@HOST
```
Replace `HOST` with the IP address of the remote virtual private server (VPS).

Enter the password for the `lnd` user when prompted.

Once connected, you should be able to execute commands as the `lnd` user on the remote server. Try this one:
```bash
whoami
```
This should output your username.


## Starting the bitcoin node daemon

`btcd` is the gateway that `lnd` nodes will use to interact with the Bitcoin network. `lnd` needs `btcd` for creating on-chain addresses or transactions, watching the blockchain for updates, and opening/closing channels.

In the terminal window where you are connected as the `lnd` user, run the following:
```bash
btcd
```

Wait until you see the following:
```
2018-01-30 00:00:00.000 [INF] RPCS: RPC server listening on 127.0.0.1:18556
2018-01-30 00:00:00.000 [INF] CMGR: Server listening on 0.0.0.0:18555
2018-01-30 00:00:00.000 [INF] CMGR: Server listening on [::]:18555
```


## Creating and Funding Wallets

Open a new terminal window and connect as the `lnd` user again.

In this new terminal window, run the following command:
```bash
lnd-alice
```
Wait until the daemon says that it has started and is listening.

Create Alice's wallet:
```bash
lncli-alice create
```
Set the wallet password to whatever you like or hit enter twice to set an empty password.

Create a new bitcoin address:
```bash
lncli-alice newaddress np2wkh
```
Example output:
```json
{
    "address": "<new address printed here>"
}
```

In the terminal window where `btcd` is running, stop it by pressing `Ctrl` plus the `C` key.

Re-start `btcd` but with Alice's address as the recipient "mining" address:
```bash
btcd --miningaddr=<ALICE_ADDRESS>
```

Generate blocks using `btccli`:
```bash
btcctl generate 100
```
Example output:
```json
[
  "68d12bead9ac4a87599582d5186bf08634c9dfe86678d123b27d7d682ecd39df",
  "14cd20d79a0b2786b6a24710e75822eadd949f57cac93ea0dc8abcf858f7bf18",
  "67e5d6ceb6db58d334a195ab85613753aa909a2783c61c600320733614d6c7c7",
  "4e92a4c34792ac9394efb8470d6251656b09ae1e9d8bdfb95f5f7a69569ce925",
  "5ba8cbc7d51c80372aa48512fc81760f48f26167df518e6845ace3fab0978882",
  "... more block hashes omitted"
]
```
Normally blocks are mined about once every 10 minutes. But we don't have that kind of time, so we use the above command to generate blocks on-demand as we need them.

Let's check Alice's wallet balance:
```bash
lncli-alice walletbalance --witness_only
```
Example output:
```json
{
    "total_balance": "5000000000",
    "confirmed_balance": "5000000000",
    "unconfirmed_balance": "0"
}
```
* The amounts shown are in satoshis
* `1 bitcoin = 100 million satoshis`
* `--witness_only` mean that we only want to consider witness outputs when calculating the wallet balance. This is important because we will need "witness outputs" later to be able to open channels.

Repeat the above procedure to give Bob and Charlie some bitcoin as well.


## Opening a Channel

Get Bob's lightning network public key:
```bash
lncli-bob getinfo
```
Example output:
```json
{
    "identity_pubkey": "<Bob's public key printed here>",
    "alias": "030ec2d6d951bc86067a",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 200,
    "block_hash": "52f390510e2c35249d0d4bba817d2d71bc5b75dc212b95b54c5c72d2bbedee0c",
    "synced_to_chain": true,
    "testnet": false,
    "chains": [
        "bitcoin"
    ],
    "uris": [
    ]
}
```

Connect Alice's lightning network node to Bob's:
```bash
lncli-alice connect <BOB_PUBKEY>@localhost:10012
```
Example output:
```json
{
    "peer_id": 0
}
```

Check that Alice's node is connected to Bob:
```bash
lncli-alice listpeers
```
Example output:
```json
{
    "peers": [
        {
            "pub_key": "<Bob's public key printed here>",
            "peer_id": 1,
            "address": "127.0.0.1:10012",
            "bytes_sent": "7",
            "bytes_recv": "7",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "0"
        }
    ]
}
```

And now connect Charlie's node to Bob:
```bash
lncli-charlie connect <BOB_PUBKEY>@localhost:10012
```

But all we've done so far is make each user's lightning node aware of the others. We haven't setup any channels yet.

Let's create a channel between Alice and Bob:
```bash
lncli-alice openchannel --peer_id=1 --local_amt=1000000
```
Example output:
```json
{
	"funding_txid": "41f045e8d3a33cceff237183e37eeabff3aea0cf4d64e28238807afe5df2dfa5"
}
```

If you see the following error message:
```
[lncli] rpc error: code = Unknown desc = not enough witness outputs to create funding transaction, need 0.01 BTC only have 0 BTC  available
```
Then you need to go back to the [Creating and Funding Wallets](#creating-and-funding-wallets) step above and check that you are using the correct address type.

You will now need to mine at least six blocks for the transaction to be considered valid:
```bash
btcctl generate 6
```

Check that the channel was created:
```bash
lncli-alice listchannels
```

Following the steps above to open a second channel from Charlie to Bob.


## Sending a Payment

To request a payment, a user must create a new invoice:
```bash
lncli-charlie addinvoice --amt=10000
```
Example output:
```json
{
	"r_hash": "a9b22eca10de08426f11f3f59b8a733f1af831a699c1b3f6ca632533239dc1dd",
	"pay_req": "lnsb1pd8pxdzpp54xezajssmcyyymc3706ehznn8ud0svdxn8qm8ak2vvjnxguac8wsdqqcqzyse0qkh2fdn4adwlz598s4v9l2ulner3jalncsjf33za0r3hksv2u3m7vw2663ypaqcc4fjsuzeh5n5hfsqyggwk3rzp6neng4hza8stgp4aaszp"
}
```
The `pay_req` field is what we will share with our customer (or payer).

Let's have Alice make a payment to Charlie by using the invoice we just created:
```bash
lncli-alice sendpayment --pay_req <Charlie's Payment Request>
```

Check the balances in Charlie's channel to see that the payment was received:
```bash
lncli-charlie listchannels
```
