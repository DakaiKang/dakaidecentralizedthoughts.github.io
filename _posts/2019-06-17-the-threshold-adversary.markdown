---
title: The threshold adversary
date: 2019-06-17 09:11:00 -04:00
tags:
- dist101
- models
author: Ittai Abraham
---

Once we fix the communication model (synchrony, asynchrony, or partial synchrony see [here](https://decentralizedthoughts.github.io/2019-06-01-2019-5-31-models/)), we then need some way to limit the power of the adversary. 


> Power tends to corrupt, and absolute power corrupts absolutely.
> -- <cite> [John Dalberg-Acton 1887](https://en.wikipedia.org/wiki/John_Dalberg-Acton,_1st_Baron_Acton) </cite>

As John observed almost 150 years ago, if the adversary has no limits to his power, then there is very little we can do. Let's begin with the traditional notion of a _threshold adversary_ as used in Distributed Computing and Cryptography in order to limit the power of the adversary. 

## Traditional threshold adversary 
The simplest model is that of a _threshold adversary_ given a static group of **_n_** nodes. 

A threshold adversary is an adversary that controls some **_f_** nodes. There are three typical thresholds:
1. $n>f$ where the adversary can control all parties but one. Often called the _dishonest majority_ adversary (or [anytrust](https://www.ohmygodel.com/publications/d3-eurosec12.pdf) model).
2. $n>2f$ where the adversary controls a minority of the nodes. Often called the _dishonst minority_ adversary.
3. $n>3f$ where the adversary controls less than a third of the nodes. Often called the _dishonst third_ adversary.

There are many examples of protocols that work in the above threshold models. Here are some classics:
1. The Dolev, Strong [Broadcast protocol](https://www.cs.huji.ac.il/~dolev/pubs/authenticated.pdf)  solves Byzantine broadcast assuming an adversary that can control up to $n-1$ parties out of $n$ in the Synchronous model. See [this post](https://decentralizedthoughts.github.io/2019-12-22-dolev-strong/) for details.
2. Lamport's [Paxos](https://lamport.azurewebsites.net/pubs/lamport-paxos.pdf) protocol solves state machine replication assuming an adversary that can control less than $n/2$ parties out of $n$ in the Partially synchronous model.
3. Ben Or's [randomized protocol](http://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/p27-ben-or.pdf) solves Byzantine agreement in the Asynchronous model assuming a $5f<n$ threshold. This was later [improved by Bracha](https://core.ac.uk/download/pdf/82523202.pdf) to the optimal $3f<n$ bound. See [this series of posts](https://decentralizedthoughts.github.io/2022-03-30-asynchronous-agreement-part-one-defining-the-problem/) for details.

## Generalized bounded resource threshold adversary 
In this model instead of having a static total _n_ that represents the total number of nodes or participating parties, we assume some general **_bounded resource_** (see Szabo definition of [scarce object](https://nakamotoinstitute.org/scarce-objects/)). A generalized threshold adversary is an adversary that controls some **_fraction_** of this _bounded resource_. Again, there are three typical thresholds:

1. The adversary can control any amount of the bounded resource (but not all of it).
2. The adversary controls at most a minority of the total resource. Sometimes there is an explicit parameter $0< \epsilon << 1$ and the assumption is that the adversary controls at most $1/2 - \epsilon$ fraction of the bounded resource (say at most 49 percent).
3. The adversary controls less than $1/3$ (or less than $1/3 - \epsilon$) fraction of the total bounded resource. 

There may be mechanisms where the total amount of the bounded resource changes over time, so the above restrictions have to hold at all times.

Let's consider two common examples of potential bound resources:

1. In Nakamoto Consensus (the consensus mechanism used by Bitcoin), one can consider the bounded resource being the total CPU power of the participants. The assumption is then that the adversary controls less CPU power than the honest nodes (a minority adversary). In fact, the Nakamoto authors explicitly write the assumption of a resource-bounded _minority_ adversary:
> The system is secure as long as honest nodes collectively control more CPU power than any cooperating group of attacker nodes.
> -- <cite>[Bitcoin whitepaper](https://bitcoin.org/bitcoin.pdf) </cite>

2. In systems that use [proof-of-stake](https://www.investopedia.com/terms/p/proof-stake-pos.asp) the assumption is that the bounded resource is some finite set of coins. It is then natural to assume that the adversary controls a threshold of the total coins. On can often map voting power based on the relative amount of coins at a given time. For example, Tendermint mentions the total voting power of the adversary is bounded by a third:
> it requires that the total voting power of faulty processes is smaller than one-third of the total voting power
> -- <cite> [Tendermint whitepaper](https://arxiv.org/pdf/1807.04938.pdf) </cite>


## A note on _stake_ based bounded resources
Basing the security assumption on the adversary controlling a threshold of some set of coins generated by the system has both advantages and disadvantages.
1. On the one hand, using a resource that is controlled by the platform allows to control the resource allocation in ways that are not possible when the resource is external. In particular, the platform can create _punishments mechanisms_ to better incentivize honest behavior. For example, [Buterin suggests]((https://medium.com/@VitalikButerin/minimal-slashing-conditions-20f0b500fc6c)) conditions to detect malicious behavior and then punish the adversary by reducing (slashing) the offender's stake.
2. On the other hand, if the value of the stake depends on the platform, this may cause some circular reasoning about security and in particular a bootstrapping problem. If the value of the total coins is too high, then it may happen that not enough honest entities will sign up and the system will lose liveness. If the value of the total coins is initially too low, then an attacker can gain early monopoly power. One option is to bootstrap proof-of-stake using an existing decentralized high-value Proof-of-work coin or high-value traditional fiat (see [here](https://bitcoinist.com/visa-paypal-10-million-run-facebook-coin-node/)). This type of solution assumes that the adversary does not already have monopoly power on the bootstrapping resource. Another option is to bootstrap a Proof-of-stake system from an existing high-value Proof-of-work system (for example, see [eth2.0](https://github.com/ethereum/eth2.0-specs)).

## A note on Distributed Computing vs Game Theory
Note that this model has a clear separation between honest parties and a monolithic adversary. This model captures some real-world interactions where worst-case assumptions capture important attack vectors. There is a completely different view that aims to reason about how _rational_ parties will make decisions in cases of strategic interaction. This is captured in solution concepts of Game Theory. In later posts, we will cover the amazing connections between these seemingly different points of view.


Please leave comments on [Twitter](https://twitter.com/ittaia/status/1141475000278556674?s=20)

