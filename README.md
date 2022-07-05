# A Tour of Core Lightning

Core Lightning (CLN) is a lightweight, highly customizable and standard compliant implementation of the Bitcoin Lightning Network protocol. We'll be running everything in this repl, all the dependencies are set up through Replit's nix backend, but if you're interested in how to install and run Core Lightning on your machine see the walkthrough under "Installation" in [the coreln github repo](https://github.com/ElementsProject/lightning). If you want to run it with Nix like we do here, see [nix-bitcoin](https://github.com/fort-nix/nix-bitcoin), a collection of Nix packages and NixOS modules for easily installing full-featured Bitcoin (and Lightning) nodes with an emphasis on security.

We're going to show you how to:

    1. Set up a CLN regtesting environment to play around in. We'll make 3 lightning nodes with a shared bitcoind backend (more on this later).
    2. Explore the plugin architecture and plug a couple more processes in for additional functionality.
    3. Connect your CLN nodes, fund them in a regtesting environment, and open one way channels.
    4. Perform a collaborative channel open with liquidity ads to dual-fund a channel.
    5. Create and pay bolt11 invoices and bolt12 offers.

Let's get started!

## Setting Up Your CoreLN Regtesting Environment

We've set up the dependencies already for you to run bitcoin and lightning on regtest in this repl. Run the following command to start your bitcoin daemon on regtest for a second (this will build a .bitcoin folder, we need this for the regtest script we're about to run), then shut it down by entering Ctrl-C. 
```
bitcoind -regtest

## You should see something like this:
## 2022-07-03T21:19:15Z Bitcoin Core version v22.0.0 (release build)
## 2022-07-03T21:19:15Z Validating signatures for all blocks.
## 2022-07-03T21:19:15Z Setting nMinimumChainWork=0000...
## ...
```

Niftynei wrote a startup_regtest script (it's also available in the lightning/contrib folder if you're following along on your local machine) that we've copied here that will start up a bitcoin daemon process on regtest with 3 CLN nodes sharing it as a backend. 

Side Note: Lightning nodes don't always need to run their own bitcoin backends, they just need a way to get on-chain data like what the UTXO set is currently and a way to broadcast transactions. There's actually a plugin for CoreLN called "sauron" which lets you start a node that uses blockstream.info's Esplora API to get and send on-chain data. It makes running a lightning node easier but makes you entirely reliant on trusting blockstream!

You can start up the bitcoin and lightning regtesting environment by running the following command in your repl's shell. The script sets a bunch of aliases for your bitcoin and lightning nodes to make typing commands easier:

```
source ./startup_regtest.sh
```

Then spin up your bitcoin backend and 3 lightning nodes by running:
```
start_ln 3
```

If you see the following, you've just spun up your core lightning nodes and your regtest environment is ready to play!
```
Bitcoin Core starting
awaiting bitcoind...
[1] 192 ## this is the process id of cln node1, aliased to l1
[2] 221 ## this is the process id of cln node2, aliased to l2
[3] 255 ## this is the process id of cln node3, aliased to l3
Commands: 
    l1-cli, l1-log,
    l2-cli, l2-log,
    l3-cli, l3-log,
    bt-cli, stop_ln
```

### Exercise A 
Copy down the connection info for your 3 coreln nodes. You can do find them by running `getinfo` for each node like `l1-cli getinfo`, and combining the id, address, and port info as such

```
## id@address:port
## e.g. 038ae5cab4c52ba987dba51536632ae12d90c10138e8e0a0163eae042d03458593@127.0.0.1:7171

## l1 Connection info:

## l2 Connection info:

## l3 Connection info:
```

## So What Exactly IS a Core Lightning Node?

Let's briefly talk about what we mean when we say we 'started our core lightning nodes'. Your Core Lightning "Node" is a node in the lightning network, it is the set of processes which "speak" the lightning protocol. We can add additional processes and functionality to that lightning "node", or remove them sometimes, and this is what brings us to the core lightning architecture of *plugins*.

## Core Lightning Plugins

