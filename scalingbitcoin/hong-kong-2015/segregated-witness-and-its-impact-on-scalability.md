---
title: Segregated Witness And Its Impact On Scalability
transcript_by: Bryan Bishop
tags:
  - segwit
speakers:
  - Pieter Wuille
media: https://www.youtube.com/watch?v=fst1IK_mrng&t=36m
---
slides: <https://prezi.com/lyghixkrguao/segregated-witness-and-deploying-it-for-bitcoin/>

code: <https://github.com/sipa/bitcoin/commits/segwit>

SPV = simplified payment verification

CSV = checksequenceverify

CLTV = checklocktimeverify

Segregated witness (segwit) and deploying it for Bitcoin

Okay. So I am Pieter Wuille. I'll be talking about segregated witness for Bitcoin. Before I can explain this, I want to give some context. We all know how bitcoin transactions work. Every bitcoin transaction gets inputs, which refer to previous outputs being spent. Every input has the txid and the signature to prove that it is allowed, plus an amount and script in every output. What this presentation will mostly be about is the question of whether all of this data is equally important.

In particular, we are going to be talking about signatures. It's important to realize here that signatures are really only needed for fully-validating nodes. As a light-weight client, you are not validating signatures, even though they are part of the transactions you still have to download them. If you are using a full-node that is syncing historical data, you don't actually validate all of the signatures in there. Currently there is a mechanism in there using checkpoints, which we want to deprecate soon, but the result will still be that we're not validating all signatures from years ago in deep history.

These signatures are only needed at time of validation. They don't go into the UTXO set, the database of all unspent coins. These unspent transaction outputs don't enter into the UTXO set. This is a significant cost on the resources of both keeping a node running but also the speed of propagation and access to the UTXO set needs to be fast. Of all the data in a transaction, signatures don't go into the UTXO set, even though they account for 60% of the blockchain data. Segregated witness is about ignoring this whenever possible.

Where does the name witness come from? For now, it's motsly a word to refer to the scriptSig in inputs or signatures inside transactions. Later I will extend this meaning. The reason for this name is because signatures are not part of the transaction. They don't describe what the transaction is doing. The only thing they are doing is proving that the transaction is authorized by the previous owners of the coins. There are usually multiple possible valid signature for the same transaction. We don't really care what the signature is, all we care about is that at least one signature for that existed. Such an example of where something exists is known as a witness.

We don't care that what it is, well we do for auditing purposes, like in multi-sign setup where you have 1-of-3 people that are able to spend a particular output, perhaps you would really like to know which person signed, which we will solve later. Inside a transaction, you still don't care.

Wouldn't it be nice to just drop the signatures? The reason why we can't do this is because the signature is part of the transaction hash. If we would just drop the sig from the transaction, the block wouldn't validate, you wouldn't be able to prove an output spend came from that transaction, so that's not something we could do. But let's simplify the problem. What if we could redesign Bitcoin from scratch? What if you're designing an altcoin, there's really no reason why you would want to do this in Bitcoin. This is actually something we did in sidechain alpha.

You would mark the signature data as special. Here indicated by green color on this slide. Everything but the green part goes into the hash of a transaction. The signature doesn't. It's just a piece of data that's still there, but we don't consider it part of the transaction.

This redesign would allow you to drop this data. For tampering puproses and other reasons, you still want to make blocks commit to the signatures. A node relaying a fully-valid block with all signatures in it that are all valid, would relay it to another node, and just change the signature data. This node does not see that the block is being tampered with. The node would see an invalid block, and you don't want that to happen. So we still need to make the blocks themselves, not the transactions, but the block commits to the signatures of transactions.

One way to do this is change the merkle tree that blocks have to commit to the transaction IDs, into two-sided tree where one side refers to the transaction IDs without the signatures, and then there's a second tree, exactly constructed in the same way, but it contains the hashes of the witnesses/signatures.

