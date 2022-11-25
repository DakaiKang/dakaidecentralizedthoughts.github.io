---
title: Two Round HotStuff
date: 2022-11-24 04:00:00 -05:00
tags:
- dist101
author: Ittai Abraham
---

In the first part of this post we describe a single-shot variation of Two Round HotStuff (see the [HotStuff v1 paper](https://arxiv.org/pdf/1803.05069v1.pdf)) using [Locked Broadcast](https://decentralizedthoughts.github.io/2022-09-10-provable-broadcast/) that follows a similar path as our previous post on [Paxos](https://decentralizedthoughts.github.io/2022-11-04-paxos-via-recoverable-broadcast/) and [Linear PBFT](https://decentralizedthoughts.github.io/2022-11-20-pbft-via-locked-braodcast/). In the second part, we describe a fully pipelined multi-shot State Machine Replication version of Two Round HotStuff that is similar to [Casper FFG](https://arxiv.org/abs/1710.09437) and [Streamlet](https://decentralizedthoughts.github.io/2020-05-14-streamlet/).


The simplified single-shot Two Round HotStuff variation captures the essence of the [Tendermint](https://tendermint.com/static/docs/tendermint.pdf) view change protocol (also see more recent [Tendermint paper](https://arxiv.org/pdf/1807.04938.pdf)) which reduces the size of the view change messages relative PBFT.

The model is [Partial Synchrony](https://decentralizedthoughts.github.io/2019-06-01-2019-5-31-models/). We assume the standard $f<n/3$ [Byzantine failures](https://decentralizedthoughts.github.io/2019-06-07-modeling-the-adversary/), but for safety we prove an even stronger **accountable safety** statement inspired by [Casper FFG](https://arxiv.org/abs/1710.09437).


# Part one: single shot with rotating leaders

Here we use the same view based framework as in the [PBFT post](https://decentralizedthoughts.github.io/2022-11-20-pbft-via-locked-braodcast/):

1. The goal in this part is Single-shot Consensus with External Validity.
2. We use a view based protocol where view $v$ is the time interval $[v(10 \Delta),(v+1)(10 \Delta))$.
3. The protocol uses [Locked broadcast](https://decentralizedthoughts.github.io/2022-09-10-provable-broadcast/).

Here is the pseudocode which is (almost) the same as the [PBFT post](https://decentralizedthoughts.github.io/2022-11-20-pbft-via-locked-braodcast/):



```
Two Round HotStuff single-shot consensus protocol

View 1 primary:
    LB(1, val, val-proof)

View v>1 primary:
    (p, p-proof) := RML(v)

    if p = bot then 
        LB(v, val, val-proof)
    otherwise 
        LB(v, p, p-proof)
        

Upon delivery-certificate dc for (v,x)
    send (v, x, dc) to all

All parties:

Upon valid delivery-certificate dc for (v,x)
    output x    
```

A minor difference is that when $p= \bot$ then $p{-}proof$ is not sent.


Two major differences in the sub-protocols $RML$ and external validity of $LB$:
1. *Recover Max Lock* protocol: Instead of using ```PBFT-RML(v)``` which returns a set of $n-f$ lock-certificates, we use ```TNDRMNT-RML(v)``` which returns only **one** lock-certificate. This is where Tendermint based protocols (like Two Round HotStuff) save on message size.
2. The *external validity* of the *Locked broadcast* $EV_{\text{LB}}$ is changed from $EV_{\text{LB-PBFT}}$ to $EV_{\text{LB-TNDRMNT}}$ to work with the change above in the Recover Max Lock protocol.

We go over these changes in more detail:


### Recover Max Lock for Tendermint based protocols

The ```TNDRMNT-RML(v)``` protocol for view $v$ finds the lock-certificate with the highest view and simply returns the associated value and the lock-certificate. 

The other major change in ```TNDRMNT-RML(v)``` is the additional wait of $\Delta$ at the primary (in addition to waiting for at least $n-f$). This is required for the liveness proof.

```
TNDRMNT-RML(v):

Party i upon start of view v:
    send <echoed-max(v, v', p, LC(v',p) )>_i 
        of the highest view v' it has a lock-certificate LC(v',p)
    or send <echoed-max(v, bot)>_i 
        it does not have any lock-certificate

Primary waits for at least n-f responses <echoed-max(v,*)> and Delta time:
    if all are bot then output (bot, bot)
    otherwise, output (proposal, LC(v',proposal)) 
        where proposal is associated with the highest view in responses
        and LC(v',proposal) is the associated lock-certificate
```

 
### External Validity for the Locked Broadcast of Tendermint based protocols


$EV_{\text{LB-TNDRMNT}}$ checks the validity relative to the ```TNDRMNT-RML(v)``` protocol. Unlike the $EV_{\text{LB-PBFT}}$ check, the $EV_{\text{LB-TNDRMNT}}$ verification is *stateful*, the check compares the input to the highest lock-certificate that the party has seen.

To be explicit about this, each part maintains state  ```my-lock``` that contains the lock-certificate with the highest view it saw. Denote by ```view(my-lock)``` the view number of this block-certificate. Initially ```my-lock := bot``` and ```view(my-lock) := 0```.

Define $EV_{\text{LB-TNDRMNT}}$:

For view 1, nothing changes, just check external validity of the consensus protocol: $EV_{\text{LB-TNDRMNT}}(1, val, val-proof)= EV_{\text{consensus}}(val, val-proof)$.

For view $v>1$ there are two cases:

1. Given a 3-tuple $(v, val, val{-}proof$:
    1. Check that $view(my{-}lock) = 0$.
    2. Output true if $EV_{\text{consensus}}(val, val{-}proof)=1$.
2. Otherwise, given a 3-tuple $(v, p, p{-}proof)$:
    1. Check that $p{-}proof$ is a valid lock-certificate $LC(v',p')$ for view $v'$ and value $p'$.
    2. Check that $view(my{-}lock) \leq v'$ 
    3. Output true of $p=p'$.

    
In words: the check makes sure that the proposed value has a valid lock-certificate that has a view that is at least as high as the view of the lock-certificate this server currently holds.
 
 
The ```TNDRMNT-RML(v)```  and $EV_{\text{LB-TNDRMNT}}$ are defined. This fully defines the single-shot consensus protocol. Let's prove Agreement and Liveness (the Validity argument is the same as in PBFT).

### Agreement (accountable safety)

**Accountable safety lemma**: Let $v^{\star}$ be the first view with a commit-certificate, say on $(v^\star, x)$, then for any view $v\geq v^\star$, if a lock-certificate forms for view $v$ with value $x' \neq x$ then at least $f+1$ parties can be detected as malicious.

*Note: this is stronger than proving Agreement assuming at most $f$ corruptions. It also says that no matter how many corruptions the adversary can do, if it breaks Agreement then at least $f+1$ parties can be detected (and perhaps punished).*


*Proof of accountable safety lemma*: Let $S_1$ be the set of $n-f$ parties that signed the lock-certificate for $v^\star$ and $S_2$ be the set of $n-f$ parties that signed the delivery-certificate for view $v^\star$. Let $v^-$ be the *first* view after view $v^\star$ for which there is a lock-certificate for a value $x' \neq x$. Let $L$ be the set of parties that signed this lock-certificates.

The first case is if $v^- = v^\star$. In this case due to quorum intersection, there are at least $n-2f \geq f+1$ parties in the intersection of $S_1 \cap L$ that must have violated the protocol by signing two different values in the same view. The two lock-certificates from $S_1$ and $L$ contain irrefutable cryptographic evidence of their misbehavior.

The second case is if $v^- > v^\star$, here we use the minimality assumption on $v^-$ to ensure the only non-$x$ lock-certificates must be for a view that is less than $v^\star$. In this case due to quorum intersection, there are at least $n-2f \geq f+1$ parties in the intersection of $S_2 \cap L$ that must have violated the protocol $EV_{\text{LB-TNDRMNT}}$ by accepting (and signing) on a proposal in view $v^- > v^\star$ that has a lock-certificate whose view is smaller than $v^\star$, but they signed (as members of $S_2$) that they had a lock-certificate in view $v^\star$. The delivery certificate from $S_2$ and the lock certificate from $L$ contain irrefutable cryptographic evidence of their misbehavior. 
This concludes the proof of Safety Lemma.

### Liveness

The only difference relative the Liveness proof for [PBFT](https://decentralizedthoughts.github.io/2022-11-20-pbft-via-locked-braodcast/) is that we need to show that all non-faulty parties will verify $EV_{\text{LB-TNDRMNT}}$ on the honest primary proposal. This is where the additional wait for $\Delta$ time in the gathering phase of ```TNDRMNT-RML(v)``` is required - it guarantees that after GST, by waiting for $\Delta$ time, the primary will hear from all non-faulty parties, to chooses the lock-certificate that is high enough to verify $EV_{\text{LB-TNDRMNT}}$ for all non-faulty parties. 

This concludes the proof of Liveness.


### Time and Message Complexity

Like PBFT, the time complexity is $O(f \Delta)$.

Unlike PBFT, the size of a proposal message is just one lock-certificate instead of $n-f$ lock-certificates. With a threshold signature setup, the size of a lock-certificate can also be a constant (in the security parameter). So the total number of messages is $O(n^2)$ and the size of each message can be just $O(1)$ words - where a word constants a constant number of values and cryptographic signatures.



# Part two: pipelined multi-shot State Machine Replication version of Two Round HotStuff

The protocol revolves around a data structure which is a chain of blocks and 2-3 certificates (each certificate is a set of $n-f$ signatures on the block). Lets define:

A ```block``` $B$ contains a triplet $B=(cmd, view, pointer)$ where *cmd* is an externally valid client command, *view* is the view this block was proposed in, and *pointer* is a link to a previous block (that has a smaller view).


A ```chain``` is just a chain of ```block```s that starts with an empty ```genesis block``` of view 0 and empty pointer. The views in the chain are monotonic but not nessisarily sequential. Some examples:
```
(Genesis,0)
(Genesis,0)<-(c1,1)
(Genesis,0)<-(c1,1)<-(c2,2)
(Genesis,0)<-(c1,1)<-(c2,2)<-(c4,4)
```

The last block which also has the highest view in a chain is called the ```tip```.

A ```block-cert``` for a block $B$ is a set of $n-f$ distinct signatures on $B$. We assume the ```genesis block``` implicitly has a block-certificate.

A ```valid chain``` is a chain of blocks with a ```commit-cert``` and a ```lock-cert```. Both are defined below:

A ```commit-cert``` for a block $B=(cmd, view, pointer)$ on a ```valid chain``` that is not the genesis is a ```block-cert``` for $B$, and a second block $B'=(cmd, view+1, pointer{-}to{-}B)$  with a ```block-cert``` for $B'$. Importantly, $B'$ does not have to be part of the ```valid chain``` but its view must be exactly one more than $B$'s view. By default, if there is no explicit pair of block-certificates, we say that the ```valid chain``` has a ```commit-cert``` on the  ```genesis block```. Given a ```commit-cert``` for block $B$, we call the chain from $B$ to the ```genesis block``` the ```committed chain```.

The ```lock-cert``` is a ```block-cert``` for a block $B$ that is on the ```valid chain``` but not part of the ```committed chain```.
 
The ```lock-cert``` can be the ```block-cert``` on the second block of the ```commit-cert```. In this case, this second block must be part of the ```valid chain```. By default, if there is no explicit Lock-certificates, we say that the ```valid chain``` has a ```lock-cert``` on the  ```genesis block```.

Some examples of valid chains:
```
(Genesis,0)

(Genesis,0)<-(c1,1)<-(c2,2)<-(c4,4)<-(c5,5)
Commit-cert on (c1,1), (c2,2). Lock-cert on (c2,2)

(Genesis,0)<-(c1,1)<-(c2,2)<-(c3,3)<-(c4,4)<-(c5,5)
Commit-cert on (c1,1)<-(c2,2). Lock-cert on (c4,4)

(Genesis,0)<-(c1,1)<-(c2,2)<-(c4,4)<-(c5,5)
Commit-cert on (c2,2)<-(c3,3). Lock-cert on (c5,5)

```

1. The first example is an empty chain. It implicitly has a commit-cert and lock-cert on the genesis.
2. Second example shows a commit-cert and lock-cert that share the block-cert of a block from view 2.
3. Third example shows a lock-cert is on a block of view 4 and there are more blocks after the lock-cert (in this case a block from view 5).
4. Forth example shows a commit-cert that uses a block-cert $(c3,3)$ that supports the block-cert on $(c2,2)$ but this supporting block $(c3,3)$ is **not** on the chain.


## The protocol

Each party stores a ```valid chain``` in a variable called  ```my-chain``` which stores the valid chain with the lock-certificate of highest view.


Protocol for view $v$:


On the start of view $v$, each party sends its signature on its valid chain to the primary:

```
Party i:

On start view v
    Send <"start view", v, my-chain>_i to view v primary
```

The primary of view $v$ waits for both $\Delta$ time and at least $n-f$ valid responses. There are two cases:
1. If $n-f$ parties send the *same* valid chain, and its tip is from view $v-1$ then:
    1. The primary creates a new block-cert for the tip and updates the lock-cert to be this new block-certificate ```chain.updateLockCert(tip(chain))```.
    2. If a new commit-cert is formed (because both the block in view $v-1$ and the block in $v-2$ now have block-certificates) then update the commit-cert to be this new pair ```chain.updateCommitCert()```.
    3. Finally the primary appends its new block as the new tip.
2. Otherwise the primary chooses the valid chain with the lock-certificate of the highest view and appends its new block to it as the new tip.

Finally the primary sends this as the proposal for view $v$ to all parties.


```    
Primary of view v:

On at least n-f valid <"start view", v, *> and waiting Delta time
    If n-f <"start view", v, chain> are the same and tip(chain).view = v-1
        chain.updateLockCert(tip(chain))
        If a new commit-cert created
            chain.updateCommitCert()
        chain.addTip(cmd, v)
        my-chain := chain
    
    Otherwise
        Let h-chain be the valid chain 
            with the lock-cert of highest view in <"start view", v, *> messages
        h-chain.addTip(cmd, v)
        my-chain := h-chain
        
    Send <"propose", v, my-chain> to all
```

When a party sees a proposal chain from the view $v$ primary it checks:
1. That its view is still $v$;
2. That this is the first such message from the primary;
3. That the proposal is a valid chain with a valid lock-cert and a valid commit-cert;
4. That all the blocks after the lock-cert of the proposed chain contain externally valid commands;
5. That the lock-cert of the proposed chain has a view that is at least as high as the view of the lock-cert of ```my-chain```.
6. If all these checks pass, then:
    1. If the proposed chain has a commit-cert for a higher view than that of  ```my-chain``` then execute the blocks in between the previous commit-cert and the new commit-cert.
    2. Update ```my-chain``` to the new proposed chain so it will be signed as part of the "start view" of view $v+1$.

```   
Party i:

    Upon <"propose", v, chain> from primary
        Check your view is v
        Check this is the first view v proposal
        Check that chain is valid (has valid lock-cert and commit-cert)
        Check that all blocks after chain.lock-cert are externally valid
        Check that chain.lock-cert.view >= my-chain.lock-cert.view
        
        If chain.commit-cert.view > my-chain.commit-cert.view
            execute commands between the two commit-certs
        my-chain := chain 
        
        
    If the view v timer expires then start view v+1
        
```


### Accountable Safety

**Theorem**: Assuming $f<n/3$, for any two ```committed chain```s, taken at any two times, one chain is a sub-chain of the other.

We prove this theorem via the following accountable safety lemma which is a stronger statement. It shows that even if the adversary controls more than $f$ parties, if agreement is violated then at least $f+1$ parties can be detected (and potentially punished). 


**Accountable safety lemma**: If there is a commit-cert that consists of two block-certs on two blocks $B_1$ and $B_2$ of consecutive views $v^\star$ and $v^\star +1$ and a lock-cert on block $B_3$ in view $v \geq v^\star$ such that $B_1$ is not a prefix of $B_3$ then at least $f+1$ parties can be detected as malicious.


*Proof of the accountable safety lemma:*

Let $v^-$ be the *first* view after view $v^\star$ for which there is a block-cert for a block $B'$ such that $B_1$ is not a prefix of $B'$. Let $L$ be the set of $n-f$ parties that singed the block-cert for $B'$.

Let set $S_1$ be the set of $n-f$ parties that singed the block-cert for $B_1$, $S_2$ be the set of $n-f$ parties that singed the block-cert for $B_2$.


The first case is if $v^- = v^\star$. In this case due to quorum intersection, there are at least $n-2f \geq f+1$ parties in the intersection of $S_1 \cap L$ that must have violated the protocol by signing two different blocks in the same view. The two lock-certs from $S_1$ and $L$ contain irrefutable cryptographic evidence of their misbehavior.

An example of this violation, members of $S_1 \cap L$ signed in view 2 on both $c2$ and $c4$.
```
(Genesis,0)<-(c1,1)<-(c2,2)<-(c3,3)
(Genesis,0)<-(c1,1)<-(c4,2)
                     
(B1,B2) = Commit-cert on (c2,2)<-(c3,3)
L = Lock cert on (c4,2)
```



The second case is if $v^- > v^\star$, here we use the minimality assumption on $v^-$ to ensure the only blocks with a lock-cert that do not contain $B_1$  must be for a view that is less than $v^\star$. In this case due to quorum intersection, there are at least $n-2f \geq f+1$ parties in the intersection of $S_2 \cap L$ that must have violated the protocol by signing on a proposal chain in view $v^- > v^\star$ that has a lock-cert whose view is smaller than $v^\star$, but they signed in view $v^\star +1$ (as members of $S_2$) that they had a lock-cert in view $v^\star$. The delivery certificate from $S_2$ and the lock certificate from $L$ contain irrefutable cryptographic evidence of their misbehavior.


An example of this violation, the members of $L\cap S_2$: in view $3$ they declared their highest lock-cert is from view 2, but in view 5 they validated a chain that has a strictly lower view lock-cert (either 0 or block-cert on view 1).
```
(Genesis,0)<-(c1,1)<-(c2,2)<-(c3,3)
(Genesis,0)<-(c1,1)<-(c5,5)
                     
(B1,B2) = Commit-cert on (c2,2)<-(c3,3)
L = Lock cert on (c5,5)
```



 
This concludes the proof of Safety Lemma.


### Liveness

**Theorem**: There will be a commit-cert after the first three consecutive honest primaries after GST.

*Proof sketch*
1. Assuming perfect clock synchronization to obtain view synchronization.
2. The primary waits for $\Delta$ time so it hears the valid chain with the lock-certificate of highest view among all non-faulty parties.
3. Need 3 consecutive honest parties. The first creates valid proposal; the second creates the first block-cert on it;  the third creates the second block-cert on the block with one view above it, so a commit cert is formed and sent to all parties.


### Using an authenticated data structure

The way the protocol is described, parties send back and forth the whole chain. This is not bandwidth efficient. Instead, we could view the chain as an authenticated data structure. For example, the hash of the tip can be used as a digest of a simple hash chain authenticated structure.

The protocol can send the digest of the chain instead of the chain. Any data this is missing for validation can be requested in a separate mechanism along with a proof that this is the correct data relative to the digest.