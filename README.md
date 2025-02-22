import requests
from solana.rpc.api import Client
from solana.publickey import PublicKey
from solana.transaction import Transaction
from solana.system_program import TransferParams, transfer
from solana.keypair import Keypair
from solana.rpc.types import TxOpts

# Set up Solana client
solana_client = Client("https://api.mainnet-beta.solana.com")

# Load your wallet private key (keep this secure!)
wallet = Keypair.from_secret_key(bytes([YOUR_PRIVATE_KEY_HERE]))

# DexScreener API endpoint for Solana tokens
DEX_SCREENER_API = "https://api.dexscreener.com/latest/dex/tokens/"

# Function to fetch new tokens on Solana
def fetch_new_tokens():
    response = requests.get(DEX_SCREENER_API)
    if response.status_code == 200:
        tokens = response.json()["pairs"]
        return tokens
    else:
        print("Failed to fetch tokens from DexScreener.")
        return []

# Function to filter tokens based on criteria
def filter_tokens(tokens):
    filtered_tokens = []
    for token in tokens:
        market_cap = token.get("marketCap", 0)
        liquidity = token.get("liquidity", 0)
        volume = token.get("volume", 0)

        # Example criteria: Market cap between $5K and $50K, liquidity > $1K, volume > $500
        if 5000 <= market_cap <= 50000 and liquidity > 1000 and volume > 500:
            filtered_tokens.append(token)
    return filtered_tokens

# Function to execute a trade
def execute_trade(token_mint, amount_in_sol):
    transaction = Transaction().add(
        transfer(TransferParams(
            from_pubkey=wallet.public_key,
            to_pubkey=PublicKey(token_mint),
            lamports=int(amount_in_sol * 1e9)  # Convert SOL to lamports
        )
    )

    # Send the transaction
    response = solana_client.send_transaction(transaction, wallet, opts=TxOpts(skip_preflight=True))
    print(f"Transaction sent: {response}")

# Main loop
if __name__ == "__main__":
    print("Monitoring new tokens...")
    tokens = fetch_new_tokens()
    filtered_tokens = filter_tokens(tokens)

    if filtered_tokens:
        print(f"Found {len(filtered_tokens)} promising tokens.")
        for token in filtered_tokens:
            print(f"Snipping token: {token['baseToken']['name']} (Market Cap: ${token['marketCap']})")
            execute_trade(token['baseToken']['address'], 0.1)  # Swap 0.1 SOL for the token
    else:
        print("No promising tokens found.")
