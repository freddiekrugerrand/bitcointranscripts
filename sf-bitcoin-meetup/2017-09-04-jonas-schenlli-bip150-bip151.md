---
title: Jonas Schenlli Bip150 Bip151 (2017-09-04)
transcript_by: Bryan Bishop
categories: ['meetup']
tags: ['P2P', 'bitcoin core']
---

Jonas Schnelli

bip150 and bip151: Bitcoin p2p network encryption and authentication

<https://github.com/bitcoin/bips/blob/master/bip-0150.mediawiki>

<https://github.com/bitcoin/bips/blob/master/bip-0151.mediawiki>

<http://diyhpl.us/wiki/transcripts/scalingbitcoin/milan/bip151-peer-encryption/>

# Introduction

Alright guys.. take a seat. ((various mumblings)) Want to thank our sponsors, Digital Garage (applause). Second order of business, if you guys could... trash away.. that would be awesome. There are trash cans in the back and right there and right next to the drinks. So at the end of the night, that would be great. Jonas Schnelli is a bitcoin core developer and he is here to talk about bitcoin encryption and authentication on the p2p network.


Thank you for allowing me to speak in the city. I already got two tickets with my standard sized van in the city, probably a bad choice for vehicle. I am going to talk about bitcoin p2p encryption and authentication.

First I am going to tell people who I am and what I am not. I am not a cryptographer. Nor am I an economist. I am also not Mr. Know Everything. Who am I? I am the guy.. who wants to connect the right people to get this done. The whole thing I am going to present now is not just my own idea, but ideas that are walking around since years.

Let me try to dive in.

We all know that the bitcoin network is a p2p network. The whole practice is unencrypted which maybe makes sense because what do we transaction, it's transactions of blocks and it's going to end up in the public blockahin anyway so why bother with encryption? We can have a look at bitcoin SPV which is like the worst case-- there are many smart phones with SPV wallets. They communicate with bloom filters about their wallet characteristics, and at least since the paper about deanonymized SPV wallet users, we know that this part of information is not supposed to be public, it's private information. What we actually do is we allow every ISPs and network providers to analyze your own financial history and payments, which is probably not what we want.

There's a new proposal from... SPV client side filtering. It's not something particularly new. But even there, maybe not really public, what we were to fetch. So assume you had filters, you find out which blocks were relevant to you, and maybe just download one or two blocks... well then it's obvious which addresses you wanted. So even there, you need a vested peer connection and not reveal this information.

If you do the full block SPV where you download the full chain in general, then there's some information that you are sharing there, which is the starting height, shouldn't hurt too much, but it's also kind of dangerous.

Right now we know that there are ISPs with peers, and they can make a graph of the traffic without the peers knowing about this. They can tamper the network without you being able to detect it. If the ISP is malicious, you don't know about that.

There are other cases where there are malicious peers. An example of this is one where one of the Chainalysis started to bootstrap a bunch of peers to analyze where transactions come from. There is some we can do with encryption, but there's other proposals that are working in that direction, but encryption certainly helps.

There is also a BGP routing partitioning attacks, we know it's simple to delay blocks and you can delay certain miners so that they get blocks later. They can even partition you by intentionally giving you or kind of sending wrong information from certain peers so that you can disconnect peers. This is relatively trivial at the moment because there's no peer authentication.

If you look at the SPV problem again, which is simple to understand, you're going to share information and there acn be man-in-the-middle attacks to take that information and analyze your internet behavior. If you're going to download or purchase certain things on the internet, they might collect your personal information such as transaction history and other data.

The same paper showed that 13 ISPs host 30% of the peers. That's kind of frightening. 60% of all bitcoin connections pass over 3 ISPs. So if these ISPs work together, they can intercept a large amount of bitcoin traffic and manipulate it.

So if we are talking about passive interception where you don't know if they are intercepting, and undetectable message manipulation. This is the curret state. It allows people to deanonymize users, cause delays, force disconnects, use BGP partitioning attacks.

So yeah we know what SPV means for the user... you're going to lose a lot of privacy and information.

# bip151

bip151 was proposed to create secure channels between peers. It doesn't allow you to have full man in the middle protection. If a peer is malicious, then you don't know. But at least network authority is no longer albe to manipulate the content of messages.

How does it work? This is a bit more technical, I hope you don't mind. This is still high level. It's standard form-- create a secure channel, two session keys created on each side, so 4, one for each direction, so there's an ephemeral session key created and shared over a EC diffie-hellman key exchange, in order to calculate a shared secret on both sides.

