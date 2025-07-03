<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

## How to Get Free Access to Algorand APIs via Nodely

You do not need to sign up or use an API key to access Algorand APIs through Nodely. The process is straightforward:

**1. Choose the Appropriate Endpoint**

Nodely provides public endpoints for different Algorand networks (MainNet, TestNet, BetaNet, FNet). Here are some examples:


| Network | Indexer API Endpoint | Algod API Endpoint |
| :-- | :-- | :-- |
| MainNet | https://mainnet-idx.4160.nodely.dev | https://mainnet-api.4160.nodely.dev |
| TestNet | https://testnet-idx.4160.nodely.dev | https://testnet-api.4160.nodely.dev |
| BetaNet | https://betanet-idx.4160.nodely.dev | https://betanet-api.4160.nodely.dev |
| FNet | https://fnet-idx.4160.nodely.dev | https://fnet-api.4160.nodely.dev |

**2. Use the Endpoint in Your SDK or API Client**

- Simply copy the endpoint URL for the network you want.
- Paste it into your Algorand SDK or API client as the base URL.
- No authentication or API key is required.

**3. Example Usage**

If you are using an Algorand SDK (such as Python, JavaScript, etc.), set the API base URL to one of the provided endpoints. For example, in Python:

```python
from algosdk.v2client import algod

algod_address = "https://testnet-api.4160.nodely.dev"
algod_token = ""  # No token required
algod_client = algod.AlgodClient(algod_token, algod_address)
```

**4. Additional Features**

- Nodely endpoints use Cloudflare for load balancing and acceleration.
- The endpoints are suitable for both backend and frontend use.
- You can also access online API documentation and playgrounds by clicking on the provided URLs[^1].

**Summary:**
You get free access by directly using the public endpoint URLs in your SDK or API client. No signup, API key, or special configuration is needed[^1].

<div style="text-align: center">⁂</div>

[^1]: https://nodely.io/docs/free/start

