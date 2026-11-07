import requests
import time
import random
import os
from datetime import datetime

# ================== SETTINGS ==================
STARTING_EQUITY = 100
POSITION_SIZE_PCT = 0.5
SLEEP_SECONDS = 7

# Telegram
TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")

def send_telegram(message):
    if not TELEGRAM_TOKEN or not TELEGRAM_CHAT_ID:
        return
    try:
        url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
        data = {
            "chat_id": TELEGRAM_CHAT_ID,
            "text": message,
            "parse_mode": "HTML"
        }
        requests.post(url, data=data, timeout=10)
    except:
        pass

class State:
    def __init__(self):
        self.equity = STARTING_EQUITY
        self.start_equity = STARTING_EQUITY
        self.wins = 0
        self.losses = 0
        self.open_trade = None
        self.line = 140
        self.cycle = 0

    def log(self, agent, message):
        now = datetime.now().strftime("%H:%M:%S")
        print(f"{now}  {agent:<11} {message}", flush=True)
        self.line += 1

state = State()

class ScannerAgent:
    def run(self):
        state.log("SCANNER", "Scanning new pairs + volume spikes...")
        try:
            r = requests.get("https://api.dexscreener.com/token-boosts/top/v1", timeout=10)
            boosts = r.json()

            addresses = [item["tokenAddress"] for item in boosts if item.get("chainId") == "solana" and item.get("tokenAddress")]
            if not addresses:
                state.log("SCANNER", "No tokens found")
                return []

            addr_str = ",".join(addresses[:8])
            r2 = requests.get(f"https://api.dexscreener.com/tokens/v1/solana/{addr_str}", timeout=10)
            pairs = r2.json()

            tokens = []
            for p in pairs:
                try:
                    symbol = p["baseToken"]["symbol"]
                    price = float(p.get("priceUsd") or 0)
                    liq = float(p.get("liquidity", {}).get("usd") or 0)
                    vol = float(p.get("volume", {}).get("h24") or 0)
                    
                    # Get real price change data
                    price_change = p.get("priceChange", {})
                    change_h1 = float(price_change.get("h1") or 0)
                    change_h24 = float(price_change.get("h24") or 0)

                    if price > 0 and liq > 10000:
                        tokens.append({
                            "symbol": "$" + symbol[:10],
                            "price": price,
                            "liquidity": liq,
                            "volume": vol,
                            "change_h1": change_h1,
                            "change_h24": change_h24
                        })
                except:
                    continue

            state.log("SCANNER", f"Found {len(tokens)} tokens")
            return tokens
        except Exception as e:
            state.log("SCANNER", f"Error: {str(e)[:60]}")
            return []

class RugCheckAgent:
    def run(self, token):
        symbol = token["symbol"]
        liquidity = token["liquidity"]
        volume = token["volume"]

        if liquidity < 25000:
            state.log("RUG CHECK", f"FILTERED {symbol} → low liquidity")
            return False
        if volume < 8000:
            state.log("RUG CHECK", f"FILTERED {symbol} → low volume")
            return False
        if volume < liquidity * 0.12:
            state.log("RUG CHECK", f"FILTERED {symbol} → weak volume")
            return False

        state.log("RUG CHECK", f"PASS {symbol}")
        return True

class SizerAgent:
    def run(self):
        size = state.equity * POSITION_SIZE_PCT
        state.log("SIZER", f"{POSITION_SIZE_PCT*100:.0f}% = ${size:,.2f}")
        return size

class ExecutorAgent:
    def enter(self, token, size_usd):
        if state.open_trade:
            return
        state.open_trade = {
            "symbol": token["symbol"],
            "entry": token["price"],
            "size": size_usd,
            "change_h1": token.get("change_h1", 0),
            "change_h24": token.get("change_h24", 0)
        }
        state.log("EXECUTOR", f"ENTER {token['symbol']} LONG")
        state.log("LOGGER", f"line #{state.line} enter {token['symbol']}")
        
        send_telegram(f"<b>ENTER</b> {token['symbol']}\nSize: ${size_usd:,.2f}\n1h Change: {token.get('change_h1', 0):+.1f}%")

    def try_exit(self):
        if not state.open_trade:
            return

        # More realistic exit based on real data + some randomness
        if random.random() < 0.42:
            real_momentum = state.open_trade.get("change_h1", 0)
            
            # Base the simulated result more on real momentum
            if real_momentum > 8:
                change = random.uniform(0.08, 0.45)
            elif real_momentum > 2:
                change = random.uniform(-0.05, 0.28)
            elif real_momentum < -10:
                change = random.uniform(-0.35, -0.05)
            else:
                change = random.uniform(-0.22, 0.32)

            pnl = state.open_trade["size"] * change
            state.equity += pnl
            symbol = state.open_trade["symbol"]

            if pnl >= 0:
                state.wins += 1
                result = f"+${pnl:,.2f}"
            else:
                state.losses += 1
                result = f"-${abs(pnl):,.2f}"

            state.log("EXECUTOR", f"EXIT {symbol} {result}")
            state.log("EQUITY", f"Balance → ${state.equity:,.2f}")
            
            send_telegram(f"<b>EXIT</b> {symbol}\nResult: {result}\nBalance: ${state.equity:,.2f}")
            
            state.open_trade = None

class LoggerAgent:
    def heartbeat(self):
        state.log("LOGGER", f"line #{state.line} system alive")

class EquityAgent:
    def report(self):
        total = state.wins + state.losses
        wr = (state.wins / total * 100) if total > 0 else 0
        state.log("EQUITY", f"${state.equity:,.2f} | {state.wins}W {state.losses}L ({wr:.0f}%)")

# ================== MAIN ==================
scanner = ScannerAgent()
rug = RugCheckAgent()
sizer = SizerAgent()
executor = ExecutorAgent()
logger = LoggerAgent()
equity = EquityAgent()

print("="*60, flush=True)
print("  GROK BOT TERMINAL - IMPROVED SIMULATION", flush=True)
print(f"  Starting Balance: ${state.equity:,.2f}", flush=True)
print("="*60, flush=True)

state.log("SYSTEM", "Improved simulation started (still Paper Trading)")
send_telegram("<b>Grok Bot Updated</b>\nNow using improved DexScreener simulation\nStill Paper Trading only")

while True:
    state.cycle += 1
    print(f"\n------ Cycle {state.cycle} ------", flush=True)

    tokens = scanner.run()

    for token in tokens[:2]:
        if rug.run(token):
            size = sizer.run()
            executor.enter(token, size)

    executor.try_exit()
    logger.heartbeat()
    equity.report()

    time.sleep(SLEEP_SECONDS)
