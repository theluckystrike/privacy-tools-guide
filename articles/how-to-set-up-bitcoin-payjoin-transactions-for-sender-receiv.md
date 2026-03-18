---
layout: default
title: "How to Set Up Bitcoin PayJoin Transactions for Sender."
description: "A practical guide for developers and power users to implement Bitcoin PayJoin, a privacy-enhancing technique that breaks common transaction heuristics."
date: 2026-03-16
author: theluckystrike
permalink: /how-to-set-up-bitcoin-payjoin-transactions-for-sender-receiver/
categories: [guides]
tags: [tools]
reviewed: true
score: 8
intent-checked: true
---

{% raw %}

Bitcoin PayJoin (BIP 78) lets the receiver contribute inputs to a transaction, making it impossible for blockchain observers to identify who sent the funds. Unlike normal payments where the sender's inputs are obvious, PayJoin creates ambiguity that defeats standard on-chain analysis heuristics. This guide provides step-by-step technical implementation for both senders and receivers, including Python and Flask code examples, testing strategies, and production deployment checklists.

## Understanding PayJoin Fundamentals

In a standard Bitcoin transaction, blockchain observers can confidently identify which party is sending and receiving based on input/output patterns. PayJoin disrupts this by having the receiver contribute inputs to the transaction, making the flow of funds ambiguous.

A standard payment looks like this:

```
Sender Wallet: 1.5 BTC → Receiver: 1.0 BTC
                 └─→ Change: 0.49 BTC
```

With PayJoin, the transaction becomes indistinguishable from a coinjoin:

```
Sender Wallet: 1.5 BTC ─┐
Receiver Wallet: 0.5 BTC ─┼→ Output: 1.5 BTC
                          └─→ Change: 0.49 BTC
```

From the blockchain perspective, you cannot determine whether this is a payment, a consolidation, or a payment to self.

## Prerequisites

Before implementing PayJoin, ensure you have:

- A Bitcoin full node (Bitcoin Core recommended)
- A wallet with multiple UTXOs
- Python 3.9+ or a Bitcoin library like `bitcoinlib` or `btcpy`
- Tor for receiver endpoints (recommended for privacy)

## Implementing the Sender (Payee-Initiated)

The sender initiates the PayJoin by requesting a PayJoin URL from the receiver. Here's a Python implementation using the `btcpy` library:

```python
import urllib.parse
from btcpy.structs.address import Address
from btcpy.structs.transaction import TxBuilder, TxExecutor
from btcpy.structs.script import P2wpkhScript

class PayjoinSender:
    def __init__(self, node_url="127.0.0.1:8332", wallet_name="payjoin_sender"):
        self.node_url = node_url
        self.wallet_name = wallet_name
    
    def create_payjoin_proposal(self, receiver_url: str, send_amount: int):
        """
        Create a PayJoin proposal for the receiver.
        send_amount: in satoshis
        """
        # Parse receiver's PayJoin endpoint
        parsed = urllib.parse.urlparse(receiver_url)
        
        # Get UTXOs from our wallet
        utxos = self.getWalletUtxos()
        
        # Calculate maximum we can contribute
        total_available = sum(u['value'] for u in utxos)
        change_amount = total_available - send_amount
        
        # Create the unsigned transaction
        tx = TxBuilder().add_inputs(utxos).add_output(
            address=parsed.hostname,  # Receiver's address
            amount=send_amount
        ).add_output(
            address=self.get_change_address(),
            amount=change_amount
        ).build()
        
        # Create the PayJoin URL with our PSBT
        psbt = self.tx_to_psbt(tx)
        
        return self.create_payjoin_url(receiver_url, psbt)
    
    def getWalletUtxos(self):
        # Call Bitcoin Core RPC
        # Implementation depends on your setup
        pass
    
    def tx_to_psbt(self, tx):
        # Convert transaction to Partially Signed Transaction
        pass
```

## Implementing the Receiver (Payee)

The receiver must set up an endpoint to receive PayJoin requests. Here's a Flask-based implementation:

```python
from flask import Flask, request, jsonify
from btcpy.structs.transaction import Psbt
from btcpy.structs.script import P2wpkhScript
import bitcoinrpc

app = Flask(__name__)

class PayjoinReceiver:
    def __init__(self, rpc_connection):
        self.rpc = rpc_connection
        self.extended_private_key = "xprv..."  # Your xpub
    
    def handle_payjoin_request(self, psbt_base64: str):
        """
        Process incoming PayJoin proposal from sender.
        """
        # Decode the PSBT
        psbt = Psbt.from_base64(psbt_base64)
        
        # Verify the transaction is valid so far
        if not self.validate_proposal(psbt):
            return {"error": "Invalid proposal"}, 400
        
        # Add our inputs to the transaction
        psbt = self.add_receiver_inputs(psbt)
        
        # Set up our output (the payment amount we expect to receive)
        psbt = self.add_payment_output(psbt, expected_amount=100000)
        
        # Sign our inputs
        psbt = self.sign_psbt(psbt)
        
        # Return the finalized PSBT
        return {
            "psbt": psbt.to_base64(),
            "originalOutputs": psbt.tx.outputs
        }
    
    def add_receiver_inputs(self, psbt):
        """
        Select our UTXOs to add to the transaction.
        """
        # Get our available UTXOs
        our_utxos = self.get_receiver_utxos()
        
        # We want to contribute enough to make the amount ambiguous
        # but not so much that we create unnecessary change
        receiver_contribution = self.select_receiver_utxo(
            psbt.tx.outputs[0].amount  # Match sender's payment
        )
        
        for utxo in receiver_contribution:
            psbt.add_input(utxo)
        
        return psbt
    
    def validate_proposal(self, psbt):
        """
        Validate the sender's proposal:
        - Check fee rate is reasonable
        - Verify output amounts
        - Ensure no unexpected inputs
        """
        # Implementation details
        return True
```

## PayJoin URL Format

The PayJoin protocol uses a specific URL scheme defined in BIP 78:

```
https://receiver.example.com/pj?amount=0.01&label=Payment%20for%20Invoice%20123
```

Parameters include:
- `amount`: Suggested payment amount in BTC
- `label`: Optional label for the payment
- `exp`: Expiration time (Unix timestamp)
- `expire`: Expiration duration in seconds
- `signature`: Signature proving the receiver controls the endpoint

## Privacy Considerations

When implementing PayJoin, consider these privacy-critical aspects:

1. **UTXO Selection**: Both parties should avoid creating obvious change outputs. The receiver's contribution should roughly match the payment amount to minimize detectable change.

2. **Fee Management**: PayJoin transactions are larger than standard payments, requiring higher fees. Budget for approximately 2-3x the normal transaction fee.

3. **Timing**: Avoid initiating or completing PayJoin transactions at predictable intervals. Random delays between request and completion improve unlinkability.

4. **Network Isolation**: Run your PayJoin receiver over Tor to prevent IP address correlation between the sender and receiver.

5. **Replay Protection**: Ensure each PayJoin session uses fresh UTXOs to prevent transaction replay.

## Testing Your Implementation

Test PayJoin on Bitcoin's testnet before production use:

```bash
# Start a testnet node
bitcoind -testnet -server -rpcuser=testuser -rpcpassword=testpass

# Create test wallets
./bitcoin-cli -testnet createwallet "payjoin_sender"
./bitcoin-cli -testnet createwallet "payjoin_receiver"

# Get testnet coins from a faucet
# https://bitcoinfaucet.net/
```

Verify your implementation using the PayJoin Dev Toolkit:

```python
from payjoin import Sender, Receiver, Environment

# Test that your implementation follows BIP 78
def test_payjoin_compliance():
    sender = Sender(Environment.TESTNET)
    receiver = Receiver(Environment.TESTNET)
    
    # Run the compliance test suite
    result = sender.test_receiver_compatibility(receiver)
    assert result.is_valid(), f"Compliance errors: {result.errors}"
```

## Common Implementation Mistakes

Avoid these frequent errors when implementing PayJoin:

- **Fixed Contribution Amounts**: Always vary the receiver's contribution amount to prevent amount-based clustering.

- **Missing Expiration**: Always implement expiration to prevent indefinite waiting and resource consumption.

- **Inadequate Fee Estimation**: PayJoin transactions require more vbytes. Use a fee estimator that accounts for additional inputs.

- **No Input Value Matching**: The receiver should attempt to match or exceed the sender's payment amount to maximize privacy.

- **Ignoring RBF**: Always enable Replace-By-Fee for PayJoin transactions to handle fee market volatility.

## Production Deployment Checklist

Before deploying PayJoin in production:

- [ ] Implement Tor hidden service for receiver endpoint
- [ ] Set up monitoring for failed PayJoin attempts
- [ ] Configure reasonable expiration times (5-15 minutes recommended)
- [ ] Implement proper error handling and logging
- [ ] Test with multiple wallet implementations
- [ ] Verify chain analysis resistance with tools like OXT or Samurai Wallet
- [ ] Document API endpoints for integration
- [ ] Set up key rotation procedures

PayJoin represents one of the most effective practical improvements in Bitcoin transaction privacy. By carefully implementing both sender and receiver components, you can significantly reduce on-chain analysis effectiveness while maintaining full Bitcoin security guarantees.


## Related Reading

- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)
- [Privacy Tools Guides Hub](/privacy-tools-guide/guides-hub/)

Built by theluckystrike — More at [zovo.one](https://zovo.one)

{% endraw %}