What are the advantages of this? It allows you to drop the signatures from relay whenever you are relaying to a node that is not actually doing full-validation at the time. It also allows us to effectively prune this data from history, maybe we're fine with not all nodes in the network actually maintaining these gigabytes of signatures that are buried under years of proof-of-work now. To show you how much data this actually is, here's the red line is the blockchain data today, the green line is what it would look like without the signatures. It's a significant difference.

Maybe more importantly, this change also solves all forms of potential malleability. This is a big problem right now for all sorts of more complicated contracts that rely on being able to spend outputs of unconfirmed zero-confirmation transactions. The inherent problem is that the signature data already does not go into the data being signed, but it does go into the transaction id (txid). And we use the txid to refer to previous transactions, as a result the txid can change without impacting the validity of the block.

I recently withdrew [bip62](https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki) because bip62 cannot solve various forms of important malleability. In particular, ECDSA which we are using for signatures, has an inherent problem, namely you can, as a signer, as someone who has the private key, you can just change the signature into something else. This means that even with strong restrictions on what can be changed, you can never change, such as in 2-of-3 multisig, you cannot prevent only 1-of-3 of the participants from changing their signature. This is a form of malleability. The only people you would want to be able to construct the valid transaction are those who had that right in the first place that right, so only when all of them agree this should be possible. Separating signature data from the transaction completely solves this problem.

There are still malleability problems that remain, like Bitcoin selecting which part of the transaction is being signed, like the sighash flags. This remains possible, obviously. That's something that you opt-in to, though. This directly has an effect on scalability for various network payment transaction channels and systems like lightning and others.

This brings us to the actual full title of my talk, Bitcoin scalability with segwit. So far, I was talking hypothetically about the scheme presented so far, because the deployment would not be easy. All transaction data structures would have to be changed, which is a huge deployment friction.

This seemed like a hard problem. I personally dismissed this as a solution for a long time as something non-viable, until Luke-Jr discovered that it's possible to do this as a soft-fork. What we're going to do is inputs, we just deprecate the signature field inside of inputs. It's going to be an empty string from now on. Obviously, an empty signature is not going to be able to spend an actual output that requires a signature. Instead, the outputs do not push these scripts that we required to be satisfied, they would be encapsulated, it would be pushed as a piece of data. This allows us to, this effectively to every node, and every node not using this system, it's an ANYONECANSPEND. It's just an output that pushes data on the stack, the output doesn't do anything else. It's ANYONECANSPEND. In a soft-fork, we can add a new rule that restricts what's valid. We can add a rule that says whenever we see such a form of a script being pushed as a data element we give it a new meaning. Namely this is a new type of script that is able to instead of taking its inputs from the signature field, it takes it from the witness instead. The witness becomes a third part of the transaction in addition to the inputs and outputs of a transaction. For now it would only contain a signature.

So doesn't this change a transaction completely? It's just a realization that, whenever we relay data to an old node, we can drop the witness. To them, the transaction is valid without it. Because the witness does not impact the txid, you can say it's not really a part of the transaction, it's just another piece of data we relay along with the transaction instead.

The scheme we were using before, to make blocks commit to the witness data, is not possible because we cannot change the structure of the merkle tree because that would be a hard-fork. But we can build two separate merkle tree, one with commitments to witnesses, and one with commitments to transaction data as it is now and store the root of the merkle tree of the signatures in the coinbase transaction. This gives us almost the same power, except now it can be deployed as a soft-fork.

There are more things we can do here. These were discovered while this solution was being thought of. One is, we're really adding a new script type, and this script gets encapsulated in a PUSH op now. We could say every script could begin with a version byte. The reason for doing so is making it easier to do soft-forks. Right now, any time we want to introduce new functionality to Bitcoin Script, really the only possibility is to redefine an OP\_NOP. And the only redefinition we could do is make the OP\_NOP do something special, do a test, if it fails the transaction is invalid as a whole and if it's valid, it must have absolutely no effect at all. This is because even if it returns true, someone could add a negation after it, which would make something that went from valid to invalid, go from invalid to valid, which would make it a hard-fork. So this is the reason why previous soft-forks in particular, like CSV (CHECKSEQUENCEVERIFY) [bip112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki) and CLTV (CHECKLOCKTIMEVERIFY) [bip65](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki), the only thing they do is redefine that OP\_NULL. This is sad. There are way, way more nice improvements to Script that we could imagine. By adding a version byte with semantics like, whenever you see a version byte that you don't know, consider it ANYONECANSPEND. This allows us to make any change at all in the Script language, like introducing new signature types like <a href="http://diyhpl.us/wiki/transcripts/blockchain-protocol-analysis-security-engineering/2018/schnorr-signatures-for-bitcoin-challenges-opportunities/">Schnorr signatures</a>, which increase scalability by reducing the size of multisig transactions dramatically, or other proposals like merklized abstract syntax trees which is a research topic mostly. But there really are a lot of ideas for potential improvement to Script that we cannot do right now. This would enable it for free by just adding one more byte to all Script scripts.

