# Streaming Tutorial

## What you need before you start:

* complete the [Letter Shop](./letter-shop) tutorial first

## What you'll learn:

* convert the proxy from the Letter Shop tutorial into a payment tool
* using the `Pay` header in your shop and your payment tool
* using the ILP packet
* streaming payments
* how to use trustlines to speed things up
* Bilateral Transfer Protocol (BTP) and its relation to ILP

## The Pay Header

We'll change the Letter Shop from the previous tutorial a bit, to `shop2.js`:
```js
const IlpPacket = require('ilp-packet')
const http = require('http')
const crypto = require('crypto')
const Plugin = require('ilp-plugin-xrp-escrow')
function base64 (buf) { return buf.toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '') }
function hash (secret) { return crypto.createHash('sha256').update(secret).digest() }
function hmac (secret, input) { return crypto.createHmac('sha256', secret).update(input).digest() }

let users = {}

const plugin = new Plugin({
  secret: 'ssGjGT4sz4rp2xahcDj87P71rTYXo',
  account: 'rrhnXcox5bEmZfJCHzPxajUtwdt772zrCW',
  server: 'wss://s.altnet.rippletest.net:51233',
  prefix: 'test.crypto.xrp.'
})

plugin.connect().then(function () {
  plugin.on('incoming_prepare', function (transfer) {
    const ilpPacketContents = IlpPacket.deserializeIlpPayment(Buffer.from(transfer.ilp, 'base64'))
    const parts = ilpPacketContents.account.split('.')
    // 0: test, 1: crypto, 2: xrp, 3: rrhnXcox5bEmZfJCHzPxajUtwdt772zrCW, 4: userId, 5: paymentId
    if (parts.length < 6 || typeof users[parts[4]] === 'undefined' || ilpPacketContents.amount !== transfer.amount) {
      plugin.rejectIncomingTransfer(transfer.id, {}).catch(function () {})
    } else {
      const { secret, res } = users[parts[4]]
      const fulfillment = hmac(secret, ilpPacketContents.account)
      const condition = hash(fulfillment)
      if (transfer.executionCondition === base64(condition)) {
        plugin.fulfillCondition(transfer.id, base64(fulfillment)).then(function () {
          const letter = ('ABCDEFGHIJKLMNOPQRSTUVWXYZ').split('')[(Math.floor(Math.random() * 26))]
          res.write(letter)
        }, function (err) {
          console.error(err.message)
        })
      } else {
        console.log('no match!', { secret, fulfillment, condition, transfer })
      }
    }
  })

  http.createServer(function (req, res) {
    const secret = crypto.randomBytes(32)
    const user = base64(crypto.randomBytes(8))
    users[user] = { secret, res }
    console.log('user! writing head', user)
    res.writeHead(200, {
      'Pay': [ 1, plugin.getAccount() + '.' + user, base64(secret) ].join(' ')
    })
    // Flush the headers in a first TCP packet:
    res.socket.write(res._header)
    res._headerSent = true
  }).listen(8000)
})
```

Starting with the last part, `http.createServer`, you can see the flow of the http server
is a bit simpler; when a request comes in it sends headers, and
then the body will be sent letter-by-letter, as payments come in. The `Pay` header contains 3 parts:

* amount (in this case, the price of one letter, in XRP)
* user-specific destination address
* a Base64-encoded conditionSeed

Note how we are appending `'.' + user` to the shop's Interledger address! This is a special feature of Interledger
addresses, they can be subnetted endlessly, just add another `.` at the end to convert an account address
to a ledger prefix, and then add new sub-accounts after that. In this case, we want to know which user is paying
for letters, so by telling each user a different Interledger sub-address, we can neatly keep our users apart.