When you started your core lightning nodes, your terminal printed out the process ids of each of your 3 nodes (they'll be different for you, for me they were 192, 221, 255, see above). Let's look at the process tree, the parent process and child subprocesses, for our first lightning node, by running the following command:
```
pstree 221 ## replace this with your l1's processid
```

Which should output something like...
```
-+= 00221 runner /nix/store/8kgsjv57icc18qhpmj588g9x1w34hi4j-bash-interactive-5.1-p12/bin/bash -rcfile /nix/store/dzpjafx42kx0rmkqyba0m46vpkbjk62r-replit-bashrc/bashrc 
 \-+- 00223 runner lightningd --lightning-dir=/tmp/l2-regtest 
   |--- 00287 runner /nix/store/...clightning-0.10.2/libexec/c-lightning/lightning_gossipd 
   |--- 00284 runner /nix/store/...clightning-0.10.2/libexec/c-lightning/lightning_connectd 
   |--- 00265 runner /nix/store/...clightning-0.10.2/libexec/c-lightning/lightning_hsmd 
   |--- 00247 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/spenderp 
   |--- 00245 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/txprepare 
   |--- 00242 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/pay 
   |--- 00239 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/offers 
   |--- 00238 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/keysend 
   |--- 00235 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/topology 
   |--- 00233 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/funder 
   |--- 00231 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/bcli 
   \--- 00230 runner /nix/store/...clightning-0.10.2/bin/../libexec/c-lightning/plugins/autoclean
```
(nix dependencies make these kinda hard to read, I've replaced the hash check, pfxk1qxj7p82ziljikwgsxm7h68c0yq2, with '...')

So what's going on here? 

We've got a parent lightningd, aka lightning daemon, process, and almost a dozen child processes, which "plug in" to the parent process and together make up the lightning "node". This is a unique architecture to core lightning, and allows for extreme flexibility and modularity by plugging in and unplugging different processes depending on what we want or need.

Core Lightning ships with a bunch of default plugins, those listed above when you ran pstree, but if we want additional functionality we can plug that functionality into the running lightnigd process.

## Plugging in Our Own Plugins
We've cloned down a bunch of coreln plugins from github to the "plugins" folder in your file tree, feel free to take a look at some of the different plugins. CoreLN itself is written in C, but the modular plugins architecture allows you to write your own plugins in whatever languages you want like Rust, Python, or Go.

We're going to use some of the python ones like "helpme" and "summary" to help us set up and manage our regtest lightning nodes. 

First, we'll install some dependencies for doing lightning network things in python,
```
pip install pyln-client pyln-proto pyln-bolt1 pyln-bolt4 pyln-bolt7
```

Then start the "helpme" plugin for each of our nodes, a fun plugin that helps you set up and use your core lightning node! Notice that we don't have to spin down the nodes, we're plugging in these additional processes with 0 down time.

```
l1-cli plugin start $PWD/plugins/helpme/helpme.py &&
l2-cli plugin start $PWD/plugins/helpme/helpme.py &&
l3-cli plugin start $PWD/plugins/helpme/helpme.py
```

When you start a new plugin, coreln will show you all of the currently running processes including the newly started one. You should see an output that ends with:
```
      ...
         "name": "/nix/store/pfxk1qxj7p82ziljikwgsxm7h68c0yq2-clightning-0.10.2/bin/../libexec/c-lightning/plugins/spenderp",
         "active": true
      },
      {
         "name": "/home/runner/Base58-A-Tour-Of-Core-Lightning/plugins/helpme/helpme.py",
         "active": true
      }
   ]
```
Notice that all of the default plugins are listed, along with our new "helpme" plugin which is now running.

## 'Helpme' Run my CoreLN Node
We're going to run through the helpme instructions for 1 node, l1, but feel free to run through these same steps with your other nodes for additional practice.

Now that we've got helpme plugged in, let's see if it can help us set up and run our lightning node. Run,
```
l1-cli helpme
```

Which should output:
```
Welcome to Core-Lightning!

The lightning network consists of bitcoin channels between computers
(like this one), and the ability to send those bitcoins between them.

It's still beta sofware, so DON'T PUT TOO MUCH MONEY in your lightning
node!  Be prepared to lose your funds (but please report a bug if you do!)

*** You are on TESTNET, not real bitcoin!  See 'helpme mainnet'
STAGE 1 (funds): INCOMPLETE: No bitcoins yet.  Try 'helpme funds'
STAGE 2 (peers): Not connected to the network.  Try 'helpme peers'
STAGE 3 (channels): INCOMPLETE: No channels open.  Try 'helpme channels'
STAGE 4 (making payments): INCOMPLETE: No payments made.  Try 'helpme pay'
STAGE 5 (receiving payments): INCOMPLETE: No payments made.  Try 'helpme invoice'
STAGE 6 (adding bling): You have not customized alias or color.  Try 'helpme bling'
```

So let's walk through it!

### Stage 1: Funding our Lightning Nodes
The first thing we'll need is some bitcoin. Now this is regtest, so we'll have to do things a little differently than on chain. We're in complete control of the regtest environment, so we can mine blocks and generate coins as we like. For simplicity, let's mine coins to the layer 1 bitcoin node, bt-cli, and send from there to our lightning nodes as necessary.

In regtest we use the command `generatetoaddress` to generate a block where the block reward goes to an address we control. Run the following to mine 101 blocks where the coinbase reward goes to the bt-cli's wallet. (we do 101 blocks because coinbase outputs are only spendable after 100 confirmations).
```
bt-cli generatetoaddress 101 $(bt-cli getnewaddress)
```

You should see an array of hashes if this worked, those are the block ids for the blocks you just mined. Check that you've got a 50BTC coin in the wallet by running,
```
bt-cli listunspent ## you'll actually have 2 coins here, the regtest startup script starts at block 1 instead of block 0, but usually when you're running regtests yourself you'll follow this process as we did it here mining 101 blocks
```

So now we've got a bunch of bitcoin on our bt-cli layer 1 node. We have to send that bitcoin to addresses controlled by our lightning nodes. First we generate an address for each of our nodes:

```
l1-cli newaddr && l2-cli newaddr && l3-cli newaddr
```

Then we send some bitcoin to each one of these nodes, let's do 10 BTC to each one. We can do this in a single tx using the bitcoin-cli sendmany command,

```
bt-cli sendmany '' '{"replace_me_with_l1_address": 10, "replace_me_with_l2_address": 10, "replace_me_with_l3_address": 10}'

## for example this could look like
## bt-cli sendmany '' '{"bcrt1qeg4k75dj5gq5hrtg39j4v09ycp6hs0rcc0l3ws": 10, "bcrt1q0zej0qnvqyc55re4hu4rdrhc5q249cxqd3glaf": 10, "bcrt1qekt723mjayza474qlcrgcc28q9uw683ut0v8f7": 10}'
```
(make sure you have the empty string, '', between sendmany and your json of address:amount)

If the transaction went through, bitcoind returns the txid of the transaction confirming it was sent to the mempool. Reminder, we're on regtest where WE control the block production, so we need to mine a block to confirm the TX.

```
bt-cli generatetoaddress 1 $(bt-cli getnewaddress)
```

Now if we run l1-cli helpme again, we'll see...

```
STAGE 1 (funds): COMPLETE (1000000000000msat a.k.a. 10.00000000btc)
```

### Stage 2: Connecting to Peers

Our lightning nodes have been running for a little while, let's see if our nodes can see each other...
```
$ l1-cli getinfo
{
   "id": "03e46bab701efb2503f1c48d56654674e0e3836f6dd92a851887b3de8c3800c7fb",
   "alias": "WEIRDROUTE",
   "color": "03e46b",
   "num_peers": 0,
   ...
}
```

Alright, what's going on? Why do we not have any peers, even though we have 3 nodes running on our regtest network?

Our nodes start as islands, they're not connected and don't automatically connect to each other by default. It's a little different than layer 1 Bitcoin where when I first start my bitcoin node it reaches out to DNS seeds to find some peers and begin connecting. The Lightning Nodes actually don't have any idea how to connect to each other right now, we have to do it manually. Let's connect l1 to l2 using the connection info you wrote down earlier.
```
## replace connection info with your l2 node's connection infow you should have it above or just run l2-cli getinfo
l1-cli connect node_id@ip_address:port

## e.g. $ l1-cli connect 022979576a85cbc73cc3d8a8928a3a9c7c97995983822ca8854ab5c69f327f3073@127.0.0.1:7272

```

You should see a return json like this:
```
{
   "id": "022979576a85cbc73cc3d8a8928a3a9c7c97995983822ca8854ab5c69f327f3073", ## l2's node_id
   "features": "08026aa2",
   "direction": "out",
   "address": {
      "type": "ipv4",
      "address": "127.0.0.1", ## l2's address
      "port": 7272 ## l2's port
   }
}
```

Now if we run l1-cli getinfo, we should see that we've got a peer!

##### Exercise B
    1. Connect l2 to l3 using the same process as above. 
    2. Try running l2-cli listpeers, then repeat the listpeers command for l1 and l3. You should see l2 has 2 peers after connecting the 2 of them, but l1 and l3 only have 1. 
    

### Stage 3: Opening Channels

Alright, now that our nodes have peer connections (l1 with l2, l2 with l3) we're ready to open our first channel! 

Through the peer connection, our nodes can pass INFORMATION back and forth. Notice that if you run getinfo on l1 and l3, both only have 1 peer, l2. They don't know about each other yet, only about their connection to l2, they haven't gossipped their peers' info to each other yet. We'll get back to that when we talk about the difference between PEERS and NODES.

Through a channel, our nodes can pass VALUE. A channel opens as an on-chain, layer 1 bitcoin transaction. We can open 1 way channels pretty easily: let's open a 1-way channel from l1 to l2. On-chain, this will look like we're locking bitcoin to a custom script. We won't reveal that the lock was a 2-2 multisig creating a channel between lightning nodes unless we close the channel.

Each node should now have a 10BTC UTXO. We can confirm this by running listfunds.
```
l1-cli listfunds
```

We're now going to open a 10million sat channel between l1 and l2, where l1 is the only party funding the channel so starts with owning all the bitcoin in it. To open the channel we need to specify:
    1. Which node_id we're opening to.
    2. How much of our funds we'd like to commit to the channel.
    3. and what feerate we'd like to pay to the miners

Remembering the order you have to input command line arguments can get annoying, so coreLN lets you add the -k flag to label your arguments as key value pairs,

```
l1-cli fundchannel -k id=02348345cbe1b14d5660c47a3a17005f882fb96be38497fc4c5f2259af6e4e1b78 amount=10000000sat feerate=250
```

If you did it right, you should see a return with the completed tx information.

##### An Aside on The Funding Transaction

Let's take a sec to parse what the funding transaction looks like. Yours will be different, but should look similar to this:
```
{
   "tx": "020000000001011ab18c3018b34fa83fea330d1fea046aaba05e48907ff0f5494606cc03e632370300000000fdffffff0280969800000000002200209793b535a2ed541eda684496ef0ed0d1059aaaf0fd5dd4bbf2f59f52721523e5e632023b00000000160014f89ad37ae2fada6473228e6039f6b0e5f330affe02473044022072cfac46806ba0d2b0eb0967e178d670bd185557cd3f92ef8e3f6c445b79c91002205f3d36186fc04394708c1fe8a9f1a97d064a2d67e3f57a1eea068eac112bacf0012102984ff808535936ff8f2de909e6d6f5ad6e79eccc98e970c446ac929044e8cf7a67000000",
   "txid": "0a5d6cb1f597dc846051765e085770854cae911edec63094bce673aa4b1f0d28",
   "channel_id": "280d1f4baa73e6bc9430c6de1e91ae4c857057085e76516084dc97f5b16c5d0a",
   "outnum": 0
}
```

You can parse this into something more human readable with the bitcoin-cli by running,
```
bt-cli decoderawtransaction replace_me_with_the_raw_tx

## replace that with the string starting with 020000000001...
```

As we can see, the funding transaction takes a single input (the UTXO we sent from the bt-cli earlier), and has 2 outputs to 2 different address types. The second output is l1's change back to itself, a new Pay2WitnessPubKeyHash address. The first output, for 10million sats, locks to a Pay2WitnessScriptHash address. We know it's a lightning channel, but as of right now no one else does for sure. We're eventually going to gossip that it's a lightning channel to everyone else, but if someone were only looking at layer 1 bitcoin they wouldn't know that we're using that P2WSH for lightning stuff.

### Exercise C

    1. Open another one-way channel from l2 to l3. If you did it correctly the command l2-cli , while l3-cli should only list 2 channels.

### Channels: Why does a single channel look like 2?

A question you're probably asking right now, why are there 





### Stage 3a: Collaborative Channel Opens aka Liquidity Ads

### Stage 4: Making Payments

### Stage 5: Receiving Payments

### Stage 6: Adding Bling???