Another thing it allows us is fraud proofs. Bitcoin right now only has two real security models. Either you are a full-node and maintain a full UTXO set and you are able to validate every Script script and you can validate all rules in the system; the only alternative is what SPV clients do, validate the headers. In the Bitcoin whitepaper, in its simplified payment verification (SPV) section, suggested to use fraud proofs. A rule violation announcement could be made by peers on the network, to provide data that proves the violation was made, and relay it along the network, which other nodes who don't do validation could then pick up. This doesn't work for SPV, but it could work for something between a full-node and SPV. One that downloads the full blocks, but doesn't maintain the UTXO set. So we could have a Bitcoin model where you only choose to validate a certain percentage of the UTXO set, for example 1%-10% and then you rely on the censorship resistance of the network. So your security assumption goes from "I am not being sybilled and there is no collusion attack by miners", goes to "and I am not censored from other nodes which altogether do 100% validation" (for receiving fraud proofs).

This is a far-more scalable full-node or partial-full-node model that we could evolve to. It's a security tradeoff. It's certainly not one that everyone would want to make, but it doesn't effect those who do.

The problem with this, is that Bitcoin right now does not have any means of (compactly) proving two of its consensus rules. For almost all rules, there's a way to prove them. In double spends, we show the two transactions. In script validation you show inputs and outputs. But there are two that you can't. One is subsidy fraud, where miners would introduce too many coins,and you cannot prove in a short way that the amount introduced—the fee—was wrong, because you would need to show all the inputs to all the transactions in the block, to do this. So by simply adding the input amounts to the witness data, you could get a consensus rule that the witness needs to have correct input data, it could be pruned like all the other witness data, and it allows us to prove that.

In addition to that, you could also right now not prove the spending of a non-existing input. If it's an existing input that was previously spent, you can, but you cannot actually, if someone claims "oh I'm spending this particular output", how do you go prove that the input wasn't there? We have no means of proving that it wasn't there (you can't prove a negative). Every block could commit to the entire UTXO set, which would allow you to make a proof. But there's an easier solution here, and I felt stupid when gmaxwell told me about this. You can instead make the witness contain the height of the block which produced the outputs, and the position within the block, and if it's wrong you can just show a proof that the claimed to be there transaction is not there and so it is wrong. So by adding maybe a dozen bytes to the witness for every input, we could make these proofs for every single consensus rule in Bitcoin right now.

Importantly, it also allows us to do soft-fork scaling. All of what I have been talking about is implemented in a prototype, but it's not quite ready for production. I did a simulation of what the block size looks like if we had switched to this model from a start. The black line at the top is the actual size, the dark line is the base data without witness, light blue is the witness data. The dark blue part is the only thing that all nodes see. It is the only part that the block size limit applies to. So I guess you can hear this coming, we can increase the block size with a soft-fork.

This is my proposal that we do right now. We implement segregated witness right now. What we do is discount the witness data by 75% for block size. So this enables us to say we allow 4x as many signatures in the chain. What this normally corresponds to, with a difficult transaction load, this is around 75% capacity increase for transactions that choose to use it. Another way of looking at it, is that we raise the block size to 4 MB for the witness part, but the non-witness has the same size, and this makes it a soft-fork. The reason for doing this discount is that it disincentivizes UTXO impact because the signature doesn't go into the UTXO set and can be pruned.