And you know probably that there's still a possibility of an active MiTM where a malicious peer or ISP spawns up a node; and it proxies or forwards the connection traffic to the peer that you had the idea to connect to. So they can substitute keys in both directions and be the middle man and collect everything and tamper everything.

A very important differetiation is that this is detectable in this scenario. Imagine peer A could call peer B which it planned to connect to, and ask what's the session id? If the session id is not identiticle, then you know there's a peer in the middle. If something is detectable in surveillance territory, then it's quite not ideal to intercept those connections.

# TOFU?

A good example is how ssh does this. When you connect to a host the first time, it remembers and shows you the fingerprint and it asks you to verify the fingerprint-- but nobody does that. Well, do any of you? Oh, that's good. That's my host. And then once you connect to the host, it stores the fingerprint in the `known_hosts` file. And then if the fingerprint changes, you get a big warning, and then you delete the line and everything is good again... this is called TOFU, trust on first use. So if your first connection is MITM then you're going to trust that. You need to login over an existing secure channel and verify the fingerprint. bip151 does not have a TOFU concept, but maybe one could be added. I think it's good that it does not have one, though.

The details about key exchange is that it's diffie-hellman key exchange is based on the secp256k1 curve. There's no new crypto library involved. We can use what we have already to do the ECDH. There's a key exchange in both directions. You get the pubkey of the remote host and you do the same in the opposite direction, so you have two shared secrets one for each direction. Once the handshake has been done, no more plaintext packets are allowed. So you do this first, and then from that point on it's encrypted and has no possibility of further tampering.

bip151 has flexible network scheme numbers so it's possible to add other symmetric cipher. Right now there's one proposed which is the chacha20 poly1305. It's also stream cipher so that you can do AESGCM... which allows fast stream encryption. It's basically 256-bit keys, so poly1305 informs the AEED... mac attack. Really makes it robust.

The proposal in bip151 is to use the openssh variant of chacha20 poly1305. It's similar to Adam Langley RFC539. It also encrypts the length so that the packet size... the packet size in bitcoin shows pretty quickly what it is. We can encrypt the length and it's slightly different way to how to calculate the MAC than the ssh variant.

It's not thousands of lines of code, it's only 400 lines of code including tests. It's not so complicated. This is not new crypto, it's known crypto that you can find in openssh. And an implementation has already been written in openssh of course.

Once we have a shared secret, we can use HKDF to do key derivation from the secrets. We can generate two keys, one from the symmetric encryption and the other from the other tag... your peer can say, you want to re-key every 10 minutes or every N bytes. There's a re-keying that has to be done on an interval defined by the peer. And also not more than once every 10 seconds. Must re-key after every 1 GB of data.

And the big thing that I really like is that we can have a new message structure. If you're familiar with the bitcoin p2p protocol, then you know about the 4 byte magic that indicates what netwrok you're on. And there's 12 byte fixed length in block headers. And there's the length of the payload and then the checksum which is very important. And then the payload itself. Once we establish encryption, we can use whatever we want. So the idea is to have the encrypted length, encrypt the payload, the MAC tag, and then in the encrypted payload we can have multiple packages, whih have a slightly different form... This is an example of a new INV message. Before, we had the magic and 12-byte comment, and also we had ... as some clients filled up the some random data from these private keys. And then there is a length, a checksum so that it comes out to roughly 61 bytes. With encryption we can drop the magic and the checksum. And then with the MAC tag we come down to roughly 65 bytes. It's not much larger. So we might say what about ... as long as.. slows you down? But no, I don't think so. I took the crypto++ benchmark. Here we can see, maybe before that, ... does the two round sha256 on old packet in order to get you a 4-byte checksum. It's quite expensive. The crypto++ benchmark says 13.4 cycles/byte.... I think these numbers are quite optimistic, but it shows us that it wont be much slower, it may even be faster to do it in the encrypted fashion.

# Downsides of bip151

Well, we have to know that it's still a proposal. secp256k1 Diffie-Hellman (ECDH) is not quantum crypto safe. Lamport and merkle signatures could help. We know that secp256k1 is not quantum-safe. There are proposals to do so.

# Criticism

Not sure if it's just one or some, some people said false sense of security which yeah I kind of agree if you say it's encrypted is it really encrypted. But let me go back to this to show you an example. The sense of security, if you say well it's end-to-end encrypted... we use signal because it's end-to-end encrypted. But what does this mean? What is an end? What if an end sn't safe? There's lcear evidence that android on my phone isn't safe. So it should always be known that end-to-end encryption never means that the end is safe. It's only for in-flight.

