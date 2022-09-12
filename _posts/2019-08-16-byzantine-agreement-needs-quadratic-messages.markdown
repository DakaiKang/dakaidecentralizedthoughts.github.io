---
title: 'The Dolev and Reischuk Lower Bound: Does Agreement need Quadratic Messages?'
date: 2019-08-16 18:30:00 -04:00
tags:
- lowerbound
- dist101
author: Kartik Nayak, Ittai Abraham
---

How scalable is Byzantine agreement? Specifically, does solving agreement require the non-faulty parties to send a quadratic number of messages (in the number of potential faults)? In this post, we highlight the [Dolev and Reischuk](http://cs.huji.ac.il/~dolev/pubs/p132-dolev.pdf) lower bound from 1982 that addresses these fundamental questions.

**[Dolev and Reischuk 1982](http://cs.huji.ac.il/~dolev/pubs/p132-dolev.pdf): any deterministic Broadcast protocol that is resilient to $f$ Byzantine failures must have an execution where the non-faulty parties send  $> (f/2)^2$ messages.** 

In fact, we will observe that the result is stronger and holds even for omission failures:

**[Dolev and Reischuk 1982, (modern)](http://cs.huji.ac.il/~dolev/pubs/p132-dolev.pdf): any deterministic Broadcast protocol that is resilient to $f$** ***omission*** **failures must have an execution where the non-faulty parties send  $> (f/2)^2$ messages.** 


In 1980, [PSL](https://lamport.azurewebsites.net/pubs/reaching.pdf) showed the *first* feasibility result for consensus in the presence of Byzantine adversaries. However, their solution had an *exponential* (in $n$, the number of parties) communication complexity. An obvious question then is to figure out the lowest communication complexity that could be obtained. Dolev and Resichuk showed that the barrier to quadratic communication complexity cannot be broken by deterministic protocols.

At a high level, the Dolev and Resichuk lower bound says that if the non-faulty always send few messages (specifically $< (f/2)^2$), then the adversary can cause some non-faulty party to receive no message! The party that receives no message has no way of reaching agreement with the rest. We use a trivial indistinguishability argument and create a world where a party $p$ receives no message at all. Thus, party $p$ cannot distinguish between a world where the designated sender sends 0 vs a world where it sends 1.

Here’s the proof intuition: In any set of $f/2$ parties, if each of these parties receives $> f/2$ messages from non-faulty parties, then we have a protocol with $> (f/2)^2$ messages. So, if there exists a protocol sending fewer messages, there must exist one party, say $p$, that receives $\leq f/2$ messages. Now imagine that all of the parties sending messages to $p$ (there can be at most $f/2$ of them) are corrupt. If these corrupt parties omit messages and do not send anything to $p$, then it may output a value that is not the same as the other non-faulty (consistency violation) or not output any value (termination violation).

Now let us formalize this intuition. Consider a broadcast problem, where the *designated sender* has a binary input. First, we need to guarantee that the isolated party $p$ will indeed not decide like all the other non-faulty parties. Observe what happens to a party that receives no messages. It will either not decide 0 or not decide 1. Without loss of generality, assume that a majority of parties (other than the designated sender) that receive no message will not decide 0. Let $Q$ be this set of parties and note that $\|Q\| \geq (n-1)/2$.

We will prove the theorem by describing two worlds and using indistinguishability for all honest parties. Here we go.

**World 1:** 

<p align="center">
  <img src="/uploads/dr-world1.png" width="256" title="DR world 1">
</p>

In World 1, the adversary corrupts a set $V \subset Q$ of $f/2$ parties that do not include the designated sender. Let $U$ denote the remaining parties (not in $V$). All parties in $V$ run the designated protocol but suffer from omission failures: each $v \in V$ (i) omits the first $f/2$ messages sent to them from parties in $U$, and (ii) omits all messages it sends to and receives to and from parties in $V$. Suppose the non-faulty designated sender has input 0. So from the validity property all non-faulty parties must output 0.

**World 2:**

<p align="center">
  <img src="/uploads/dr-world2.png" width="352" title="DR world 2">
</p>

If the protocol has the non-faulty parties send at most $\leq (f/2)^2$ messages, then there must exist some party $p \in V$ that receives $\leq f/2$ 
messages. In World 2, the adversary does everything as in World 1, except (i) it does not corrupt party $p$, and (ii) it corrupts all parties in $U$ that send messages to $p$ (this may also include the designated sender). Messages sent by these corrupt parties to $p$ are omitted. Since $p$ receives $\leq f/2$ messages in World 1, at most $f$ parties are corrupted in World 2 ($\leq f/2$ senders and $\|V\| = f/2$).

What do honest parties in $U$ output in World 2? We argue that they will output 0. Observe that for the non-faulty parties, the two worlds are indistinguishable. Since the protocol is deterministic, they receive exactly the same messages in both worlds. However, since party $p$ does not receive any messages and $p 
\in Q$, then it will not output 0, so will either violate the agreement property (if it outputs 1) or violate the termination property (if it does not output anything).

The lower bound uses the fact that the protocol is deterministic. There have been several attempts at circumventing the lower bound using **randomness** and even against an adaptive adversary. Here are a few notable ones:
- [King-Saia](https://arxiv.org/pdf/1002.4561.pdf): Through a sequence of fascinating new ideas, King and Saia presented a beautiful information-theoretic protocol that broke the quadratic communication complexity. Their protocol uses randomness and assumes that honest parties can erase data - so if they later get corrupt the adversary cannot extract the erased data. 
- [Algorand](https://www.sciencedirect.com/science/article/pii/S030439751930091X?via%3Dihub) uses randomness to cryptographic techniques to form small committees. Algorand assumes the adaptive adversary cannot cause the corrupt parties to remove the in-flight messages that were sent before the party was corrupted.
- In PODC’19, we [generalized this lower bound for randomized protocols](https://arxiv.org/abs/1805.03391).

[Randomized version of Dolev-Reischuk.](https://users.cs.duke.edu/~kartik/papers/podc2019.pdf) Any (possibly randomized) Byzantine Agreement protocol must in expectation incur at least $\Omega(f^2)$ communication in the presence of a strongly adaptive adversary capable of performing "after-the-fact removal", where $f$ denotes the number of corrupt parties.


**Remark:**
The bound is presented for Broadcast (not Agreement). In terms of feasibility, [both problems are equivalent](https://decentralizedthoughts.github.io/2020-09-14-broadcast-from-agreement-and-agreement-from-broadcast/) and each of them can be reduced from the other. However, communication complexity remains the same only when Byzantine Broadcast is realized using Byzantine Agreement; the sender can send the value to all parties and they can run a Byzantine Agreement protocol. Thus, for communication complexity, showing a bound on Byzantine Broadcast is strictly better.

This post was updated in November 2021 to reflect that the lower bound holds for omission failures.

Please leave comments on [Twitter](https://twitter.com/kartik1507/status/1162564876721692675?s=20) 