Why 4x? Well, 0.12 just made transaction validation like 7x faster I believe with [libsecp256k1](https://github.com/bitcoin/secp256k1). The impact on relay, there are technologies that have been discussed earlier like IBLT and weak blocks. So segwit fixes malleability, enables simpler Script upgrades, enables fraud proofs, allows pruning witness data for historical data, reduces bandwidth usage for light nodes and historical sync, and it's P2SH compatible for old senders so that non-upgraders can still send funds. This gives us time for IBLT and weak blocks to develop, so that we can see whether the relay and propagation stuff can have time to work.

We can switch to a cost-based metric in a hard-fork soon, [which Jonas will be talking about later](https://btctranscripts.com/scalingbitcoin/hong-kong-2015/validation-cost-metric/). This with the intention to give time for high-level payments solutions to develop and hopefully we end up in a future where there is less demand for getting everything on the blockchain.

Questions?

Q: Can we change the type of signature to Schnorr signatures? Would we do that?

A: Yes. I would like to switch to Schnorr signatures. I'm not aiming for that right now. I think what we need to get in place is to get a framework in place to allow these upgrades to happen.

Q: Telecommunications bottleneck, to validation as bottleneck?

A: There are bottlenecks for different types of infrastructure in Bitcoin. The only reason why I'm fine with this is because we know of technologies which can improve the relay. I think the situation, we do need to work on making relay faster, but we know how to do that. So this is a multiplier.

Q: So your proposal depends on other things that weren't listed?

A: No, they are listed.

Q: So 4 MB is okay depends on other changes not listed?

A: We'll talk in a bit.

Q: So you said moving the signature data outside the UTXO set..

A: It's never been in the UTXO set. But we are now able to treat the part of the transaction, which has no impact on the UTXO set. The signature data has never had an impact, but it was bounded to the rest of the transaction, which did have an effect on the UTXO set.


    18:36 < aj> oh, huh, i guess segwit moving signatures out of the block means that the size of cross-chain witness proof things (what are they called?) for pegged sidechains is kind-of irrelevant...


(same-day) follow-up: <https://www.reddit.com/r/Bitcoin/comments/3vq8hm/multiple_new_bip_proposals_coming_up_on_day_2_of/> and <https://www.reddit.com/r/Bitcoin/comments/3vql34/segregated_witness_and_its_impact_on_scalability/>

previously:

<https://github.com/ElementsProject/elements/commit/663e9bd32965008a43a08d1d26ea09cbb14e83aa>

<http://gnusha.org/bitcoin-wizards/2015-11-19.log>

<https://www.reddit.com/r/Bitcoin/comments/3ngtx5/could_the_segregated_witness_part_of_the/cwnthlh>

<https://people.xiph.org/~greg/blockstream.gmaxwell.elements.talk.060815.pdf> and <http://diyhpl.us/wiki/transcripts/gmaxwell-sidechains-elements/> and <https://github.com/ElementsProject/elementsproject.github.io>

somewhat simplified explanation of segwit <https://www.reddit.com/r/Bitcoin/comments/3th0py/sipa_proposes_a_fork_of_mainnet_enabling/cx6cunn>

more follow-up and questions: <http://gnusha.org/bitcoin-wizards/2015-12-07.log> and some [roadmap](http://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-December/011865.html), also aj [1](http://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-December/011868.html) [2](http://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-December/011869.html)

reddit follow-up: [1](https://www.reddit.com/r/Bitcoin/comments/3vql34/segregated_witness_and_its_impact_on_scalability/) [2](https://www.reddit.com/r/Bitcoin/comments/3th0py/sipa_proposes_a_fork_of_mainnet_enabling/cx6cunn) [3](https://www.reddit.com/r/Bitcoin/comments/3vs524/eli5_segregated_witness/cxqerrv) [4](https://www.reddit.com/r/Bitcoin/comments/3vurqp/greg_maxwell_capacity_increases_for_the_bitcoin/)

further video (2015-12-14): <https://www.youtube.com/watch?v=NOYNZB5BCHM>