False sense of security? Sure, secure channel does not mean that you have solved man-in-the-middle attacks.

Some people said that there is no private information in traffic between peers. I disagree. I think there are pieces of information that deserve encryption.

Some people say that the needs of p2p protocol should not drive the needs of a client-server system. Say you want to connect your phone to your node at home. Well, without encryption it's sort of impossible to do that safely. But some people say that we should not extend the p2p protocol to do that. But I disagree there too. If it's possible to do bitcoin p2p protocol in a secure way, why not do it in same way to allow client-server connection. I think it is more practical to just use the same bitcoin p2p protocol. It's not scalable to setup other protocols.

# Advantages

* Eliminate passive inception.
* Eliminates undetectable packet manipulation.
* Reduce attack surface for BGP-style network partitioning.
* Build a base for secure node-to-node communication. There's ... add-on thing.. designated peer authentications, there's FIBRE, is it called FIBRE or is it called FIBRE? Last time I checked, it was just a symmetric single passphrase. So I'm not really sure it's undetectable packet manipulation in that scenario.

So yeah all these sorts of channel connections... yeah, we need this. There's no guarantee that people are connecting to the nodes peers.

# Other options

* Openssl: So people asked, why not use openssl? It's there and it works. It's kind of a joke really. This is the vulnerability list for openssl. Really small slider on the right of this browser screenshot. Also what's in there is way too much for bitcoin. And we had good experience with using well-known crypto in its own dependency chain or in its own source code base.
* Tor: Tor was not designed for this.
* Stunnel: Cumbersome setup... maybe you can set this up on your smartphone if you're an expert.

# bip150: Authentication

This is the proposal to build in authentication with peers. This avoids the active man-in-the-middle attack. It allows for whitelisting peers. It would allow private p2p extensions, like my peer wants to serve fee estimations.. that's a client-server thing, maybe controversial but we don't know yet.

And hte properties of bip150 is that it expects pre-shared identity keys. If you connect to your node at home, you need to have its keys already. If you connect to your friends or a well-known peer, you need to get the identity key on a different channel.

bip150 uses secp256k1. No new dependencies. And the special thing is the fingerprint-free authentication which gmaxwell came up with. I think we can see the advantage of using this technique over existing solutions.

This form of authentication does not reveal yours until the other side has proven that they know you already. It's just fingerprint-free authentication.

If you connect the peer, you will first only show them a hash of who you think they are, and then you can verify and let you in.

Here we have the whole chain, which is probably not visible in the audience. The connecting peer sends you a hash of the encryption session, which is the thing I showed you before where you can have a landline call and verify the session id and then you add the remote expected identity key in a hash. So the remote peer can verify if the connected peer really knows me already, and if so they give you back a signature, and the signature is fingerprintable, but it only happens if the connecting peer has already proven they know you. And same on the other side--- and at the end, you have a state where you know the peer that you connected to, or you have an assurance that the peer you are connecting to is the one with the pre-shared key, and the same on the other side.

# These are optional

These are optional proposals. They are not consensus-critical. You don't need to use it. Authentication is already happening maybe in whitebind, tor, FIBRE. There's already kind of fixed channels being built on top of the peer network in those contexts.

# Next steps

Implementation, hopefully. There's some work on this in the network library refactoring. Once that is settled, we can move on to implementing these things. Some guys in the front row have already signaled that maybe that network refactoring work is done? Hopefully.

And there are other things that may be critical for overall security in terms of partitioning attack protection. DNSSEC? First time you start up a bitcoin core node, it will get IPs over a DNS channel and this is also easy to tamper with, so somebody could partition you at the edge of the network, and perhaps this deserves some form of authentication. This is another piece of the puzzle.

# Q&A

Yeah that was pretty quick. I hope you have some questions.

Q: How many peers are SPV nodes often connected to?

A: There are BGP-attack possibilities. Chainalysis. Lots of threats. SPV only connects to 4-8 peers. Not thousands.

Q: Are there censorship resistance reasons... are any of the other encrypted protocols using 65 bytes?

A: Those 65 bytes... I think that's just an example for the packet. Yes.... I hope it's calculated correctly. But I mean, other packages, the int is short, it's just.. you get like four instead of twelve bytes.

Q: Is there a code impossible to encrypt?

A: I don't think so. I am not a cryptographer.

Q: I'm a little bit nervous about the... the kind of layer violation going on between the negotiation of the header and then having to come back and keep some state at the network. Have you looked at kind of using the current header structure so that... basically, imagine that the networking part, kind of the, by taking the header, and separating the ... and forwarding it along. I mean, .. but for the sake of example, as  you ... you have to pass it up in order to decode the message; you have to understand that the setup is being done so that you can pass it to that layer. Is it possible to separate those two layers? Or how much you have considered this?