When a transfer comes in, the server opens the [ILP packet](https://github.com/interledgerjs/ilp-packet).
In there is a destination account and a destination amount, which will usually (if all went well) be equal
to the amount of the transfer. In later tutorials, we will see how one ILP payment can be chained together
in multiple transfers, but for now, we will only use single-transfer payments.

Whereas in the previous tutorial, the fulfillment was generated randomly, here it's generated deterministically from
the combination of the shop's interledger address, the user's userId, this payment's paymentId, and the
conditionSeed (as a buffer) that was sent to this user base64-encoded in the `Pay` header, for example:

```js
amount = '1'
shopAddress = 'test.crypto.xrp.rrhnXcox5bEmZfJCHzPxajUtwdt772zrCW'
userId = 'UUxOFrawp0A'
conditionSeed = Buffer.from('Agli74z_HjSNuTE1rTIr7QCkzWdIA3QdKoPMYHhw1I4', 'base64')
xPayHeader = [ amount, shopAddress + '.' + userId, base64(conditionSeed) ]
paymentId = '1'
paymentPacketContents = {
  amount: xPayHeader[0],
  account: xPayHeader[1] + '.' + paymentId,
  data: ''
}
fulfillment = hmac(conditionSeed, paymentPacketContents.account)
condition = sha256(fulfillment)
```

To run this new version of the shop, type this into your terminal:

```sh
npm install ilp-packet
node ./shop2.js
```

The ILP paacket is not very useful in a one-transfer payment, but in future tutorials, we see how connectors can forward an
incoming transfer, and thus connect one ledger to other ledgers. When that happens, we say the whole sender
to receiver process is the "payment", and each link in the chain is a "transfer", so one payment consists of 
one or more transfers. The sender can then put information in the ILP packet which the receiver can use
to generate the fulfillment, and if that information would be tempered with by connectors along the way,
this would be detected because the fulfillment would not match.
The ILP Payment Packet is serialized into OER and then Base64-encoded before it's added to the transfer as a Memo.
Note that this assumes that the ledger supports adding a memo to the transfer, but luckily, most ledgers do.

## Payment tool

The following is mainly a mix between the `pay.js` and `proxy.js` scripts from the Letter Shop tutorial,
that can pay for content in reaction to an `Pay` header, and then stream that content to the console
as it comes in. Save it as `tool.js`:
```js
const IlpPacket = require('ilp-packet')
const Plugin = require('ilp-plugin-xrp-escrow')
const crypto = require('crypto')
const fetch = require('node-fetch')
const uuid = require('uuid/v4')
function base64 (buf) { return buf.toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '') }
function hash (secret) { return crypto.createHash('sha256').update(secret).digest() }
function hmac (secret, input) { return crypto.createHmac('sha256', secret).update(input).digest() }

const plugin = new Plugin({
  secret: 'sndb5JDdyWiHZia9zv44zSr2itRy1',
  account: 'rGtqDAJNTDMLaNNfq1RVYgPT8onFMj19Aj',
  server: 'wss://s.altnet.rippletest.net:51233',
  prefix: 'test.crypto.xrp.'
})

let counter = 0

function sendTransfer (obj) {
  obj.id = uuid()
  obj.from = plugin.getAccount()
  // to
  obj.ledger = plugin.getInfo().prefix
  // amount
  // executionCondition
  obj.expiresAt = new Date(new Date().getTime() + 1000000).toISOString()
  return plugin.sendTransfer(obj).then(function () {
    return obj.id
  })
}

plugin.connect().then(function () {
  return fetch('http://localhost:8000/')
}).then(function (inRes) {
  inRes.body.pipe(process.stdout)
  const payHeaderParts = inRes.headers.get('Pay').split(' ')
  console.log(payHeaderParts)
  // e.g. Pay: 1 test.crypto.xrp.asdfaqefq3f.26wrgevaew SkTcFTZCBKgP6A6QOUVcwWCCgYIP4rJPHlIzreavHdU
  setInterval(function () {
    const ilpPacketContents = {
      account: payHeaderParts[1] + '.' + (++counter),
      amount: '1',
      data: ''
    }
    const fulfillment = hmac(Buffer.from(payHeaderParts[2], 'base64'), ilpPacketContents.account)
    const condition = hash(fulfillment)
    sendTransfer({
      to: payHeaderParts[1],
      amount: '1',
      executionCondition: base64(condition),
      ilp: base64(IlpPacket.serializeIlpPayment(ilpPacketContents))
    }).then(function () {
      // console.log('transfer sent')
    }, function (err) {
      console.error(err.message)
    })
  }, 500)
})
```

As you can see, it parses the `Pay` header, and then sends one XRP per 500ms. Try it out!

```sh
$ node ./tool.js
```

After some startup time, you should see letters being printed.

## Using a trustline

Getting one letter per 500ms is not very fast. It would be nice if we could stream the money faster, so that
the content arrives faster! For this, we can add a ledger to the shop. The tool opens an account on this ledger,
and then pays for letters from its account at the shop's ledger, which will be much faster
than paying via the XRP ledger.

The shop's ledger will expose the Bilateral Transfer Protocol (BTP), which is an optimization of the Ledger Plugin Interface (LPI)
that we already saw in the Letter Shop tutorial. It is symmetrical over a WebSocket, so the client can send the same messages as the
server, and this will have the same effect, but in the opposite direction. For instance, both parties can send a transfer to the
other party in the form of a 'PREPARE' packet, and both parties can also fulfill transfers from the other party by sending a
'FULFILL' packet. These BTP packets are similar to the objects passed to `plugin.sendTransfer` or `plugin.fulfillCondition`,
although they are a bit more concise, and before they go onto the WebSocket, they are serialized into OER buffers. To learn
more about the BTP protocol, read [the BTP spec](https://github.com/interledger/rfcs/pull/300).

Thanks to the plugin architeture, we have to change surprisingly little to switch from XRP to BTP: we just include the
`'ilp-plugin-payment-channel-framework'` plugin instead of the `'ilp-plugin-xrp-escrow'` one, and give it the config options
it needs. You can see that here in the `shop3.js` script, which includes the BTP-enabled ledger; run `diff shop2.js shop3.js`
to see how similar they really are:

```js
const IlpPacket = require('ilp-packet')
const http = require('http')
const crypto = require('crypto')
const Plugin = require('ilp-plugin-payment-channel-framework')
function base64 (buf) { return buf.toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '') }
function hash (secret) { return crypto.createHash('sha256').update(secret).digest() }
function hmac (secret, input) { return crypto.createHmac('sha256', secret).update(input).digest() }

let users = {}
const store = {}
const plugin = new Plugin({
  listener: {
    port: 9000
  },
  incomingSecret: '',
  maxBalance: '1000000000',
  prefix: 'g.letter-shop.mytrustline.',
  info: {
    currencyScale: 9,
    currencyCode: 'XRP',
    prefix: 'g.letter-shop.mytrustline.',
    connectors: []
  },
  _store: {
    get: (k) => store[k],
    put: (k, v) => { store[k] = v },
    del: (k) => delete store[k]
  }
})

plugin.connect().then(function () {
  plugin.on('incoming_prepare', function (transfer) {
    const ilpPacketContents = IlpPacket.deserializeIlpPayment(Buffer.from(transfer.ilp, 'base64'))
    const parts = ilpPacketContents.account.split('.')
    // 0: test, 1: crypto, 2: xrp, 3: rrhnXcox5bEmZfJCHzPxajUtwdt772zrCW, 4: userId, 5: paymentId
    if (parts.length < 6 || typeof users[parts[4]] === 'undefined' || ilpPacketContents.amount !== transfer.amount) {
      plugin.rejectIncomingTransfer(transfer.id, {}).catch(function () {})
    } else {
      const { secret, res } = users[parts[4]]
      const fulfillment = hmac(secret, ilpPacketContents.account)
      const condition = hash(fulfillment)
      if (transfer.executionCondition === base64(condition)) {
        plugin.fulfillCondition(transfer.id, base64(fulfillment)).then(function () {
          const letter = ('ABCDEFGHIJKLMNOPQRSTUVWXYZ').split('')[(Math.floor(Math.random() * 26))]
          res.write(letter)
        }, function (err) {
          console.error(err.message)
        })
      } else {
        console.log('no match!', { secret, fulfillment, condition, transfer })
      }
    }
  })

  http.createServer(function (req, res) {
    const secret = crypto.randomBytes(32)
    const user = base64(crypto.randomBytes(8))
    users[user] = { secret, res }
    console.log('user! writing head', user)
    res.writeHead(200, {
      'Pay': [ 1, plugin.getAccount() + '.' + user, base64(secret) ].join(' ')
    })
    // Flush the headers in a first TCP packet:
    res.socket.write(res._header)
    res._headerSent = true
  }).listen(8000)
})
```

To run it, use:

```sh
npm install interledgerjs/ilp-plugin-payment-channel-framework#mj-fix
node ./shop3.js
```

And here is the content consumption tool, which first transfers 1000 XRP to its account at the shop's ledger, and then
consumes 1000 letters. Save it as `tool2.js`:
```js
const IlpPacket = require('ilp-packet')
const Plugin = require('ilp-plugin-payment-channel-framework')
const crypto = require('crypto')
const fetch = require('node-fetch')
const uuid = require('uuid/v4')
function base64 (buf) { return buf.toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '') }
function hash (secret) { return crypto.createHash('sha256').update(secret).digest() }
function hmac (secret, input) { return crypto.createHmac('sha256', secret).update(input).digest() }

const plugin = new Plugin({
  server: 'btp+ws://:@localhost:9000/'
})

let counter = 0

function sendTransfer (obj) {
  obj.id = uuid()
  obj.from = plugin.getAccount()
  // to
  obj.ledger = plugin.getInfo().prefix
  // amount
  // executionCondition
  obj.expiresAt = new Date(new Date().getTime() + 1000000).toISOString()
  // console.log('calling sendTransfer!',  obj)
  return plugin.sendTransfer(obj).then(function () {
    return obj.id
  })
}

plugin.connect().then(function () {
  console.log('plugin connected')
  return fetch('http://localhost:8000/')
}).then(function (inRes) {
  inRes.body.pipe(process.stdout)
  const payHeaderParts = inRes.headers.get('Pay').split(' ')
  console.log(payHeaderParts)
  // e.g. Pay: 1 test.crypto.xrp.asdfaqefq3f.26wrgevaew SkTcFTZCBKgP6A6QOUVcwWCCgYIP4rJPHlIzreavHdU
  setInterval(function () {
    const ilpPacketContents = {
      account: payHeaderParts[1] + '.' + (++counter),
      amount: '1',
      data: ''
    }
    const fulfillment = hmac(Buffer.from(payHeaderParts[2], 'base64'), ilpPacketContents.account)
    const condition = hash(fulfillment)
    sendTransfer({
      to: payHeaderParts[1],
      amount: '1',
      executionCondition: base64(condition),
      ilp: base64(IlpPacket.serializeIlpPayment(ilpPacketContents))
    }).then(function () {
      // console.log('transfer sent')
    }, function (err) {
      console.error(err.message)
    })
  }, 1)
})
```

To run it, use:

```sh
node ./tool2.js
```

## What you learned

We used the ILP packet for the first time, talked about connectors, payments as chains of transfers, and used
the `Pay` header, so that the shop could request payment from the tool that connects to it.

We added a BTP-enabled ledger to the shop, so that our content consumption tool can receive letters faster.

