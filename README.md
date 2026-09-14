import requests
from decimal import Decimal

RPC_URL = "https://eth.llamarpc.com"
COINGECKO_URL = "https://api.coingecko.com/api/v3"

WALLET = "0x0000000000000000000000000000000000000000"

TOKENS = {
    "": {
        "coingecko_id": "ethereum",
        "address": None,
        "decimals": 18,
    },
    "USDC": {
        "coingecko_id": "usd-coin",
        "address": "0xA0b86991c6218b36c1d19d4a2e9eb0ce3606eb48",
        "decimals": 6,
    },
    "DAI": {
        "coingecko_id": "dai",
        "address": "0x6B175474E89094C44Da98b954EedeAC495271d0F",
        "decimals": 18,
    },
}


def rpc_call(method, params):
    payload = {
        "jsonrpc": "2.0",
        "method": method,
        "params": params,
        "id": 1,
    }

    response = requests.post(
        RPC_URL,
        json=payload,
        timeout=15
    )

    response.raise_for_status()
    data = response.json()

    if "error" in data:
        raise RuntimeError(data["error"])

    return data["result"]


def get_eth_balance():
    result = rpc_call(
        "eth_getBalance",
        [WALLET, "latest"]
    )

    return Decimal(int(result, 16)) / Decimal(10**18)


def get_token_balance(address, decimals):
    data = "0x70a08231" + (
        WALLET.lower().replace("0x", "").rjust(64, "0")
    )

    result = rpc_call(
        "eth_call",
        [{"to": address, "data": data}, "latest"]
    )

    return Decimal(int(result, 16)) / Decimal(10**decimals)


def get_prices(token_ids):
    params = {
        "ids": ",".join(token_ids),
        "vs_currencies": "usd"
    }

    response = requests.get(
        f"{COINGECKO_URL}/simple/price",
        params=params,
        timeout=15
    )

    response.raise_for_status()
    return response.json()


def main():
    balances = {}

    for symbol, token in TOKENS.items():
        if token["address"] is None:
            balance = get_eth_balance()
        else:
            balance = get_token_balance(
                token["address"],
                token["decimals"]
            )

        balances[symbol] = balance

    prices = get_prices([
        token["coingecko_id"]
        for token in TOKENS.values()
    ])

    total_value = Decimal("0")

    print("Web3 Portfolio Tracker")
    print("-" * 45)
    print(f"Wallet: {WALLET}\n")

    for symbol, token in TOKENS.items():
        balance = balances[symbol]
        price = Decimal(str(
            prices[token["coingecko_id"]]["usd"]
        ))

        value = balance * price
        total_value += value

        print(
            f"{symbol:<6} "
            f"{balance:,.6f} "
            f"= ${value:,.2f}"
        )

    print("-" * 45)
    print(f"Total Portfolio: ${total_value:,.2f}")


if __name__ == "__main__":
    main()