A: You're thinking already lines of code, which is good. I think we will have time in the next few days to talk about that. We can improve the proposal, for sure.

Q: There's still... into... being able to talk with the peer if the ISP... if the handshake process..

A: Some of these protocols use certificate authority chains. SSH does it using pending down the .. identity key. Both are not fully preventing you from active man-in-the-middle so... it's always the problem. Pieter may have a more precise answer.

sipa: I think you,... jonasschenlli was talking about wto ddifferent proposals. One was to setup an encrypted channel. And secondly, authentication to know if you are talking with the right peer. If you are not using the authentication proposal, then of course, just encrypting your conection doesn't lead to anything if you don't know who you are talking with. Your ISP could be running a node. If you are purely talking about encryption, it makes certain attacks harder but it does not prevent anything. But as soon as you use the authentication proposal, these problems go away. If you have verified the key of the node, and you have configured the nodes to connect, then yes that problem goes away.

gmaxwell: I would like to clarify. The peer encryption does have the advantage of blocking passive observers-- and passive observers are cheap to do... and so is this prevention. If your ISP was monitoring your bitcoin connections today, you wouldn't be able to tell it. Comparing session ids is also something that is possible. This is a useful piece of machinery to have in the system, even without the peer auth part.

Q: Are there any thoughts to extending this to Web of Trust so that I don't have to authenticate my friends directly?

A: That's a possible extension, but first we need to do the other steps. But also, you need to authenticate your peers or else WoT doesn't work.

gmaxwell: Part of the reason for the split between authentication and encryption is so that we can have many different authenticaiton schemes that people could use. I don't know if WoT is ever going to be useful for bitcoin network. But maybe for trasaction relay it will be interesting. But any cool authentication ideas, that can be dropped into the scheme and use the same encryption layer.

callen: So it seems there are three states... an ongoing relationship with tohter peers, you have got an identity check, and then there's initial setup, a node that has never talked with the network or hasn't talked in a long time. Are there--- one of my concerns would be early on, establishing the network, has there been some discussion about improving the performance there? Besides DNSSEC. Which I've seen being promised for pretty much my whole 30+ year career.

A: I think DNSSEC has that kind of layer of anonymity which is for say good. But sometimes it could be more desirable to first use a static list. But I think there's, it's a piece we need. That's just the nature of the p2p system. First kickstart... hopefully all over that.

gmaxwell: The network is still an anonymous identity list. You don't need to bootstrap identities here because there is no need for identities in the first place, except in areas where people are already connecting to identified peers.

Q: Does the onde signal this with a service bit, or is this o na fork, and if you add more ciphers how does this work?

A: How do you announce the encryption type? How do you know if the connection is possible? Yeah I don't know if the current proposal is good for that. The proposal says you have to do this between the version check handshake. So there's no service bits. So you should just try the encryption handshake and if the peer ignores it, maybe disconnect and try again. It's not ideal. Ideally you encrypt at the very first point.

Q: So it makes sense for nodes to listen on multiple ports?

A: That would make it more obvious what version you have. But yeah, maybe.

Q: Would you say that encrypting the bitcoin network traffic would bring further state attention to the bitcoin network?

A: I don't think so. Kind of having undetectable transaction like zcash and stuff, that's kind of ringing the alarm clocks there. I think pure encryption? Nah, I don't think so. They probably expect at much at this point.

Q: How did you decide how often or when to check a session key?

A: The current proposal checks at the beginning and from that point there's a re-key. There's also a re-key that has to be done once the session id has been checked. And then from then there's kind of a chain of the keys that goes back to the initial detection.

gmaxwell: If you're doing it manually, how paranoid are you?

A: The host could have changed it.

gmaxwell: You could manually check the keys. To make sure they have the same session id? Do it every once in a while. If there's only one connection up, there's no way that a man in the middle came into the existing connection. So maybe look at it once a week or once a month. If only a tiny number of users check those session IDs, then no tier 1 ISP could be doing global observation of the network.

Q: With regards to checking session IDs are the same or not, if an attacker knows that you're going to be checking the first digit of the session ID, are they able to do anything?

gmaxwell: If the attacker knew that you were only going to hceck the first digit, then yeah they can grind that. But if they don't know what digit you're going to check..

A: The pure fact that it is detectable will reduce the attack surface enormously. So this makes it worth doing. More questions?


