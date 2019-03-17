# Open Mixer: [openmixer.io](openmixer.io)

Note to reviewer: the more I explored this space, the more convinced I became that there was an opportunity to actually launch a new mixer. My motivation is outlined in the first section, and I'd love to get feedback about the validity of my thinking from other engineers (which is why I'm interested in working at gemini). However for the busy engineer that's just looking to evaluate my implementation of the Jobcoin mixer, I discuss it [here](#implementation), and have published my solution at [openmixer.io](http://openmixer.io).

## Exploration of crypto mixers

Open Mixer is a POC for a digital asset mixing service which focuses on transparency. This POC mixes [JobCoins](https://jobcoin.gemini.com/sanitary/api) - a "dummy" digital asset that's significantly easier to work with. 

As most blockchains are transparent ledgers, a mixers primary utility is provided by moving transaction information "off-chain" (generally to a service's servers). For example, without using a mixing service, once your barista receives your coffee transaction they would be able to view what the input of that tx was (and the input of that one, and the input of that one, etc). Ultimately he or she would have a pretty good chance at being able to guess your income, and your employer would be able to tell what you're doing with your salary. Not ideal.

Mixing services are one of the many ways you can increase your privacy when using public ledgers. The idea is that when your barista goes to lookup the inputs of your transaction they go to a transaction with many inputs in them, and lots of outputs in them. When many people use a mixer, who's sending what to whom is not stored on the blockchain, but rather on the mixer itself.

Unfortunately this requires that you trust the mixing service with your funds, though there are some trust-less ways to perform mixing ([Coinjoin](https://en.bitcoin.it/wiki/CoinJoin) for example), though these services require specialized wallets (and are likely to suffer from lower volume as a result). 

While exploring the space of mixers I had trouble finding an open source mixer which is available without Tor. Furthermore I could not find any mixing service which made it's current transactional volume public, a critical piece of information that you'd want prior to sending your transaction through the mixer. Open Mixer aims to make transparency it's highest priority. 

## Implementation 

The two primary ways an analyst may be able to untangle a payment (that I could think off) would be tx time and tx amount.

* If the mixer predictibly takes roughly 10 minutes to mix every payment and you send it 10JCs and 10 minutes later a charity you're interested in supporting receives 10JCs, that tx is likely coming from you.
* If you send 13.50JCs to a mixer and over the course of a few days (say you solved the time problem) Edward Snowden receives 13.50JCs, that tx is likely coming from you.

Therefore the most effective way to use a mixing service like this one is to send an amount larger than you're interested in sending to someone. List a handful of outputs (one or more of which is the addresses of the person you want to transact with), and be willing to wait a suffiently long time.

When you view the site, you'll specify how many JCs you're going to be depositing (this is the amount we'll be waiting for), you'll also specify your outputs, their relative weights, and how long you're willing to wait before the last payemnt is sent.

Once you hit submit, your browser will `post` this `TxSpec` to the backend. The backend will perform the scheduling of this payment asyncronously, and will return a `txid`. The frontend can use this `txid` to poll for updates.

In the meantime, the backend will (in a separate goroutine) poll the wallet until the `Deposit Address >= Input`. Once this is done, for each of the transaction it will come up with `n` random numbers which sum to the total amount of time you were willing to wait. It will loop through each output, waiting the respective amount of time before sending the money to that output from the mixer's address.

## Some more technical details

The implementation has 2 components: a client (react based frontend) and a server (api written in go). The codebase is separated accordingly. 

### Frontend

The frontend is essentially setup like this:
```
<App>
	<NewTx txDone={callback} />
	{ if done
		return (
			<TxStatus />
		)
	}
</App>
```

# Areas of improvement:

* Currently our backend implementation is stateful and not terribly fault tolerant. When a payment is "scheduled" the details (state) of that payment live within a goroutine. If we were to replicate the backend, replica1 wouldn't be able to access status information for payments that were scheduled from replica2. If we sufer a server failure, that state is lost for ever, and not all of our client's outputs will be paid out. In this situation perhaps that money could be reallocated to hiring developers to make the mixer stateless. 
* We rely on polling in two places: the frontend polls the backend for updates regarding payment scheduling. The backend polls the "wallet" for updates regarding deposit addresses. Ideally we'd leverage a more event driven communication flow as we began to experience scale between the frontend and backend. Good technology candidates for this might be websockets or server send events (though some attention would need to be given to ensuring that the solution remain stateless). Polling between the backend and "wallet" can be solved (in the case of bitcoin) by pointing our backend to a [node that supports events](https://bcoin.io/guides/events.html).
* Currently a user can specify the "spread-out-ness" of their transactions temporally. It would be nice for a user to also be able to specify a minimum amount of time their transaction has to sit in the mixer, and then spread out the output payments.
