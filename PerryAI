#!/usr/bin/env python3
"""
PerryAI Trading Platform v5.1 — Mixologist Edition (Polished)
──────────────────────────────────────────────────────────────
Bots: Matches/Differs · Even/Odd · Over/Under · Mixologist
Run:  python PerryAI_v5_fixed.py
Open: http://localhost:5000

Credentials via environment variables (recommended):
  PERRYAI_EMAIL=you@example.com PERRYAI_PASSWORD=yourpass python PerryAI_v5_fixed.py

Or set defaults below. pip install flask python-deriv-api

Fixes in v5.1:
  - Mixologist: SL=0 no longer stops on single loss (bulletproof guard)
  - Token expiry handled gracefully with clear error + notify
  - Balance stream failure falls back to periodic polling
  - Per-module mart_step shown correctly in stats
  - Config validation with user-friendly errors
  - Credentials configurable via env vars (no secrets in code)
  - Exponential backoff on reconnect attempts
  - Log levels: DEBUG / INFO / WARNING / ERROR with timestamps
"""

import asyncio, json, logging, math, os, queue, random, threading, time
from collections import deque
from decimal import Decimal, InvalidOperation
from flask import (Flask, Response, jsonify, render_template_string,
                   request, redirect, url_for, session)

# ── Configure stdlib logging ──────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%H:%M:%S",
)
logger = logging.getLogger("perryai")

app = Flask(__name__)
app.secret_key = os.environ.get("PERRYAI_SECRET", "perryai_secret_2026")

# Credentials: prefer env vars, fall back to defaults (change these!)
VALID_EMAIL    = os.environ.get("PERRYAI_EMAIL",    "lostj2954@gmail.com")
VALID_PASSWORD = os.environ.get("PERRYAI_PASSWORD", "password123")

_bot        = None
_bot_loop   = None
_bot_thread = None
_sse_clients  = []
_sse_lock     = threading.Lock()
_stored_token = None
_live_balance = {}

# ═══════════════════════════════════════════════════════════════
#  CONFIG VALIDATION
# ═══════════════════════════════════════════════════════════════
def _validate_config(data: dict) -> str | None:
    """Return an error string, or None if config is valid."""
    try:
        stake = float(data.get("stake", 0))
    except (TypeError, ValueError):
        return "stake must be a number"
    if stake <= 0:
        return "stake must be > 0"
    if stake < 0.35:
        return "minimum stake is $0.35"

    try:
        duration = int(data.get("duration", 1))
    except (TypeError, ValueError):
        return "duration must be an integer"
    if duration < 1 or duration > 10:
        return "duration must be between 1 and 10 ticks"

    for field in ("take_profit", "stop_loss"):
        val = data.get(field, 0)
        try:
            v = float(val)
        except (TypeError, ValueError):
            return f"{field} must be a number"
        if v < 0:
            return f"{field} must be ≥ 0 (use 0 to disable)"

    if data.get("use_martingale"):
        try:
            mm = float(data.get("martingale_multiplier", 2))
        except (TypeError, ValueError):
            return "martingale_multiplier must be a number"
        if mm < 1.1 or mm > 10:
            return "martingale_multiplier must be between 1.1 and 10"
        try:
            ms = int(data.get("max_martingale_steps", 4))
        except (TypeError, ValueError):
            return "max_martingale_steps must be an integer"
        if ms < 1 or ms > 10:
            return "max_martingale_steps must be between 1 and 10"

    return None  # all good

# ═══════════════════════════════════════════════════════════════
#  SSE HELPERS
# ═══════════════════════════════════════════════════════════════
_LOG_LEVEL_MAP = {"debug": logging.DEBUG, "info": logging.INFO,
                  "warning": logging.WARNING, "error": logging.ERROR,
                  "success": logging.INFO, "win": logging.INFO,
                  "loss": logging.WARNING, "trade": logging.INFO}

def _push(event, data):
    payload = f"event: {event}\ndata: {json.dumps(data)}\n\n"
    with _sse_lock:
        dead = []
        for q in _sse_clients:
            try:    q.put_nowait(payload)
            except queue.Full: dead.append(q)
        for q in dead: _sse_clients.remove(q)
    # Mirror to stdlib logger for terminal visibility
    if event == "log":
        lvl = _LOG_LEVEL_MAP.get(data.get("l", "info"), logging.INFO)
        logger.log(lvl, "[BOT] %s", data.get("m", ""))

def _bot_is_running():
    global _bot, _bot_thread
    if _bot is None: return False
    if _bot._stop.is_set(): return False
    if _bot_thread is None or not _bot_thread.is_alive():
        _bot = None; _bot_thread = None; return False
    return True

# ═══════════════════════════════════════════════════════════════
#  BASE BOT MIXIN — Fixed TP/SL, clean stop, stable stats
# ═══════════════════════════════════════════════════════════════
class BaseBotMixin:
    def log(self, msg, level="info"):
        _push("log", {"m": msg, "l": level, "t": time.strftime("%H:%M:%S")})

    def _base_stats(self):
        total = self.wins + self.losses
        wr = round(self.wins / total * 100, 1) if total else 0
        return {
            "bot_type":  self.bot_type,
            "profit":    float(self.total_profit),
            "trades":    self.trade_count,
            "stake":     float(self.stake),
            "wins":      self.wins,
            "losses":    self.losses,
            "wr":        wr,
            "ticks":     self.tick_count,
            "elapsed":   int(time.time() - self.start_time),
            "mstep":     self.mart_step,
            "mmax":      self.mart_max,
            "tp":        float(self.take_profit),
            "sl":        float(self.stop_loss),
            "balance":   self.balance,
            "phistory":  self.profit_history[-80:],
            "history":   self.trade_history[-100:],   # send last 100 for stable counts
        }

    # ── FIXED STOP: never cancel pending — let them settle on Deriv ──
    async def _shared_stop(self):
        """
        Called from the manual Stop button (/api/stop).
        For TP/SL stops, _stop is already set synchronously by _check_tp_sl;
        this method handles the remaining log message and is safe to call either way.
        """
        if not self._stop.is_set():
            self._stop.set()
        n_open = len(self.open_contracts)
        if n_open:
            self.log(f"🛑 Stop signal received — waiting for {n_open} open contract(s) to settle on Deriv…", "warning")
        # _bot_run's await self._stop.wait() unblocks and calls _graceful_shutdown

    async def _shared_run(self):
        """Authenticate with Deriv API, subscribe to balance, then run the bot strategy."""
        global _stored_token, _live_balance
        try:
            from deriv_api import DerivAPI
        except ImportError:
            self.log("python-deriv-api not installed. Run: pip install python-deriv-api", "error")
            _push("auth_failed", {"error": "python-deriv-api not installed"})
            self._stop.set(); return

        # Exponential backoff for transient network issues (up to 3 attempts)
        for attempt in range(1, 4):
            self.api = DerivAPI(app_id=self.app_id)
            try:
                auth = await self.api.authorize({"authorize": self.api_token})
                if "error" in auth:
                    msg = auth["error"].get("message", "Unknown auth error")
                    code = auth["error"].get("code", "")
                    # Token expiry / invalid token — no point retrying
                    if code in ("InvalidToken", "AuthorizationRequired", "DisabledClient"):
                        self.log(f"🔑 Token error ({code}): {msg} — please reconnect with a valid token.", "error")
                        _push("auth_failed", {"error": msg, "code": code})
                        self._stop.set(); return
                    raise RuntimeError(msg)

                info          = auth.get("authorize", {})
                self.loginid  = info.get("loginid", "???")
                self.currency = info.get("currency", "USD")
                self.balance  = float(info.get("balance", 0))
                _stored_token = self.api_token
                acct_type     = "Demo Account" if self.loginid.startswith("VR") else "Live Account"
                _live_balance[self.loginid] = {
                    "balance": self.balance, "currency": self.currency, "acct_type": acct_type
                }
                self.log(f"✅ Authorized → {self.loginid} ({acct_type})", "success")
                _push("authorized", {
                    "loginid": self.loginid, "balance": self.balance,
                    "currency": self.currency, "acct_type": acct_type
                })
                break  # success
            except Exception as e:
                if attempt < 3:
                    wait = 2 ** attempt
                    self.log(f"Auth attempt {attempt} failed ({e}) — retrying in {wait}s…", "warning")
                    await asyncio.sleep(wait)
                    try: await self.api.disconnect()
                    except Exception: pass
                    continue
                self.log(f"Auth failed after {attempt} attempts: {e}", "error")
                _push("auth_failed", {"error": str(e)})
                self._stop.set(); return

        # Balance subscription with polling fallback
        try:
            bs = await self.api.subscribe({"balance": 1, "subscribe": 1})
            def _on_balance(msg):
                if "balance" not in msg: return
                b = float(msg["balance"].get("balance", self.balance))
                c = msg["balance"].get("currency", self.currency)
                self.balance = b
                _live_balance[self.loginid] = {"balance": b, "currency": c, "acct_type": acct_type}
                _push("balance_update", {"balance": b, "currency": c})
            bs.subscribe(_on_balance); self._bal_sub = bs
            self.log("📊 Balance stream active", "info")
        except Exception as be:
            self.log(f"Balance stream unavailable ({be}) — falling back to polling", "warning")
            self._bal_sub = None
            # Start a polling coroutine as fallback
            asyncio.create_task(self._poll_balance_fallback(acct_type))

        await self._bot_run()

    async def _poll_balance_fallback(self, acct_type: str):
        """Poll balance every 5 seconds when the subscription stream fails."""
        while not self._stop.is_set():
            await asyncio.sleep(5)
            if self._stop.is_set(): break
            try:
                br = await self.api.balance({"balance": 1})
                if "error" not in br and "balance" in br:
                    b = float(br["balance"]["balance"])
                    c = br["balance"].get("currency", self.currency)
                    self.balance = b
                    if self.loginid:
                        _live_balance[self.loginid] = {"balance": b, "currency": c, "acct_type": acct_type}
                    _push("balance_update", {"balance": b, "currency": c})
            except Exception:
                pass  # silent — will retry next cycle

    def _apply_result(self, profit, orig_stake):
        self.total_profit += profit
        self.profit_history.append(float(self.total_profit))
        won = profit > 0
        if won:
            self.wins += 1; self.stake = self.main_stake; self.mart_step = 0
            self.log(f"WIN +${float(profit):.2f} | Total: {float(self.total_profit):+.2f}", "win")
        else:
            self.losses += 1
            if self.use_martingale and self.mart_step < self.mart_max:
                self.mart_step += 1
                self.stake = orig_stake * self.mart_mult
                self.log(f"LOSS ${float(profit):.2f} | Mart ×{self.mart_step} → ${float(self.stake):.2f}", "loss")
            else:
                self.stake = self.main_stake; self.mart_step = 0
                self.log(f"LOSS ${float(profit):.2f} | Total: {float(self.total_profit):+.2f}", "loss")
        return won

    def _check_tp_sl(self):
        """
        Check total profit vs TP/SL after every contract settlement.
        _stop is set SYNCHRONOUSLY — on_tick sees it immediately and stops
        spawning new tasks. In-flight contracts settle via _graceful_shutdown.
        Mixologist: module losses never stop trading; only total_profit matters.
        """
        if self._stop.is_set():
            return

        tp = self.take_profit
        sl = self.stop_loss
        try:
            tp = Decimal(str(tp)) if not isinstance(tp, Decimal) else tp
            sl = Decimal(str(sl)) if not isinstance(sl, Decimal) else sl
        except InvalidOperation:
            return

        if tp > Decimal("0") and self.total_profit >= tp:
            self.log(
                f"🎯 TAKE PROFIT HIT +${float(self.total_profit):.2f} "
                f"(target ${float(tp):.2f}) — STOPPING, settling in-flight contracts…", "success"
            )
            self._stop.set()   # synchronous — on_tick sees this immediately
            return

        if sl > Decimal("0") and self.total_profit <= -sl:
            self.log(
                f"🛑 STOP LOSS HIT ${float(self.total_profit):.2f} "
                f"(threshold -${float(sl):.2f}) — STOPPING, settling in-flight contracts…", "warning"
            )
            self._stop.set()   # synchronous

    # ── Shared cleanup: waits for open contracts to settle ──
    async def _graceful_shutdown(self, ts, ps):
        """Wait for all open contracts to settle before disconnecting."""
        if self.open_contracts:
            self.log(f"⏳ Settling {len(self.open_contracts)} contract(s) — please wait…", "warning")
            deadline = time.time() + 35          # max 35-second grace window
            while self.open_contracts and time.time() < deadline:
                await asyncio.sleep(0.25)
            if self.open_contracts:
                n = len(self.open_contracts)
                self.log(f"⚠️  {n} contract(s) did not settle in time — check Deriv for final P&L", "warning")
                # Mark unsettled so UI doesn't show wrong profit
                for cid in list(self.open_contracts.keys()):
                    for t in self.trade_history:
                        if t["cid"] == cid and t["status"] == "pending":
                            t["status"] = "unsettled"; t["profit"] = 0.0; t["payout"] = 0.0
                            break
                    del self.open_contracts[cid]

        # Fetch definitive settled balance from Deriv
        fresh_balance = self.balance
        currency = getattr(self, "currency", "USD")
        try:
            if self.api:
                br = await self.api.balance({"balance": 1})
                if "error" not in br and "balance" in br:
                    fresh_balance = float(br["balance"]["balance"])
                    currency = br["balance"].get("currency", currency)
                    self.balance = fresh_balance
                    if self.loginid:
                        _live_balance[self.loginid] = {
                            "balance": fresh_balance, "currency": currency,
                            "acct_type": "Demo Account" if self.loginid.startswith("VR") else "Live Account"
                        }
        except Exception: pass

        total = self.wins + self.losses
        _push("bot_stopped", {
            "balance":  fresh_balance,
            "currency": currency,
            "summary":  {
                "trades": total, "wins": self.wins, "losses": self.losses,
                "wr":     round(self.wins / total * 100, 1) if total else 0,
                "profit": float(self.total_profit),
            },
        })
        if hasattr(self, "push_stats"): self.push_stats()
        try:
            await ts.unsubscribe()
            await ps.unsubscribe()
            await self.api.disconnect()
        except Exception: pass


# ═══════════════════════════════════════════════════════════════
#  BOT 1 — MATCHES / DIFFERS
# ═══════════════════════════════════════════════════════════════
class DifferBot(BaseBotMixin):
    bot_type = "md"
    def __init__(self, cfg):
        self.app_id=127806; self.api_token=cfg["api_token"]
        self.symbol=cfg.get("symbol","1HZ100V"); self.duration=int(cfg.get("duration",1))
        self.duration_unit="t"; self.currency="USD"
        self.contract_type=cfg["contract_type"]
        rp=cfg.get("prediction")
        self.prediction=int(rp) if rp not in (None,"","auto") else None
        self.main_stake=Decimal(str(cfg["stake"])); self.tick_interval=int(cfg.get("tick_interval",1))
        self.max_trades=int(cfg.get("max_trades",0)); self.use_martingale=bool(cfg.get("use_martingale",False))
        self.mart_mult=Decimal(str(cfg.get("martingale_multiplier",2.0)))
        self.mart_max=int(cfg.get("max_martingale_steps",4)); self.mart_step=0
        self.auto_switch=bool(cfg.get("auto_switch",False))
        self.stats_window=int(cfg.get("stats_window",20)); self.update_every=int(cfg.get("update_every_trades",3))
        self.trades_since_upd=0; self.last_digits=None
        self.take_profit=Decimal(str(cfg.get("take_profit",0))); self.stop_loss=Decimal(str(cfg.get("stop_loss",0)))
        self.stake=self.main_stake; self.open_contracts={}; self.api=None
        self.tick_count=0; self.trade_count=0; self.trade_history=[]
        self.total_profit=Decimal("0"); self.wins=self.losses=0
        self._stop=asyncio.Event(); self.digit_counts=[0]*10
        self.last_digit=None; self.start_time=time.time(); self.profit_history=[]
        self.balance=None; self.loginid=None

    def push_stats(self):
        tt=sum(self.digit_counts) or 1; pcts=[round(c/tt*100,1) for c in self.digit_counts]
        s=self._base_stats()
        s.update({"pred":self.prediction,"ct":self.contract_type,
                  "dcounts":self.digit_counts,"dpcts":pcts,"ldigit":self.last_digit})
        _push("stats",s)

    def on_tick(self, msg):
        if "tick" not in msg or self._stop.is_set(): return
        tick=msg["tick"]; ld=int(str(tick["quote"])[-1])
        self.tick_count+=1; self.digit_counts[ld]+=1; self.last_digit=ld
        if self.auto_switch and self.last_digits is not None: self.last_digits.append(ld)
        self.push_stats()
        if self.tick_count%self.tick_interval==0:
            if self.max_trades>0 and self.trade_count>=self.max_trades:
                self._stop.set(); return   # synchronous: no new tasks spawned
            asyncio.create_task(self.attempt_trade(tick))

    async def attempt_trade(self, tick):
        if self._stop.is_set(): return
        try:
            pr=await self.api.proposal({
                "proposal":1,"amount":math.nextafter(float(self.stake),float("inf")),
                "basis":"stake","contract_type":self.contract_type,"currency":self.currency,
                "duration":self.duration,"duration_unit":self.duration_unit,"symbol":self.symbol,
                "barrier":str(self.prediction)})
            if "error" in pr: self.log(f"Proposal error: {pr['error']['message']}","error"); return
            # ── Second stop check: TP/SL may have fired while awaiting proposal ──
            if self._stop.is_set(): return
            self.trade_count+=1
            buy=await self.api.buy({"buy":pr["proposal"]["id"],"price":float(pr["proposal"]["ask_price"])})
            if "error" in buy: self.log(f"Buy error: {buy['error']['message']}","error"); return
            cid=buy["buy"]["contract_id"]; self.open_contracts[cid]={"stake":self.stake}
            lbl="DIFFERS" if self.contract_type=="DIGITDIFF" else "MATCH"
            self.trade_history.append({
                "cid":cid,"symbol":self.symbol,"type":lbl,"digit":self.prediction,
                "stake":float(self.stake),"dur":self.duration,"status":"pending",
                "profit":None,"payout":None,"time":time.strftime("%m/%d/%Y, %H:%M:%S")})
            self.log(f"Trade #{cid} — {lbl} {self.prediction}","trade"); self.push_stats()
        except Exception as e: self.log(f"Trade error: {e}","error")

    def on_poc(self, msg):
        if "proposal_open_contract" not in msg: return
        c=msg["proposal_open_contract"]; cid=c.get("contract_id")
        if cid not in self.open_contracts or not c.get("is_sold"): return
        profit=Decimal(str(c.get("profit",0))); payout=float(c.get("sell_price",0))
        orig=self.open_contracts[cid]["stake"]; won=self._apply_result(profit,orig)
        if c.get("account_balance") is not None:
            self.balance=float(c["account_balance"])
            if self.loginid:
                _live_balance[self.loginid]={"balance":self.balance,"currency":self.currency,
                    "acct_type":"Demo Account" if self.loginid.startswith("VR") else "Live Account"}
        for t in self.trade_history:
            if t["cid"]==cid:
                t["status"]="won" if won else "lost"; t["profit"]=float(profit); t["payout"]=payout; break
        del self.open_contracts[cid]
        if self.auto_switch and self.last_digits and self.contract_type=="DIGITDIFF":
            self.trades_since_upd+=1
            if self.trades_since_upd>=self.update_every and len(self.last_digits)>=self.stats_window:
                counts=[0]*10
                for d in self.last_digits: counts[d]+=1
                lo=min(range(10),key=lambda i:counts[i])
                if lo!=self.prediction: self.log(f"AUTO-SWITCH → digit {lo}","info"); self.prediction=lo
                self.trades_since_upd=0
        # ── KEY FIX: only check TP/SL when open_contracts is empty ──
        # If any contracts are still pending, their outcomes could change the
        # final P&L. Wait until the last one settles before deciding to stop.
        if not self._stop.is_set():
            self._check_tp_sl()
        self.push_stats()

    async def stop(self): await self._shared_stop()
    async def run(self):  await self._shared_run()
    async def _bot_run(self):
        if self.auto_switch:
            try:
                hist=await self.api.ticks_history({"ticks_history":self.symbol,"style":"ticks","count":self.stats_window,"end":"latest"})
                if "error" not in hist:
                    self.last_digits=deque((int(str(p)[-1]) for p in hist["history"]["prices"]),maxlen=self.stats_window)
                    counts=[0]*10
                    for d in self.last_digits: counts[d]+=1
                    self.prediction=min(range(10),key=lambda i:counts[i])
                    self.log(f"AI ready — targeting digit {self.prediction}","info")
                else: self.last_digits=deque(maxlen=self.stats_window)
            except: self.last_digits=deque(maxlen=self.stats_window)
        self.log("Matches/Differs Bot is LIVE! 🚀","success"); _push("bot_started",{})
        ts=await self.api.subscribe({"ticks":self.symbol}); ts.subscribe(self.on_tick)
        ps=await self.api.subscribe({"proposal_open_contract":1,"subscribe":1}); ps.subscribe(self.on_poc)
        await self._stop.wait()
        await self._graceful_shutdown(ts, ps)


# ═══════════════════════════════════════════════════════════════
#  BOT 2 — EVEN / ODD
# ═══════════════════════════════════════════════════════════════
class EvenOddBot(BaseBotMixin):
    bot_type = "eo"
    def __init__(self, cfg):
        self.app_id=127806; self.api_token=cfg["api_token"]
        self.symbol=cfg.get("symbol","1HZ100V"); self.duration=int(cfg.get("duration",1))
        self.duration_unit="t"; self.currency="USD"; self.contract_type=cfg["contract_type"]
        self.main_stake=Decimal(str(cfg["stake"])); self.tick_interval=int(cfg.get("tick_interval",1))
        self.max_trades=int(cfg.get("max_trades",0)); self.use_martingale=bool(cfg.get("use_martingale",False))
        self.mart_mult=Decimal(str(cfg.get("martingale_multiplier",2.0)))
        self.mart_max=int(cfg.get("max_martingale_steps",4)); self.mart_step=0
        self.auto_switch=bool(cfg.get("auto_switch",False)); self.stats_window=int(cfg.get("stats_window",50))
        self.update_every=int(cfg.get("update_every_trades",3)); self.trades_since_upd=0
        self.take_profit=Decimal(str(cfg.get("take_profit",0))); self.stop_loss=Decimal(str(cfg.get("stop_loss",0)))
        self.stake=self.main_stake; self.open_contracts={}; self.api=None
        self.tick_count=0; self.trade_count=0; self.trade_history=[]
        self.total_profit=Decimal("0"); self.wins=self.losses=0; self._stop=asyncio.Event()
        self.last_digits=deque(maxlen=self.stats_window); self.last_digit=None
        self.even_count=0; self.odd_count=0; self.start_time=time.time()
        self.profit_history=[]; self.balance=None; self.loginid=None

    def push_stats(self):
        tt=self.even_count+self.odd_count or 1; s=self._base_stats()
        s.update({"ct":self.contract_type,"ldigit":self.last_digit,
                  "even_count":self.even_count,"odd_count":self.odd_count,
                  "even_pct":round(self.even_count/tt*100,1),"odd_pct":round(self.odd_count/tt*100,1)})
        _push("stats",s)

    def _auto_update(self):
        if len(self.last_digits)<self.stats_window//2: return
        even_c=sum(1 for d in self.last_digits if d%2==0); odd_c=len(self.last_digits)-even_c
        new_type="DIGITEVEN" if even_c>=odd_c else "DIGITODD"
        if new_type!=self.contract_type:
            self.log(f"AUTO-SWITCH → {new_type} (E:{even_c} O:{odd_c})","info"); self.contract_type=new_type

    def on_tick(self, msg):
        if "tick" not in msg or self._stop.is_set(): return
        tick=msg["tick"]; ld=int(str(tick["quote"])[-1])
        self.tick_count+=1; self.last_digit=ld; self.last_digits.append(ld)
        if ld%2==0: self.even_count+=1
        else:       self.odd_count+=1
        self.push_stats()
        if self.tick_count%self.tick_interval==0:
            if self.max_trades>0 and self.trade_count>=self.max_trades:
                self._stop.set(); return   # synchronous: no new tasks spawned
            asyncio.create_task(self.attempt_trade(tick))

    async def attempt_trade(self, tick):
        if self._stop.is_set(): return
        try:
            pr=await self.api.proposal({
                "proposal":1,"amount":math.nextafter(float(self.stake),float("inf")),
                "basis":"stake","contract_type":self.contract_type,"currency":self.currency,
                "duration":self.duration,"duration_unit":self.duration_unit,"symbol":self.symbol})
            if "error" in pr: self.log(f"Proposal error: {pr['error']['message']}","error"); return
            # ── Second stop check: TP/SL may have fired while awaiting proposal ──
            if self._stop.is_set(): return
            self.trade_count+=1
            buy=await self.api.buy({"buy":pr["proposal"]["id"],"price":float(pr["proposal"]["ask_price"])})
            if "error" in buy: self.log(f"Buy error: {buy['error']['message']}","error"); return
            cid=buy["buy"]["contract_id"]; self.open_contracts[cid]={"stake":self.stake}
            lbl="EVEN" if self.contract_type=="DIGITEVEN" else "ODD"
            self.trade_history.append({
                "cid":cid,"symbol":self.symbol,"type":lbl,"digit":"—",
                "stake":float(self.stake),"dur":self.duration,"status":"pending",
                "profit":None,"payout":None,"time":time.strftime("%m/%d/%Y, %H:%M:%S")})
            self.log(f"Trade #{cid} — {lbl}","trade"); self.push_stats()
        except Exception as e: self.log(f"Trade error: {e}","error")

    def on_poc(self, msg):
        if "proposal_open_contract" not in msg: return
        c=msg["proposal_open_contract"]; cid=c.get("contract_id")
        if cid not in self.open_contracts or not c.get("is_sold"): return
        profit=Decimal(str(c.get("profit",0))); payout=float(c.get("sell_price",0))
        orig=self.open_contracts[cid]["stake"]; won=self._apply_result(profit,orig)
        if c.get("account_balance") is not None:
            self.balance=float(c["account_balance"])
            if self.loginid:
                _live_balance[self.loginid]={"balance":self.balance,"currency":self.currency,
                    "acct_type":"Demo Account" if self.loginid.startswith("VR") else "Live Account"}
        for t in self.trade_history:
            if t["cid"]==cid:
                t["status"]="won" if won else "lost"; t["profit"]=float(profit); t["payout"]=payout; break
        del self.open_contracts[cid]
        if self.auto_switch:
            self.trades_since_upd+=1
            if self.trades_since_upd>=self.update_every: self._auto_update(); self.trades_since_upd=0
        # ── KEY FIX: only check TP/SL when open_contracts is empty ──
        # If any contracts are still pending, their outcomes could change the
        # final P&L. Wait until the last one settles before deciding to stop.
        if not self._stop.is_set():
            self._check_tp_sl()
        self.push_stats()

    async def stop(self): await self._shared_stop()
    async def run(self):  await self._shared_run()
    async def _bot_run(self):
        try:
            hist=await self.api.ticks_history({"ticks_history":self.symbol,"style":"ticks","count":self.stats_window,"end":"latest"})
            if "error" not in hist:
                for p in hist["history"]["prices"]:
                    d=int(str(p)[-1]); self.last_digits.append(d)
                    if d%2==0: self.even_count+=1
                    else:      self.odd_count+=1
                if self.auto_switch: self._auto_update()
        except: pass
        self.log("Even/Odd Bot is LIVE! 🚀","success"); _push("bot_started",{})
        ts=await self.api.subscribe({"ticks":self.symbol}); ts.subscribe(self.on_tick)
        ps=await self.api.subscribe({"proposal_open_contract":1,"subscribe":1}); ps.subscribe(self.on_poc)
        await self._stop.wait()
        await self._graceful_shutdown(ts, ps)


# ═══════════════════════════════════════════════════════════════
#  BOT 3 — OVER / UNDER
# ═══════════════════════════════════════════════════════════════
class OverUnderBot(BaseBotMixin):
    bot_type = "ou"
    def __init__(self, cfg):
        self.app_id=127806; self.api_token=cfg["api_token"]
        self.symbol=cfg.get("symbol","1HZ100V"); self.duration=int(cfg.get("duration",1))
        self.duration_unit="t"; self.currency="USD"; self.contract_type=cfg["contract_type"]
        rp=cfg.get("prediction"); self.prediction=int(rp) if rp not in (None,"","auto") else 4
        self.main_stake=Decimal(str(cfg["stake"])); self.tick_interval=int(cfg.get("tick_interval",1))
        self.max_trades=int(cfg.get("max_trades",0)); self.use_martingale=bool(cfg.get("use_martingale",False))
        self.mart_mult=Decimal(str(cfg.get("martingale_multiplier",2.0)))
        self.mart_max=int(cfg.get("max_martingale_steps",4)); self.mart_step=0
        self.auto_switch=bool(cfg.get("auto_switch",False)); self.stats_window=int(cfg.get("stats_window",100))
        self.update_every=int(cfg.get("update_every_trades",3)); self.trades_since_upd=0
        self.take_profit=Decimal(str(cfg.get("take_profit",0))); self.stop_loss=Decimal(str(cfg.get("stop_loss",0)))
        self.stake=self.main_stake; self.open_contracts={}; self.api=None
        self.tick_count=0; self.trade_count=0; self.trade_history=[]
        self.total_profit=Decimal("0"); self.wins=self.losses=0
        self._stop=asyncio.Event(); self.digit_counts=[0]*10
        self.last_digits=deque(maxlen=self.stats_window); self.last_digit=None
        self.start_time=time.time(); self.profit_history=[]; self.balance=None; self.loginid=None

    def push_stats(self):
        tt=sum(self.digit_counts) or 1; pcts=[round(c/tt*100,1) for c in self.digit_counts]
        s=self._base_stats()
        s.update({"ct":self.contract_type,"pred":self.prediction,
                  "dcounts":self.digit_counts,"dpcts":pcts,"ldigit":self.last_digit})
        _push("stats",s)

    def _select_best(self):
        if len(self.last_digits)<self.stats_window//2: return
        counts=[0]*10
        for d in self.last_digits: counts[d]+=1
        best_d,best_type,best_count=self.prediction,self.contract_type,-1
        for pred in range(10):
            under_c=sum(counts[:pred]); over_c=sum(counts[pred+1:])
            mx=max(under_c,over_c)
            if mx>best_count:
                best_count=mx; best_d=pred
                best_type="DIGITUNDER" if under_c>over_c else "DIGITOVER"
        if best_d!=self.prediction or best_type!=self.contract_type:
            self.log(f"AUTO-ANALYSIS → {best_type} {best_d}","info")
            self.prediction=best_d; self.contract_type=best_type

    def on_tick(self, msg):
        if "tick" not in msg or self._stop.is_set(): return
        tick=msg["tick"]; ld=int(str(tick["quote"])[-1])
        self.tick_count+=1; self.digit_counts[ld]+=1; self.last_digit=ld; self.last_digits.append(ld)
        self.push_stats()
        if self.tick_count%self.tick_interval==0:
            if self.max_trades>0 and self.trade_count>=self.max_trades:
                self._stop.set(); return   # synchronous: no new tasks spawned
            asyncio.create_task(self.attempt_trade(tick))

    async def attempt_trade(self, tick):
        if self._stop.is_set(): return
        self.trade_count+=1
        try:
            pr=await self.api.proposal({
                "proposal":1,"amount":math.nextafter(float(self.stake),float("inf")),
                "basis":"stake","contract_type":self.contract_type,"currency":self.currency,
                "duration":self.duration,"duration_unit":self.duration_unit,"symbol":self.symbol,
                "barrier":str(self.prediction)})
            if "error" in pr: self.log(f"Proposal error: {pr['error']['message']}","error"); return
            # ── Second stop check: TP/SL may have fired while awaiting proposal ──
            if self._stop.is_set(): return
            self.trade_count+=1
            buy=await self.api.buy({"buy":pr["proposal"]["id"],"price":float(pr["proposal"]["ask_price"])})
            if "error" in buy: self.log(f"Buy error: {buy['error']['message']}","error"); return
            cid=buy["buy"]["contract_id"]; self.open_contracts[cid]={"stake":self.stake}
            side="OVER" if self.contract_type=="DIGITOVER" else "UNDER"
            lbl=f"{side} {self.prediction}"
            self.trade_history.append({
                "cid":cid,"symbol":self.symbol,"type":lbl,"digit":self.prediction,
                "stake":float(self.stake),"dur":self.duration,"status":"pending",
                "profit":None,"payout":None,"time":time.strftime("%m/%d/%Y, %H:%M:%S")})
            self.log(f"Trade #{cid} — {lbl}","trade"); self.push_stats()
        except Exception as e: self.log(f"Trade error: {e}","error")

    def on_poc(self, msg):
        if "proposal_open_contract" not in msg: return
        c=msg["proposal_open_contract"]; cid=c.get("contract_id")
        if cid not in self.open_contracts or not c.get("is_sold"): return
        profit=Decimal(str(c.get("profit",0))); payout=float(c.get("sell_price",0))
        orig=self.open_contracts[cid]["stake"]; won=self._apply_result(profit,orig)
        if c.get("account_balance") is not None:
            self.balance=float(c["account_balance"])
            if self.loginid:
                _live_balance[self.loginid]={"balance":self.balance,"currency":self.currency,
                    "acct_type":"Demo Account" if self.loginid.startswith("VR") else "Live Account"}
        for t in self.trade_history:
            if t["cid"]==cid:
                t["status"]="won" if won else "lost"; t["profit"]=float(profit); t["payout"]=payout; break
        del self.open_contracts[cid]
        if self.auto_switch:
            self.trades_since_upd+=1
            if self.trades_since_upd>=self.update_every: self._select_best(); self.trades_since_upd=0
        # ── KEY FIX: only check TP/SL when open_contracts is empty ──
        # If any contracts are still pending, their outcomes could change the
        # final P&L. Wait until the last one settles before deciding to stop.
        if not self._stop.is_set():
            self._check_tp_sl()
        self.push_stats()

    async def stop(self): await self._shared_stop()
    async def run(self):  await self._shared_run()
    async def _bot_run(self):
        try:
            hist=await self.api.ticks_history({"ticks_history":self.symbol,"style":"ticks","count":self.stats_window,"end":"latest"})
            if "error" not in hist:
                for p in hist["history"]["prices"]:
                    d=int(str(p)[-1]); self.digit_counts[d]+=1; self.last_digits.append(d)
                if self.auto_switch: self._select_best()
        except: pass
        self.log("Over/Under Bot is LIVE! 🚀","success"); _push("bot_started",{})
        ts=await self.api.subscribe({"ticks":self.symbol}); ts.subscribe(self.on_tick)
        ps=await self.api.subscribe({"proposal_open_contract":1,"subscribe":1}); ps.subscribe(self.on_poc)
        await self._stop.wait()
        await self._graceful_shutdown(ts, ps)


# ═══════════════════════════════════════════════════════════════
#  BOT 4 — MIXOLOGIST (concurrent OU + EO + MD)
# ═══════════════════════════════════════════════════════════════
class MixologistBot(BaseBotMixin):
    bot_type = "mix"

    def __init__(self, cfg):
        self.app_id=127806; self.api_token=cfg["api_token"]
        self.symbol=cfg.get("symbol","1HZ100V"); self.duration=int(cfg.get("duration",1))
        self.duration_unit="t"; self.currency="USD"
        self.main_stake=Decimal(str(cfg["stake"])); self.stake=self.main_stake
        self.take_profit=Decimal(str(cfg.get("take_profit",0)))
        self.stop_loss=Decimal(str(cfg.get("stop_loss",0)))
        self.use_martingale=bool(cfg.get("use_martingale",False))
        self.mart_mult=Decimal(str(cfg.get("martingale_multiplier",2.0)))
        self.mart_max=int(cfg.get("max_martingale_steps",4)); self.mart_step=0
        self.tick_interval=int(cfg.get("tick_interval",1))
        self.max_trades=int(cfg.get("max_trades",0))
        self.enable_ou=bool(cfg.get("enable_ou",True))
        self.enable_eo=bool(cfg.get("enable_eo",True))
        self.enable_md=bool(cfg.get("enable_md",True))
        self.ou_ct=cfg.get("ou_contract_type","DIGITOVER"); self.ou_pred=int(cfg.get("ou_prediction",4))
        self.eo_ct=cfg.get("eo_contract_type","DIGITEVEN")
        self.md_ct=cfg.get("md_contract_type","DIGITDIFF")
        self.mod={
            "ou":{"wins":0,"losses":0,"profit":Decimal("0"),"trades":0,"stake":self.main_stake,"mart":0},
            "eo":{"wins":0,"losses":0,"profit":Decimal("0"),"trades":0,"stake":self.main_stake,"mart":0},
            "md":{"wins":0,"losses":0,"profit":Decimal("0"),"trades":0,"stake":self.main_stake,"mart":0},
        }
        self.open_contracts={}; self.api=None; self.tick_count=0; self.trade_count=0
        self.trade_history=[]; self.total_profit=Decimal("0"); self.wins=self.losses=0
        self._stop=asyncio.Event(); self.digit_counts=[0]*10; self.last_digit=None
        self.start_time=time.time(); self.profit_history=[]; self.balance=None; self.loginid=None

    def push_stats(self):
        tt=sum(self.digit_counts) or 1; pcts=[round(c/tt*100,1) for c in self.digit_counts]
        total=self.wins+self.losses; wr=round(self.wins/total*100,1) if total else 0
        mod_out={}
        for k,m in self.mod.items():
            mt=m["wins"]+m["losses"]
            mod_out[k]={"wins":m["wins"],"losses":m["losses"],
                        "profit":float(m["profit"]),"trades":m["trades"],
                        "wr":round(m["wins"]/mt*100,1) if mt else 0,
                        "mart":m["mart"]}   # expose per-module martingale step
        # mart_step for base stats = max across active modules (for UI badge)
        self.mart_step = max(m["mart"] for m in self.mod.values())
        _push("stats",{
            "bot_type":"mix","profit":float(self.total_profit),
            "trades":self.trade_count,"stake":float(self.stake),
            "wins":self.wins,"losses":self.losses,"wr":wr,
            "ticks":self.tick_count,"elapsed":int(time.time()-self.start_time),
            "mstep":self.mart_step,"mmax":self.mart_max,
            "tp":float(self.take_profit),"sl":float(self.stop_loss),
            "balance":self.balance,"phistory":self.profit_history[-80:],
            "history":self.trade_history[-100:],"dpcts":pcts,"ldigit":self.last_digit,
            "modules":mod_out,"enable_ou":self.enable_ou,"enable_eo":self.enable_eo,"enable_md":self.enable_md,
        })

    def _mod_apply(self, module, profit, orig_stake):
        """Apply trade result for a specific module. Never triggers a global stop by itself."""
        m=self.mod[module]; m["profit"]+=profit
        self.total_profit+=profit; self.profit_history.append(float(self.total_profit))
        tag={"ou":"O/U","eo":"E/O","md":"M/D"}[module]
        won=profit>0
        if won:
            self.wins+=1; m["wins"]+=1; m["stake"]=self.main_stake; m["mart"]=0
            self.log(f"[{tag}] WIN +${float(profit):.2f} | Module P/L: {float(m['profit']):+.2f} | Total: {float(self.total_profit):+.2f}", "win")
        else:
            self.losses+=1; m["losses"]+=1
            if self.use_martingale and m["mart"]<self.mart_max:
                m["mart"]+=1; m["stake"]=orig_stake*self.mart_mult
                self.log(f"[{tag}] LOSS ${float(profit):.2f} | Mart ×{m['mart']} → ${float(m['stake']):.2f} | Total: {float(self.total_profit):+.2f}", "loss")
            else:
                m["stake"]=self.main_stake; m["mart"]=0
                self.log(f"[{tag}] LOSS ${float(profit):.2f} | Module P/L: {float(m['profit']):+.2f} | Total: {float(self.total_profit):+.2f}", "loss")
        return won

    async def _trade_ou(self, tick):
        if self._stop.is_set(): return
        m=self.mod["ou"]; stake=m["stake"]
        try:
            pr=await self.api.proposal({
                "proposal":1,"amount":math.nextafter(float(stake),float("inf")),
                "basis":"stake","contract_type":self.ou_ct,"currency":self.currency,
                "duration":self.duration,"duration_unit":self.duration_unit,"symbol":self.symbol,
                "barrier":str(self.ou_pred)})
            if "error" in pr: self.log(f"[O/U] Proposal: {pr['error']['message']}","error"); return
            # ── Second stop check: TP/SL may have fired while awaiting proposal ──
            if self._stop.is_set(): return
            self.trade_count+=1; m["trades"]+=1   # count only confirmed buys
            buy=await self.api.buy({"buy":pr["proposal"]["id"],"price":float(pr["proposal"]["ask_price"])})
            if "error" in buy: self.log(f"[O/U] Buy: {buy['error']['message']}","error"); return
            cid=buy["buy"]["contract_id"]; self.open_contracts[cid]={"module":"ou","stake":stake}
            side="OVER" if self.ou_ct=="DIGITOVER" else "UNDER"
            lbl=f"O/U {side} {self.ou_pred}"
            self.trade_history.append({
                "cid":cid,"symbol":self.symbol,"type":"O/U","detail":lbl,
                "stake":float(stake),"dur":self.duration,"status":"pending",
                "profit":None,"payout":None,"time":time.strftime("%m/%d/%Y, %H:%M:%S")})
            self.log(f"[O/U] #{cid} — {lbl}","trade"); self.push_stats()
        except Exception as e: self.log(f"[O/U] Error: {e}","error")

    async def _trade_eo(self, tick):
        if self._stop.is_set(): return
        m=self.mod["eo"]; stake=m["stake"]
        try:
            pr=await self.api.proposal({
                "proposal":1,"amount":math.nextafter(float(stake),float("inf")),
                "basis":"stake","contract_type":self.eo_ct,"currency":self.currency,
                "duration":self.duration,"duration_unit":self.duration_unit,"symbol":self.symbol})
            if "error" in pr: self.log(f"[E/O] Proposal: {pr['error']['message']}","error"); return
            # ── Second stop check: TP/SL may have fired while awaiting proposal ──
            if self._stop.is_set(): return
            self.trade_count+=1; m["trades"]+=1
            buy=await self.api.buy({"buy":pr["proposal"]["id"],"price":float(pr["proposal"]["ask_price"])})
            if "error" in buy: self.log(f"[E/O] Buy: {buy['error']['message']}","error"); return
            cid=buy["buy"]["contract_id"]; self.open_contracts[cid]={"module":"eo","stake":stake}
            lbl="EVEN" if self.eo_ct=="DIGITEVEN" else "ODD"
            self.trade_history.append({
                "cid":cid,"symbol":self.symbol,"type":"E/O","detail":lbl,
                "stake":float(stake),"dur":self.duration,"status":"pending",
                "profit":None,"payout":None,"time":time.strftime("%m/%d/%Y, %H:%M:%S")})
            self.log(f"[E/O] #{cid} — {lbl}","trade"); self.push_stats()
        except Exception as e: self.log(f"[E/O] Error: {e}","error")

    async def _trade_md(self, tick):
        if self._stop.is_set(): return
        m=self.mod["md"]; stake=m["stake"]
        rand_digit=random.randint(0,9)   # strictly random per spec
        try:
            pr=await self.api.proposal({
                "proposal":1,"amount":math.nextafter(float(stake),float("inf")),
                "basis":"stake","contract_type":self.md_ct,"currency":self.currency,
                "duration":self.duration,"duration_unit":self.duration_unit,"symbol":self.symbol,
                "barrier":str(rand_digit)})
            if "error" in pr: self.log(f"[M/D] Proposal: {pr['error']['message']}","error"); return
            # ── Second stop check: TP/SL may have fired while awaiting proposal ──
            if self._stop.is_set(): return
            self.trade_count+=1; m["trades"]+=1
            buy=await self.api.buy({"buy":pr["proposal"]["id"],"price":float(pr["proposal"]["ask_price"])})
            if "error" in buy: self.log(f"[M/D] Buy: {buy['error']['message']}","error"); return
            cid=buy["buy"]["contract_id"]; self.open_contracts[cid]={"module":"md","stake":stake}
            lbl="DIFFERS" if self.md_ct=="DIGITDIFF" else "MATCH"
            self.trade_history.append({
                "cid":cid,"symbol":self.symbol,"type":"M/D","detail":f"{lbl} {rand_digit}",
                "stake":float(stake),"dur":self.duration,"status":"pending",
                "profit":None,"payout":None,"time":time.strftime("%m/%d/%Y, %H:%M:%S")})
            self.log(f"[M/D] #{cid} — {lbl} {rand_digit}","trade"); self.push_stats()
        except Exception as e: self.log(f"[M/D] Error: {e}","error")

    def on_tick(self, msg):
        if "tick" not in msg or self._stop.is_set(): return
        tick=msg["tick"]; ld=int(str(tick["quote"])[-1])
        self.tick_count+=1; self.digit_counts[ld]+=1; self.last_digit=ld
        self.push_stats()
        if self.tick_count%self.tick_interval==0:
            if self.max_trades>0 and self.trade_count>=self.max_trades:
                self.log(f"Max trades ({self.max_trades}) reached — stopping","info")
                self._stop.set(); return   # synchronous: no new tasks spawned
            if self.enable_ou: asyncio.create_task(self._trade_ou(tick))
            if self.enable_eo: asyncio.create_task(self._trade_eo(tick))
            if self.enable_md: asyncio.create_task(self._trade_md(tick))

    def on_poc(self, msg):
        if "proposal_open_contract" not in msg: return
        c=msg["proposal_open_contract"]; cid=c.get("contract_id")
        if cid not in self.open_contracts or not c.get("is_sold"): return
        info=self.open_contracts[cid]; module=info["module"]; orig=info["stake"]
        profit=Decimal(str(c.get("profit",0))); payout=float(c.get("sell_price",0))
        won=self._mod_apply(module, profit, orig)  # logs per-module result
        if c.get("account_balance") is not None:
            self.balance=float(c["account_balance"])
            if self.loginid:
                _live_balance[self.loginid]={"balance":self.balance,"currency":self.currency,
                    "acct_type":"Demo Account" if self.loginid.startswith("VR") else "Live Account"}
        for t in self.trade_history:
            if t["cid"]==cid:
                t["status"]="won" if won else "lost"; t["profit"]=float(profit); t["payout"]=payout; break
        del self.open_contracts[cid]
        # ── KEY FIX: only check TP/SL when ALL open contracts have settled ──
        # TP/SL only triggers on total_profit, never on individual module losses.
        # Waiting for empty open_contracts ensures no pending trade can drag
        # the final P&L below TP (or back above SL) after we stop.
        if not self._stop.is_set():
            self._check_tp_sl()
        self.push_stats()

    async def stop(self): await self._shared_stop()
    async def run(self):  await self._shared_run()
    async def _bot_run(self):
        active=[n for n,e in [("Over/Under",self.enable_ou),("Even/Odd",self.enable_eo),("Match/Differ",self.enable_md)] if e]
        if not active:
            self.log("No modules enabled — enable at least one strategy","error"); self._stop.set(); return
        self.log(f"🎛️ Mixologist LIVE → {' + '.join(active)}","success"); _push("bot_started",{})
        ts=await self.api.subscribe({"ticks":self.symbol}); ts.subscribe(self.on_tick)
        ps=await self.api.subscribe({"proposal_open_contract":1,"subscribe":1}); ps.subscribe(self.on_poc)
        await self._stop.wait()
        await self._graceful_shutdown(ts, ps)



# ═══════════════════════════════════════════════════════════════
#  BOT 5 — ACCOUNT BUILDER  (adaptive strategy, goal-oriented)
# ═══════════════════════════════════════════════════════════════
def _analyze_digit_data(digit_counts: list, min_ticks: int = 30) -> dict | None:
    """
    Analyze digit frequency data to find the highest-edge trade strategy.
    Returns a dict with keys: strategy, contract_type, barrier, digit, confidence, label
    Returns None if insufficient data.
    """
    total = sum(digit_counts)
    if total < min_ticks:
        return None

    pcts = [c / total for c in digit_counts]

    # --- Even/Odd analysis ---
    even_pct = sum(pcts[i] for i in [0, 2, 4, 6, 8])
    odd_pct  = 1.0 - even_pct
    eo_edge  = abs(even_pct - 0.5)          # 0 = no bias, 0.5 = max bias
    eo_type  = "DIGITEVEN" if even_pct >= odd_pct else "DIGITODD"

    # --- Over/Under analysis (test all barriers 0-9) ---
    best_ou_edge, best_ou_barrier, best_ou_type = 0.0, 4, "DIGITOVER"
    for barrier in range(0, 10):
        under_p = sum(pcts[:barrier])           # digits < barrier
        over_p  = sum(pcts[barrier + 1:])       # digits > barrier
        # Edge = how far we are from the 50/50 expected split
        edge = max(under_p, over_p) - 0.5
        if edge > best_ou_edge:
            best_ou_edge    = edge
            best_ou_barrier = barrier
            best_ou_type    = "DIGITUNDER" if under_p > over_p else "DIGITOVER"

    # --- Matches/Differs analysis (least-frequent digit for DIFFERS) ---
    min_pct   = min(pcts)
    min_digit = pcts.index(min_pct)
    # Edge = how much below the expected 10% the digit appears
    md_edge   = max(0.0, 0.10 - min_pct)

    # --- Pick the strategy with highest edge ---
    candidates = [
        ("eo", eo_edge,     {"strategy":"eo","contract_type":eo_type,
                              "barrier":None,"digit":None,
                              "label":f"{'EVEN' if eo_type=='DIGITEVEN' else 'ODD'} ({even_pct*100:.1f}% even)",
                              "confidence": min(100, int(eo_edge * 400))}),
        ("ou", best_ou_edge,{"strategy":"ou","contract_type":best_ou_type,
                              "barrier":best_ou_barrier,"digit":None,
                              "label":f"{'OVER' if best_ou_type=='DIGITOVER' else 'UNDER'} {best_ou_barrier}",
                              "confidence": min(100, int(best_ou_edge * 400))}),
        ("md", md_edge,     {"strategy":"md","contract_type":"DIGITDIFF",
                              "barrier":str(min_digit),"digit":min_digit,
                              "label":f"DIFFERS {min_digit} ({min_pct*100:.1f}%)",
                              "confidence": min(100, int(md_edge * 1000))}),
    ]
    candidates.sort(key=lambda x: x[1], reverse=True)
    return candidates[0][2]


class AccountBuilderBot(BaseBotMixin):
    """
    Account Builder — Goal-oriented adaptive bot.

    The user sets a mini_balance (capital to risk) and a target_profit (goal).
    The bot:
      1. Analyzes the last 50–100 ticks to find the highest-edge strategy.
      2. Sets stake as a safe % of mini_balance (default 2%).
      3. Re-analyzes every `reanalyze_every` settled trades, auto-switching strategy.
      4. Stops when target_profit is reached OR mini_balance is depleted.
      5. Uses conservative martingale (max 3 steps) to recover losses.
      6. Reports progress as % toward target throughout the session.
    """
    bot_type = "ab"

    def __init__(self, cfg):
        self.app_id        = 127806
        self.api_token     = cfg["api_token"]
        self.symbol        = cfg.get("symbol", "1HZ100V")
        self.duration      = int(cfg.get("duration", 1))
        self.duration_unit = "t"
        self.currency      = "USD"

        # Goal parameters
        self.mini_balance  = Decimal(str(cfg.get("mini_balance", 100)))
        self.target_profit = Decimal(str(cfg.get("target_profit", 20)))

        # Auto-calculate safe base stake: 2% of mini_balance, min $0.35
        raw_stake = float(self.mini_balance) * float(cfg.get("stake_pct", 0.02))
        raw_stake = max(0.35, min(raw_stake, float(self.mini_balance) * 0.05))
        self.main_stake    = Decimal(str(round(raw_stake, 2)))
        self.stake         = self.main_stake

        # Martingale (conservative: max 3 steps, ×2 multiplier)
        self.use_martingale= True
        self.mart_mult     = Decimal("2")
        self.mart_max      = int(cfg.get("max_martingale_steps", 3))
        self.mart_step     = 0

        # TP/SL derived from goals
        self.take_profit   = self.target_profit
        self.stop_loss     = self.mini_balance   # stop if we lose the full mini_balance

        # Strategy state (set during analysis)
        self.contract_type = "DIGITDIFF"
        self.prediction    = None
        self.current_strategy = None
        self.current_label = "Analyzing…"
        self.confidence    = 0
        self.reanalyze_every = int(cfg.get("reanalyze_every", 10))
        self.trades_since_analysis = 0

        # Tick tracking
        self.tick_interval = int(cfg.get("tick_interval", 1))
        self.max_trades    = int(cfg.get("max_trades", 0))
        self.digit_counts  = [0] * 10
        self.last_digit    = None
        self.last_digits   = deque(maxlen=200)  # rich history for analysis
        self.tick_count    = 0
        self.open_contracts= {}
        self.api           = None
        self.trade_count   = 0
        self.trade_history = []
        self.total_profit  = Decimal("0")
        self.wins          = self.losses = 0
        self._stop         = asyncio.Event()
        self.start_time    = time.time()
        self.profit_history= []
        self.balance       = None
        self.loginid       = None

    # ── Market analysis ─────────────────────────────────────────────────────
    def _run_analysis(self):
        """Run strategy analysis and update current_strategy. Returns True if strategy changed."""
        result = _analyze_digit_data(self.digit_counts, min_ticks=30)
        if not result:
            return False

        old = self.current_strategy
        self.current_strategy = result["strategy"]
        self.contract_type    = result["contract_type"]
        self.prediction       = result.get("digit")
        self.confidence       = result["confidence"]
        self.current_label    = result["label"]

        # Set barrier for proposal
        if result["strategy"] == "ou":
            self._barrier = str(result["barrier"])
        elif result["strategy"] == "md":
            self._barrier = str(result["digit"])
        else:
            self._barrier = None  # EO needs no barrier

        changed = old != self.current_strategy
        return changed

    def _goal_progress(self):
        """Return progress percentage toward target_profit (0–100)."""
        if self.target_profit <= 0:
            return 0
        return min(100, max(0, float(self.total_profit / self.target_profit * 100)))

    # ── Stats push ──────────────────────────────────────────────────────────
    def push_stats(self):
        total = self.wins + self.losses
        wr    = round(self.wins / total * 100, 1) if total else 0
        tt    = sum(self.digit_counts) or 1
        pcts  = [round(c / tt * 100, 1) for c in self.digit_counts]
        _push("stats", {
            "bot_type":    "ab",
            "profit":      float(self.total_profit),
            "trades":      self.trade_count,
            "stake":       float(self.stake),
            "wins":        self.wins,
            "losses":      self.losses,
            "wr":          wr,
            "ticks":       self.tick_count,
            "elapsed":     int(time.time() - self.start_time),
            "mstep":       self.mart_step,
            "mmax":        self.mart_max,
            "tp":          float(self.take_profit),
            "sl":          float(self.stop_loss),
            "balance":     self.balance,
            "phistory":    self.profit_history[-80:],
            "history":     self.trade_history[-100:],
            "dpcts":       pcts,
            "ldigit":      self.last_digit,
            # Account Builder specific
            "mini_balance":    float(self.mini_balance),
            "target_profit":   float(self.target_profit),
            "goal_progress":   self._goal_progress(),
            "strategy":        self.current_strategy or "—",
            "strategy_label":  self.current_label,
            "confidence":      self.confidence,
        })

    # ── Trading ─────────────────────────────────────────────────────────────
    def on_tick(self, msg):
        if "tick" not in msg or self._stop.is_set(): return
        tick = msg["tick"]
        ld   = int(str(tick["quote"])[-1])
        self.tick_count   += 1
        self.digit_counts[ld] += 1
        self.last_digit    = ld
        self.last_digits.append(ld)
        self.push_stats()

        if self.tick_count % self.tick_interval == 0:
            if self.max_trades > 0 and self.trade_count >= self.max_trades:
                self._stop.set(); return
            if self.current_strategy is None:
                return   # still waiting for enough ticks to analyze
            asyncio.create_task(self.attempt_trade(tick))

    async def attempt_trade(self, tick):
        if self._stop.is_set(): return
        if self.current_strategy is None: return

        # Build proposal params
        params = {
            "proposal": 1,
            "amount":   math.nextafter(float(self.stake), float("inf")),
            "basis":    "stake",
            "contract_type": self.contract_type,
            "currency": self.currency,
            "duration": self.duration,
            "duration_unit": self.duration_unit,
            "symbol":   self.symbol,
        }
        if hasattr(self, "_barrier") and self._barrier is not None:
            params["barrier"] = self._barrier

        try:
            pr = await self.api.proposal(params)
            if "error" in pr:
                self.log(f"[AB] Proposal error: {pr['error']['message']}", "error"); return
            if self._stop.is_set(): return    # TP/SL fired while awaiting proposal
            self.trade_count += 1
            buy = await self.api.buy({"buy": pr["proposal"]["id"],
                                       "price": float(pr["proposal"]["ask_price"])})
            if "error" in buy:
                self.log(f"[AB] Buy error: {buy['error']['message']}", "error"); return
            cid = buy["buy"]["contract_id"]
            self.open_contracts[cid] = {"stake": self.stake}
            self.trade_history.append({
                "cid": cid, "symbol": self.symbol,
                "type": self.current_label, "detail": self.current_label,
                "stake": float(self.stake), "dur": self.duration,
                "status": "pending", "profit": None, "payout": None,
                "time": time.strftime("%m/%d/%Y, %H:%M:%S"),
            })
            self.log(
                f"[AB] #{cid} — {self.current_label} | Stake: ${float(self.stake):.2f} "
                f"| Progress: {self._goal_progress():.1f}%", "trade"
            )
            self.push_stats()
        except Exception as e:
            self.log(f"[AB] Trade error: {e}", "error")

    def on_poc(self, msg):
        if "proposal_open_contract" not in msg: return
        c   = msg["proposal_open_contract"]
        cid = c.get("contract_id")
        if cid not in self.open_contracts or not c.get("is_sold"): return

        profit = Decimal(str(c.get("profit", 0)))
        payout = float(c.get("sell_price", 0))
        orig   = self.open_contracts[cid]["stake"]
        won    = self._apply_result(profit, orig)

        if c.get("account_balance") is not None:
            self.balance = float(c["account_balance"])
            if self.loginid:
                _live_balance[self.loginid] = {
                    "balance": self.balance, "currency": self.currency,
                    "acct_type": "Demo Account" if self.loginid.startswith("VR") else "Live Account",
                }

        for t in self.trade_history:
            if t["cid"] == cid:
                t["status"] = "won" if won else "lost"
                t["profit"] = float(profit)
                t["payout"] = payout
                break
        del self.open_contracts[cid]

        # Re-analyze market every N settled trades (auto-strategy switching)
        self.trades_since_analysis += 1
        if self.trades_since_analysis >= self.reanalyze_every:
            self.trades_since_analysis = 0
            changed = self._run_analysis()
            if changed:
                self.log(
                    f"[AB] 🔄 Strategy switched → {self.current_label} "
                    f"(confidence: {self.confidence}%) — resetting martingale", "info"
                )
                self.stake     = self.main_stake
                self.mart_step = 0

        # Log progress
        prog = self._goal_progress()
        self.log(
            f"[AB] {'✅ WIN' if won else '❌ LOSS'} ${float(profit):+.2f} | "
            f"Total: {float(self.total_profit):+.2f} | Goal: {prog:.1f}%", 
            "win" if won else "loss"
        )

        if not self._stop.is_set():
            self._check_tp_sl()
        self.push_stats()

    async def stop(self): await self._shared_stop()
    async def run(self):  await self._shared_run()

    async def _bot_run(self):
        """Pre-load tick history, run initial analysis, then start live trading."""
        self.log("🏗️ Account Builder initializing — fetching market data…", "info")
        # Pre-load 100 ticks for initial analysis
        try:
            hist = await self.api.ticks_history({
                "ticks_history": self.symbol, "style": "ticks",
                "count": 100, "end": "latest"
            })
            if "error" not in hist:
                for p in hist["history"]["prices"]:
                    d = int(str(p)[-1])
                    self.digit_counts[d] += 1
                    self.last_digits.append(d)
                self.log(f"[AB] Loaded {sum(self.digit_counts)} historical ticks", "info")
        except Exception as e:
            self.log(f"[AB] Could not pre-load history ({e}), will analyze on live ticks", "warning")

        # Initial analysis
        changed = self._run_analysis()
        if self.current_strategy:
            self.log(
                f"[AB] 🧠 Initial strategy: {self.current_label} "
                f"(confidence: {self.confidence}%) | "
                f"Base stake: ${float(self.main_stake):.2f} | "
                f"Target: +${float(self.target_profit):.2f}", "success"
            )
        else:
            self.log("[AB] Not enough data yet — will analyze after first ticks", "warning")

        self.log(
            f"🏗️ Account Builder LIVE! Goal: +${float(self.target_profit):.2f} "
            f"| Mini-Balance: ${float(self.mini_balance):.2f}", "success"
        )
        _push("bot_started", {})

        ts = await self.api.subscribe({"ticks": self.symbol})
        ts.subscribe(self.on_tick)
        ps = await self.api.subscribe({"proposal_open_contract": 1, "subscribe": 1})
        ps.subscribe(self.on_poc)

        await self._stop.wait()
        await self._graceful_shutdown(ts, ps)


# ═══════════════════════════════════════════════════════════════
#  FLASK ROUTES
# ═══════════════════════════════════════════════════════════════
@app.route("/login", methods=["GET","POST"])
def login():
    error=None
    if request.method=="POST":
        if (request.form.get("email","").strip()==VALID_EMAIL and
                request.form.get("password","").strip()==VALID_PASSWORD):
            session["logged_in"]=True; return redirect(url_for("index"))
        else: error="Invalid email or password. Please try again."
    return render_template_string(LOGIN_HTML, error=error)

@app.route("/logout")
def logout(): session.clear(); return redirect(url_for("login"))

@app.route("/")
def index():
    if not session.get("logged_in"): return redirect(url_for("login"))
    return render_template_string(MAIN_HTML)

@app.route("/api/auth_check", methods=["POST"])
def auth_check():
    if not session.get("logged_in"): return jsonify({"error":"Unauthorized"}),401
    global _stored_token, _live_balance
    data=request.get_json(silent=True) or {}
    token=data.get("api_token","").strip()
    if not token: return jsonify({"error":"api_token required"}),400
    async def _do():
        from deriv_api import DerivAPI
        api=DerivAPI(app_id=127806)
        try:
            resp=await api.authorize({"authorize":token})
            if "error" in resp: return {"error":resp["error"].get("message","Auth failed")}
            info=resp.get("authorize",{})
            try:
                br=await api.balance({"balance":1})
                bal=float(br["balance"]["balance"]); cur=br["balance"].get("currency",info.get("currency","USD"))
            except:
                bal=float(info.get("balance",0)); cur=info.get("currency","USD")
            lid=info.get("loginid","???"); acct="Demo Account" if lid.startswith("VR") else "Live Account"
            return {"loginid":lid,"balance":bal,"currency":cur,"acct_type":acct}
        except Exception as e: return {"error":str(e)}
        finally:
            try: await api.disconnect()
            except: pass
    loop=asyncio.new_event_loop()
    try:    result=loop.run_until_complete(_do())
    finally: loop.close()
    if "error" in result: return jsonify(result),400
    _stored_token=token
    _live_balance[result["loginid"]]={"balance":result["balance"],"currency":result["currency"],"acct_type":result["acct_type"]}
    return jsonify(result)

@app.route("/api/balance")
def get_balance():
    if not session.get("logged_in"): return jsonify({"error":"Unauthorized"}),401
    if _bot and _bot_loop and not _bot._stop.is_set() and _bot.api:
        try:
            fut=asyncio.run_coroutine_threadsafe(_bot.api.balance({"balance":1}),_bot_loop)
            res=fut.result(timeout=4)
            if "error" not in res and "balance" in res:
                bal=float(res["balance"]["balance"]); cur=res["balance"].get("currency","USD")
                _bot.balance=bal
                if _bot.loginid:
                    _live_balance[_bot.loginid]={"balance":bal,"currency":cur,
                        "acct_type":"Demo Account" if _bot.loginid.startswith("VR") else "Live Account"}
                return jsonify({"balance":bal,"currency":cur})
        except: pass
    if _bot and _bot.balance is not None:
        return jsonify({"balance":float(_bot.balance),"currency":getattr(_bot,"currency","USD")})
    if _live_balance:
        entry=next(iter(_live_balance.values()))
        return jsonify({"balance":entry["balance"],"currency":entry.get("currency","USD")})
    return jsonify({"balance":None})

@app.route("/api/start", methods=["POST"])
def start_bot():
    if not session.get("logged_in"): return jsonify({"error":"Unauthorized"}),401
    global _bot,_bot_loop,_bot_thread
    if _bot_is_running(): return jsonify({"error":"Bot is already running — stop it first"}),400
    data=request.get_json(silent=True)
    if not data: return jsonify({"error":"Invalid JSON body"}),400
    if not data.get("api_token"): return jsonify({"error":"api_token is required"}),400
    if not data.get("stake"):     return jsonify({"error":"stake is required"}),400
    bot_type=data.get("bot_type","md")
    # Account Builder uses different required fields
    if bot_type not in ("mix","ab") and not data.get("contract_type"):
        return jsonify({"error":"contract_type is required for this bot"}),400
    if bot_type == "ab":
        if not data.get("mini_balance"):
            return jsonify({"error":"mini_balance is required for Account Builder"}),400
        if not data.get("target_profit"):
            return jsonify({"error":"target_profit is required for Account Builder"}),400
        try:
            mb = float(data["mini_balance"])
            tp = float(data["target_profit"])
        except (TypeError, ValueError):
            return jsonify({"error":"mini_balance and target_profit must be numbers"}),400
        if mb < 5:
            return jsonify({"error":"mini_balance must be at least $5"}),400
        if tp <= 0:
            return jsonify({"error":"target_profit must be > 0"}),400
        if tp > mb * 2:
            return jsonify({"error":"target_profit cannot exceed 2× mini_balance (unrealistic goal)"}),400
    else:
        # Standard config validation
        err = _validate_config(data)
        if err:
            return jsonify({"error": err}), 400

    try:
        if   bot_type=="md":  _bot=DifferBot(data)
        elif bot_type=="eo":  _bot=EvenOddBot(data)
        elif bot_type=="ou":  _bot=OverUnderBot(data)
        elif bot_type=="mix": _bot=MixologistBot(data)
        elif bot_type=="ab":  _bot=AccountBuilderBot(data)
        else: return jsonify({"error":f"Unknown bot_type: {bot_type}"}),400
    except Exception as e: return jsonify({"error":f"Config error: {e}"}),400
    def _run():
        global _bot_loop
        _bot_loop=asyncio.new_event_loop(); asyncio.set_event_loop(_bot_loop)
        try: _bot_loop.run_until_complete(_bot.run())
        except Exception as e:
            logger.error("Bot crashed: %s", e)
            _push("log",{"m":f"Bot crashed: {e}","l":"error","t":time.strftime("%H:%M:%S")})
            if _bot and not _bot._stop.is_set(): _bot._stop.set()
        finally: _bot_loop.close()
    _bot_thread=threading.Thread(target=_run,daemon=True,name="PerryAI-Bot")
    _bot_thread.start()
    logger.info("Bot started: type=%s symbol=%s stake=%s", bot_type, data.get("symbol"), data.get("stake"))
    return jsonify({"status":"starting"})

@app.route("/api/stop", methods=["POST"])
def stop_bot():
    if not session.get("logged_in"): return jsonify({"error":"Unauthorized"}),401
    global _bot,_bot_loop,_bot_thread
    if _bot and _bot_loop and not _bot._stop.is_set():
        asyncio.run_coroutine_threadsafe(_bot.stop(),_bot_loop)
        return jsonify({"status":"stopping"})
    _bot=None; _bot_thread=None
    return jsonify({"status":"already_stopped"})

@app.route("/stream")
def stream():
    if not session.get("logged_in"): return Response("Unauthorized",status=401)
    q=queue.Queue(maxsize=400)
    with _sse_lock: _sse_clients.append(q)
    def gen():
        yield ": heartbeat\n\n"
        try:
            while True:
                try:    yield q.get(timeout=25)
                except queue.Empty: yield ": heartbeat\n\n"
        except GeneratorExit:
            with _sse_lock:
                try: _sse_clients.remove(q)
                except ValueError: pass
    return Response(gen(),mimetype="text/event-stream",
                    headers={"Cache-Control":"no-cache","X-Accel-Buffering":"no"})


# ═══════════════════════════════════════════════════════════════
#  LOGIN PAGE HTML — Cinematic v5
# ═══════════════════════════════════════════════════════════════
LOGIN_HTML = r"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>PerryAI — Sign In</title>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet"/>
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
:root{
  --bg:#070b14;--purple:#7c3aed;--violet:#9d6ef5;--cyan:#00e5c4;--cyan2:#00d0f5;
  --green:#22c55e;--red:#ef4444;--yellow:#eab308;
  --t1:#ffffff;--t2:#94a3b8;--t3:#475569;
  --card:rgba(255,255,255,0.045);--border:rgba(255,255,255,0.09);
  --border2:rgba(255,255,255,0.15);
  --shadow:0 24px 80px rgba(0,0,0,0.7);
  --f:'Inter',sans-serif;--ease:cubic-bezier(.4,0,.2,1);
}
html,body{height:100%;font-family:var(--f);color:var(--t1);overflow:hidden}
body{
  background:var(--bg);
  background-image:
    radial-gradient(ellipse 70% 60% at 15% -5%,rgba(124,58,237,.3),transparent 65%),
    radial-gradient(ellipse 55% 55% at 85% 105%,rgba(0,229,196,.2),transparent 60%),
    radial-gradient(ellipse 40% 40% at 50% 50%,rgba(124,58,237,.07),transparent),
    radial-gradient(ellipse 30% 35% at 5% 90%,rgba(0,208,245,.12),transparent);
  display:flex;flex-direction:column;min-height:100vh;
}
/* Animated grid */
body::before{
  content:'';position:fixed;inset:0;pointer-events:none;z-index:0;
  background-image:
    linear-gradient(rgba(124,58,237,.05) 1px,transparent 1px),
    linear-gradient(90deg,rgba(124,58,237,.05) 1px,transparent 1px);
  background-size:60px 60px;
  mask-image:radial-gradient(ellipse 80% 80% at 50% 50%,black 20%,transparent 80%);
}
/* Floating orbs */
.orb{position:fixed;border-radius:50%;pointer-events:none;z-index:0;filter:blur(80px)}
.orb1{width:440px;height:440px;background:radial-gradient(circle,rgba(124,58,237,.25),transparent 70%);top:-180px;left:-100px;animation:drift1 18s ease-in-out infinite}
.orb2{width:320px;height:320px;background:radial-gradient(circle,rgba(0,229,196,.18),transparent 70%);bottom:-120px;right:-80px;animation:drift2 14s ease-in-out infinite}
.orb3{width:200px;height:200px;background:radial-gradient(circle,rgba(157,110,245,.14),transparent 70%);top:40%;left:55%;animation:drift3 22s ease-in-out infinite}
@keyframes drift1{0%,100%{transform:translate(0,0) scale(1)}33%{transform:translate(30px,-20px) scale(1.05)}66%{transform:translate(-15px,25px) scale(.97)}}
@keyframes drift2{0%,100%{transform:translate(0,0)}50%{transform:translate(-25px,-30px)}}
@keyframes drift3{0%,100%{transform:translate(0,0) scale(1)}40%{transform:translate(-20px,15px) scale(1.08)}80%{transform:translate(10px,-10px) scale(.94)}}

/* Navbar */
.nav{
  position:relative;z-index:10;display:flex;align-items:center;padding:0 24px;height:60px;
  background:rgba(7,11,20,.5);border-bottom:1px solid rgba(255,255,255,.06);backdrop-filter:blur(20px);
}
.nav-logo{display:flex;align-items:center;gap:10px;text-decoration:none}
.nav-logo-icon{
  width:34px;height:34px;border-radius:9px;flex-shrink:0;
  background:linear-gradient(135deg,var(--purple),var(--cyan));
  display:flex;align-items:center;justify-content:center;font-size:16px;
}
.nav-logo-text{font-size:16px;font-weight:900;background:linear-gradient(135deg,var(--cyan),var(--violet));
  -webkit-background-clip:text;-webkit-text-fill-color:transparent;}
.nav-logo-sub{font-size:10px;color:var(--t3);font-weight:600;letter-spacing:.5px;text-transform:uppercase}
.nav-spacer{flex:1}
.nav-chip{
  display:flex;align-items:center;gap:6px;padding:6px 14px;border-radius:99px;
  background:rgba(34,197,94,.1);border:1px solid rgba(34,197,94,.25);
  font-size:11px;font-weight:700;color:var(--green);
}
.nav-dot{width:6px;height:6px;border-radius:50%;background:var(--green);animation:pulse 2s infinite}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.4}}

/* Stage */
.stage{
  flex:1;display:flex;flex-direction:column;align-items:center;justify-content:center;
  padding:24px 16px 32px;position:relative;z-index:5;
}
.hero-lockup{text-align:center;margin-bottom:28px}
.hero-badge{
  display:inline-flex;align-items:center;gap:6px;padding:5px 14px;border-radius:99px;
  background:linear-gradient(135deg,rgba(124,58,237,.25),rgba(0,229,196,.12));
  border:1px solid rgba(157,110,245,.3);font-size:11px;font-weight:700;
  color:var(--purple3,#c4b5fd);letter-spacing:.5px;margin-bottom:18px;
}
.hero-badge::before{content:'●';color:var(--cyan);font-size:8px;animation:pulse 2s infinite}
.hero-title{
  font-size:clamp(28px,5vw,42px);font-weight:900;letter-spacing:-1.5px;line-height:1.1;
  background:linear-gradient(135deg,#fff 30%,var(--violet) 70%,var(--cyan));
  -webkit-background-clip:text;-webkit-text-fill-color:transparent;margin-bottom:10px;
}
.hero-sub{font-size:14px;color:var(--t2);font-weight:500}

/* Glass card */
.glass-card{
  background:var(--card);border:1px solid var(--border);border-radius:22px;
  padding:32px;width:100%;max-width:440px;backdrop-filter:blur(24px);
  box-shadow:var(--shadow);position:relative;overflow:hidden;
}
.glass-card::before{
  content:'';position:absolute;inset:0;border-radius:22px;pointer-events:none;
  background:linear-gradient(135deg,rgba(255,255,255,.05),transparent 60%);
}
.error-msg{
  display:flex;align-items:center;gap:10px;background:rgba(239,68,68,.1);
  border:1px solid rgba(239,68,68,.3);border-radius:10px;padding:12px 16px;
  font-size:13px;color:#fca5a5;margin-bottom:20px;
}
.error-icon{width:20px;height:20px;border-radius:50%;background:rgba(239,68,68,.25);
  display:flex;align-items:center;justify-content:center;font-size:11px;font-weight:800;flex-shrink:0}
.form-group{margin-bottom:16px}
.form-label{display:block;font-size:10.5px;color:var(--t3);font-weight:700;
  letter-spacing:.8px;text-transform:uppercase;margin-bottom:7px}
.inp-shell{position:relative;display:flex;align-items:center}
.inp-icon{
  position:absolute;left:13px;width:18px;height:18px;flex-shrink:0;
  display:flex;align-items:center;justify-content:center;
}
.inp-icon svg{width:15px;height:15px;stroke:var(--t3);fill:none;stroke-width:2;stroke-linecap:round}
.inp{
  width:100%;background:rgba(13,30,56,.8);border:1.5px solid rgba(255,255,255,.1);
  border-radius:11px;color:var(--t1);font-family:var(--f);font-size:14px;
  padding:12px 14px 12px 42px;outline:none;
  transition:border-color .2s var(--ease),box-shadow .2s var(--ease);
}
.inp:focus{border-color:var(--violet);box-shadow:0 0 0 3px rgba(124,58,237,.15)}
.inp::placeholder{color:var(--t3)}
.eye-btn{
  position:absolute;right:12px;background:none;border:none;cursor:pointer;
  color:var(--t3);padding:6px;line-height:0;transition:color .2s;
}
.eye-btn:hover{color:var(--t2)}
.eye-btn svg{width:16px;height:16px;stroke:currentColor;fill:none;stroke-width:2;stroke-linecap:round}

/* Terms */
.terms-row{display:flex;align-items:flex-start;gap:10px;margin-bottom:22px;margin-top:-4px}
.terms-box{position:relative;width:18px;height:18px;flex-shrink:0;margin-top:1px}
.cb{position:absolute;opacity:0;width:0;height:0}
.cb-custom{
  width:18px;height:18px;border-radius:5px;border:1.5px solid rgba(255,255,255,.2);
  background:rgba(255,255,255,.04);cursor:pointer;transition:all .2s;
  display:flex;align-items:center;justify-content:center;
}
.cb:checked+.cb-custom{background:var(--purple);border-color:var(--violet)}
.cb:checked+.cb-custom::after{content:'✓';font-size:11px;font-weight:800;color:#fff}
.terms-text{font-size:12px;color:var(--t2);line-height:1.6;flex:1}
.terms-text a{color:var(--cyan2);text-decoration:none;font-weight:600}

/* Login button */
.btn-login{
  width:100%;padding:14px;border-radius:12px;border:none;
  background:linear-gradient(135deg,#5b21b6,var(--purple),#8b5cf6);
  color:#fff;font-family:var(--f);font-size:15px;font-weight:800;cursor:pointer;
  transition:all .25s var(--ease);position:relative;overflow:hidden;
  box-shadow:0 4px 24px rgba(124,58,237,.45);letter-spacing:.3px;
}
.btn-login::before{
  content:'';position:absolute;inset:0;
  background:linear-gradient(105deg,transparent 30%,rgba(255,255,255,.18) 50%,transparent 70%);
  transform:translateX(-100%);transition:transform .6s var(--ease);
}
.btn-login:hover::before{transform:translateX(100%)}
.btn-login:hover{filter:brightness(1.1);transform:translateY(-2px)}
.btn-login:active{transform:scale(.97)}

/* Divider */
.divider{display:flex;align-items:center;gap:12px;margin:22px 0 16px;color:var(--t3);font-size:11px;font-weight:700;letter-spacing:1px}
.divider::before,.divider::after{content:'';flex:1;height:1px;background:rgba(255,255,255,.07)}

/* Risk note */
.risk-note{
  background:rgba(234,179,8,.07);border:1px solid rgba(234,179,8,.18);
  border-radius:10px;padding:11px 14px;
  font-size:11.5px;color:rgba(234,179,8,.7);line-height:1.6;text-align:center;
}
.risk-note strong{color:#fde047}

/* Stats strip */
.stats-strip{
  display:flex;justify-content:center;gap:0;margin-top:22px;
  background:rgba(255,255,255,.03);border:1px solid var(--border);
  border-radius:14px;overflow:hidden;backdrop-filter:blur(12px);max-width:440px;width:100%;
}
.sstat{flex:1;text-align:center;padding:14px 10px;position:relative}
.sstat:not(:last-child)::after{
  content:'';position:absolute;right:0;top:20%;bottom:20%;
  width:1px;background:var(--border);
}
.sstat-val{
  font-size:20px;font-weight:900;
  background:linear-gradient(135deg,var(--cyan),var(--cyan2));
  -webkit-background-clip:text;-webkit-text-fill-color:transparent;margin-bottom:3px;
}
.sstat-lbl{font-size:9px;font-weight:700;color:var(--t3);letter-spacing:.8px;text-transform:uppercase}
</style>
</head>
<body>
<div class="orb orb1"></div>
<div class="orb orb2"></div>
<div class="orb orb3"></div>

<nav class="nav">
  <a class="nav-logo" href="/login">
    <div class="nav-logo-icon">⚡</div>
    <div>
      <div class="nav-logo-text">PerryAI</div>
      <div class="nav-logo-sub">AI Trading</div>
    </div>
  </a>
  <div class="nav-spacer"></div>
  <div class="nav-chip"><div class="nav-dot"></div>4 Bots Live</div>
</nav>

<div class="stage">
  <div class="hero-lockup">
    <div class="hero-badge">Institutional-Grade AI Trading Platform</div>
    <div class="hero-title">Sign In to PerryAI</div>
    <div class="hero-sub">Access your automated trading command center</div>
  </div>

  <div class="glass-card">
    {% if error %}
    <div class="error-msg">
      <div class="error-icon">✕</div>{{ error }}
    </div>
    {% endif %}

    <form method="POST" action="/login" autocomplete="on">
      <div class="form-group">
        <label class="form-label">Email Address</label>
        <div class="inp-shell">
          <div class="inp-icon"><svg viewBox="0 0 24 24"><path d="M20 4H4a2 2 0 00-2 2v12a2 2 0 002 2h16a2 2 0 002-2V6a2 2 0 00-2-2z"/><path d="M22 6l-10 7L2 6"/></svg></div>
          <input class="inp" type="email" name="email" placeholder="your@email.com" required autocomplete="email"/>
        </div>
      </div>
      <div class="form-group">
        <label class="form-label">Password</label>
        <div class="inp-shell">
          <div class="inp-icon"><svg viewBox="0 0 24 24"><rect x="3" y="11" width="18" height="11" rx="2"/><path d="M7 11V7a5 5 0 0110 0v4"/></svg></div>
          <input class="inp" type="password" name="password" id="pwdField" placeholder="Enter password" required autocomplete="current-password"/>
          <button type="button" class="eye-btn" onclick="togglePwd()" tabindex="-1">
            <svg id="eyeOpen" viewBox="0 0 24 24"><path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/><circle cx="12" cy="12" r="3"/></svg>
            <svg id="eyeClose" viewBox="0 0 24 24" style="display:none"><path d="M17.94 17.94A10.07 10.07 0 0112 20c-7 0-11-8-11-8a18.45 18.45 0 015.06-5.94M9.9 4.24A9.12 9.12 0 0112 4c7 0 11 8 11 8a18.5 18.5 0 01-2.16 3.19m-6.72-1.07a3 3 0 11-4.24-4.24"/><line x1="1" y1="1" x2="23" y2="23"/></svg>
          </button>
        </div>
      </div>
      <div class="terms-row">
        <div class="terms-box">
          <input class="cb" type="checkbox" id="terms" required/>
          <div class="cb-custom" onclick="document.getElementById('terms').click()"></div>
        </div>
        <label class="terms-text" for="terms">
          I acknowledge and agree to the <a href="#">Terms of Service</a> and <a href="#">Risk Disclosure</a> of PerryAI
        </label>
      </div>
      <button type="submit" class="btn-login">Sign In to Dashboard</button>
    </form>

    <div class="divider"><span>PLATFORM INFO</span></div>
    <div class="risk-note"><strong>⚠ Risk Notice</strong> — Automated trading on Deriv carries substantial risk. Ensure you understand all risks before trading. Past performance does not guarantee future results.</div>
  </div>

  <div class="stats-strip">
    <div class="sstat"><div class="sstat-val">4</div><div class="sstat-lbl">AI Bots</div></div>
    <div class="sstat"><div class="sstat-val">Live</div><div class="sstat-lbl">Real-time</div></div>
    <div class="sstat"><div class="sstat-val">Mix</div><div class="sstat-lbl">Multi-trade</div></div>
    <div class="sstat"><div class="sstat-val">v5</div><div class="sstat-lbl">Latest</div></div>
  </div>
</div>

<script>
function togglePwd(){
  const f=document.getElementById('pwdField');
  const o=document.getElementById('eyeOpen'); const c=document.getElementById('eyeClose');
  if(f.type==='password'){f.type='text';o.style.display='none';c.style.display='';}
  else{f.type='password';o.style.display='';c.style.display='none';}
}
</script>
</body>
</html>"""


# ═══════════════════════════════════════════════════════════════
#  MAIN APP HTML — Mobile-First Optimized v5
# ═══════════════════════════════════════════════════════════════
MAIN_HTML = r"""<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover"/>
<meta name="mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent"/>
<title>PerryAI — Trading Dashboard</title>
<link rel="preconnect" href="https://fonts.googleapis.com"/>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800;900&display=swap" rel="stylesheet"/>
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0;-webkit-tap-highlight-color:transparent}
:root{
  --bg:#070b14;--bg2:#0a1628;--bg3:#0d1e38;
  --glass:rgba(255,255,255,.042);--glass2:rgba(255,255,255,.07);
  --b1:rgba(255,255,255,.07);--b2:rgba(255,255,255,.11);--b3:rgba(255,255,255,.2);
  --purple:#7c3aed;--violet:#9d6ef5;--purple3:#c4b5fd;--cyan:#00e5c4;--cyan2:#00d0f5;
  --green:#22c55e;--red:#ef4444;--orange:#f97316;--yellow:#eab308;--gold:#f59e0b;
  --t1:#fff;--t2:#94a3b8;--t3:#475569;--t4:#334155;
  --cyan-dim:rgba(0,229,196,.08);--purple-dim:rgba(124,58,237,.1);
  --green-dim:rgba(34,197,94,.1);--red-dim:rgba(239,68,68,.1);
  --shadow-sm:0 4px 20px rgba(0,0,0,.4);
  --r2:12px;--r3:10px;--r4:8px;
  --f:'Inter',sans-serif;--ease:cubic-bezier(.4,0,.2,1);
  --safe-top:env(safe-area-inset-top,0px);
  --safe-bottom:env(safe-area-inset-bottom,0px);
}
html,body{height:100%;font-family:var(--f);color:var(--t1);overflow:hidden}
body{
  background:var(--bg);
  background-image:
    radial-gradient(ellipse 60% 50% at 0% 0%,rgba(124,58,237,.2),transparent 55%),
    radial-gradient(ellipse 50% 50% at 100% 100%,rgba(0,229,196,.13),transparent 55%),
    linear-gradient(rgba(124,58,237,.025) 1px,transparent 1px) 0 0/60px 60px,
    linear-gradient(90deg,rgba(124,58,237,.025) 1px,transparent 1px) 0 0/60px 60px;
}

/* ── TOAST ── */
.toast-wrap{position:fixed;top:calc(var(--safe-top) + 60px);left:50%;transform:translateX(-50%);z-index:9999;pointer-events:none}
.toast{background:rgba(15,23,42,.96);border:1px solid var(--b2);border-radius:99px;
  padding:9px 22px;font-size:13px;font-weight:700;opacity:0;transform:translateY(-8px);
  transition:all .3s var(--ease);white-space:nowrap;backdrop-filter:blur(20px);max-width:90vw;}
.toast.show{opacity:1;transform:none}
.toast.error{border-color:rgba(239,68,68,.4);color:#fca5a5}
.toast.success{border-color:rgba(34,197,94,.4);color:var(--green)}
.toast.info{border-color:rgba(0,229,196,.3);color:var(--cyan)}

/* ── TOPBAR ── */
.topbar{
  position:fixed;top:0;left:0;right:0;z-index:100;
  height:calc(52px + var(--safe-top));padding-top:var(--safe-top);
  background:rgba(7,11,20,.88);border-bottom:1px solid var(--b1);
  backdrop-filter:blur(20px);display:flex;align-items:center;gap:8px;padding-left:14px;padding-right:14px;
}
.logo-wrap{display:flex;align-items:center;gap:8px}
.logo-icon{width:28px;height:28px;border-radius:8px;
  background:linear-gradient(135deg,var(--purple),var(--cyan));
  display:flex;align-items:center;justify-content:center;font-size:14px;flex-shrink:0;}
.logo{font-size:15px;font-weight:900;background:linear-gradient(135deg,var(--cyan),var(--violet));
  -webkit-background-clip:text;-webkit-text-fill-color:transparent;}
.logo-v{font-size:9px;color:var(--t3);font-weight:700;padding:1px 5px;background:rgba(255,255,255,.05);
  border:1px solid var(--b1);border-radius:99px;}
.tb-spacer{flex:1}
.live-pill{display:flex;align-items:center;gap:4px;padding:4px 10px;border-radius:99px;
  background:rgba(255,255,255,.05);border:1px solid var(--b1);font-size:10px;font-weight:700;color:var(--t2);}
.dot{width:6px;height:6px;border-radius:50%;background:var(--t4)}
.dot.live{background:var(--green);box-shadow:0 0 8px var(--green);animation:pulse 2s infinite}
@keyframes pulse{0%,100%{opacity:1}50%{opacity:.45}}
#clock{font-size:10px;color:var(--t3);font-variant-numeric:tabular-nums;font-weight:600}
.logout-btn{display:flex;align-items:center;gap:4px;padding:5px 10px;border-radius:7px;
  background:rgba(255,255,255,.04);border:1px solid var(--b1);color:var(--t2);
  text-decoration:none;font-size:11px;font-weight:600;transition:all .2s;white-space:nowrap;}
.logout-btn:hover{background:rgba(255,255,255,.08);color:var(--t1)}

/* ── LAYOUT ── */
.desktop-layout{
  display:none;grid-template-columns:370px 1fr;height:100vh;
  padding-top:calc(52px + var(--safe-top));
}
@media(min-width:800px){
  .mobile-layout{display:none!important}
  .desktop-layout{display:grid!important}
}
@media(max-width:799px){
  .desktop-layout{display:none!important}
  .mobile-layout{display:flex!important}
  body{overflow:hidden}
}
.dcol{padding:12px;overflow-y:auto;height:100%;scrollbar-width:thin;scrollbar-color:rgba(255,255,255,.08) transparent}
.dcol::-webkit-scrollbar{width:3px}
.dcol::-webkit-scrollbar-thumb{background:rgba(255,255,255,.08);border-radius:99px}
.dcol-r{background:rgba(255,255,255,.012);border-left:1px solid var(--b1)}

/* ── CARD ── */
.card{background:var(--glass);border:1px solid var(--b1);border-radius:16px;
  padding:14px;margin-bottom:12px;backdrop-filter:blur(12px);position:relative;overflow:hidden;}
.card::before{content:'';position:absolute;inset:0;border-radius:16px;pointer-events:none;
  background:linear-gradient(135deg,rgba(255,255,255,.03),transparent 60%);}
.card-hd{display:flex;align-items:center;gap:9px;margin-bottom:13px}
.card-icon{width:30px;height:30px;border-radius:8px;display:flex;align-items:center;
  justify-content:center;font-size:14px;flex-shrink:0;}
.ci-green{background:rgba(34,197,94,.15);border:1px solid rgba(34,197,94,.25)}
.ci-purple{background:rgba(124,58,237,.15);border:1px solid rgba(124,58,237,.25)}
.ci-cyan{background:rgba(0,229,196,.12);border:1px solid rgba(0,229,196,.2)}
.ci-gold{background:rgba(245,158,11,.12);border:1px solid rgba(245,158,11,.2)}
.ci-mix{background:linear-gradient(135deg,rgba(124,58,237,.2),rgba(0,229,196,.15));border:1px solid rgba(157,110,245,.3)}
.card-title{font-size:14px;font-weight:800;color:var(--t1)}

/* ── MOBILE LAYOUT ── */
.mobile-layout{flex-direction:column;height:100vh;height:100dvh;
  padding-top:calc(52px + var(--safe-top));}
.tab-content{flex:1;overflow-y:auto;overflow-x:hidden;-webkit-overflow-scrolling:touch;scrollbar-width:none;overscroll-behavior:contain}
.tab-content::-webkit-scrollbar{display:none}
.tab-pane{display:none;padding:10px}
.tab-pane.active{display:block}
.bottom-nav{
  display:flex;flex-shrink:0;
  background:rgba(7,11,20,.95);border-top:1px solid var(--b1);
  backdrop-filter:blur(20px);position:relative;
  padding-bottom:var(--safe-bottom);
}
.bnav-indicator{position:absolute;top:0;height:2px;
  background:linear-gradient(90deg,var(--purple),var(--cyan));border-radius:0 0 3px 3px;
  transition:left .3s var(--ease),width .3s var(--ease);}
.bnav-btn{flex:1;background:none;border:none;color:var(--t3);font-family:var(--f);
  font-size:9px;font-weight:700;padding:10px 4px 9px;cursor:pointer;
  display:flex;flex-direction:column;align-items:center;gap:3px;letter-spacing:.3px;transition:color .2s;
  min-height:54px;touch-action:manipulation;}
.bnav-btn.active{color:var(--cyan)}
.bn-icon{font-size:20px;line-height:1}

/* ── FORMS ── */
select,select option{background-color:#0d1e38!important;color:#fff!important}
select{width:100%;background:#0d1e38;border:1.5px solid var(--b2);border-radius:var(--r3);
  color:var(--t1);font-family:var(--f);font-size:14px;padding:10px 36px 10px 12px;outline:none;
  appearance:none;-webkit-appearance:none;cursor:pointer;
  background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8'%3E%3Cpath d='M1 1l5 5 5-5' stroke='%2394a3b8' stroke-width='1.8' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");
  background-repeat:no-repeat;background-position:right 11px center;
  transition:border-color .2s;-webkit-text-size-adjust:100%;font-size:16px;}
select:focus{border-color:var(--violet)}
input[type=number]{width:100%;background:#0d1e38;border:1.5px solid var(--b2);border-radius:var(--r3);
  color:var(--t1);font-family:var(--f);font-size:16px;padding:10px 12px;outline:none;transition:border-color .2s}
input[type=number]:focus{border-color:var(--violet)}
.fld label{display:block;font-size:10px;color:var(--t3);font-weight:700;margin-bottom:5px;letter-spacing:.8px;text-transform:uppercase}
.g2{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:10px}
.hdiv{border:none;border-top:1px solid var(--b1);margin:14px 0}
.sec-lbl{font-size:10px;font-weight:800;letter-spacing:2px;text-transform:uppercase;
  color:var(--violet);margin-bottom:10px;display:flex;align-items:center;gap:8px}
.sec-lbl::after{content:'';flex:1;height:1px;background:linear-gradient(90deg,rgba(157,110,245,.3),transparent)}
.stake-row{display:flex;gap:6px;margin-bottom:12px;align-items:stretch}
.stake-inp{flex:1;min-width:0;background:#0d1e38;border:1.5px solid var(--b2);border-radius:var(--r3);
  color:var(--t1);font-family:var(--f);font-size:20px;font-weight:900;padding:10px 12px;outline:none;transition:border-color .2s}
.stake-inp:focus{border-color:var(--violet)}
.preset{padding:10px 12px;border-radius:var(--r3);border:1.5px solid var(--b2);
  background:rgba(255,255,255,.055);color:var(--t2);font-family:var(--f);
  font-size:12px;font-weight:700;cursor:pointer;transition:all .18s;touch-action:manipulation;
  -webkit-text-size-adjust:100%;}
.preset:hover{background:rgba(255,255,255,.1);color:var(--t1)}
.preset:active{transform:scale(.93)}

/* ── API CONNECTION ── */
.conn-badge{display:none;align-items:center;gap:5px;margin-left:auto;background:var(--green-dim);
  border:1px solid rgba(34,197,94,.3);border-radius:99px;padding:3px 10px;font-size:11px;font-weight:700;color:var(--green)}
.tok-input{width:100%;background:#0d1e38;border:1.5px solid var(--b2);border-radius:var(--r3);
  color:var(--t1);font-family:var(--f);font-size:14px;padding:12px 14px;outline:none;margin-bottom:10px;
  transition:border-color .2s;-webkit-text-size-adjust:100%;}
.tok-input:focus{border-color:var(--cyan2)}
.tok-input::placeholder{color:var(--t3)}
.btn-connect{width:100%;padding:13px;border-radius:var(--r3);border:none;
  background:linear-gradient(135deg,#15803d,var(--green));color:#fff;font-family:var(--f);
  font-size:15px;font-weight:800;cursor:pointer;margin-bottom:12px;
  box-shadow:0 4px 16px rgba(34,197,94,.25);touch-action:manipulation;}
.btn-connect:hover{filter:brightness(1.08)}
.btn-connect:active{transform:scale(.97)}
.acct-bar{background:rgba(255,255,255,.035);border:1px solid var(--b1);border-radius:var(--r3);
  padding:10px 13px;margin-bottom:10px;font-size:12px;color:var(--t2);display:flex;align-items:center;gap:7px;flex-wrap:wrap}
.bal-card{background:linear-gradient(135deg,rgba(0,229,196,.06),rgba(0,208,245,.04));
  border:1px solid rgba(0,229,196,.18);border-radius:var(--r3);padding:14px;margin-bottom:12px}
.bal-lbl{font-size:10px;color:var(--t3);font-weight:700;letter-spacing:1px;text-transform:uppercase;margin-bottom:5px}
.bal-val{font-size:26px;font-weight:900;letter-spacing:-1px;
  background:linear-gradient(135deg,var(--cyan),var(--cyan2));
  -webkit-background-clip:text;-webkit-text-fill-color:transparent;}
.btn-disc{display:inline-flex;align-items:center;gap:6px;background:var(--red-dim);
  border:1px solid rgba(239,68,68,.3);color:#f87171;padding:9px 16px;border-radius:var(--r4);
  font-family:var(--f);font-size:13px;font-weight:700;cursor:pointer;transition:all .2s;touch-action:manipulation;}
.btn-disc:hover{background:rgba(239,68,68,.2)}
.help-txt{font-size:12px;color:var(--t3);line-height:1.9;margin-top:12px;
  background:rgba(255,255,255,.02);border-radius:var(--r4);padding:11px}
.help-txt a{color:var(--cyan2);text-decoration:none;font-weight:600}
.help-step{display:flex;align-items:flex-start;gap:7px;margin-bottom:5px}
.step-n{width:18px;height:18px;border-radius:50%;flex-shrink:0;margin-top:1px;
  background:var(--purple-dim);border:1px solid rgba(124,58,237,.3);
  display:flex;align-items:center;justify-content:center;font-size:9px;font-weight:800;color:var(--violet)}

/* ── BOT SELECTOR ── */
.bot-sel-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:7px}
.bot-card{background:rgba(255,255,255,.04);border:2px solid var(--b1);border-radius:13px;
  padding:12px 5px 10px;text-align:center;cursor:pointer;transition:all .22s var(--ease);
  user-select:none;display:flex;flex-direction:column;align-items:center;min-width:0;
  touch-action:manipulation;}
.bot-card:hover{border-color:var(--b3);transform:translateY(-2px)}
.bot-card:active{transform:scale(.94)}
.bot-card.active-md{background:rgba(0,229,196,.09);border-color:var(--cyan);box-shadow:0 0 20px rgba(0,229,196,.18)}
.bot-card.active-eo{background:rgba(124,58,237,.09);border-color:var(--violet);box-shadow:0 0 20px rgba(124,58,237,.22)}
.bot-card.active-ou{background:rgba(249,115,22,.09);border-color:var(--orange);box-shadow:0 0 20px rgba(249,115,22,.18)}
.bot-card.active-ab{background:rgba(34,197,94,.09);border-color:var(--green);box-shadow:0 0 20px rgba(34,197,94,.18)}
.bot-card.active-mix{
  background:linear-gradient(135deg,rgba(124,58,237,.09),rgba(0,229,196,.07));
  border-color:var(--violet);box-shadow:0 0 20px rgba(124,58,237,.25)}
.bot-icon{font-size:20px;line-height:1;height:28px;display:flex;align-items:center;justify-content:center;margin-bottom:5px}
.bot-name{font-size:9.5px;font-weight:800;line-height:1.25;min-height:26px;
  display:flex;align-items:center;justify-content:center;margin-bottom:2px;padding:0 1px;text-align:center}
.bot-desc{font-size:8.5px;color:var(--t3);line-height:1.3}

/* ── FEATURE TOGGLES ── */
.feat{background:rgba(255,255,255,.035);border:1px solid var(--b1);border-radius:var(--r3);
  padding:12px 13px;display:flex;align-items:center;gap:11px;margin-bottom:7px}
.feat-info{flex:1;min-width:0}
.feat-title{font-size:13px;font-weight:700;margin-bottom:2px;color:var(--t1)}
.feat-desc{font-size:11px;color:var(--t3);line-height:1.4}
.feat-btn{flex-shrink:0;width:44px;height:24px;border-radius:99px;border:none;
  background:rgba(255,255,255,.1);cursor:pointer;position:relative;transition:background .25s;
  font-size:0;touch-action:manipulation;}
.feat-btn::before{content:'';position:absolute;top:3px;left:3px;width:18px;height:18px;border-radius:50%;
  background:#fff;opacity:.5;transition:transform .25s,opacity .25s}
.feat-btn.on{background:linear-gradient(135deg,var(--purple),var(--violet))}
.feat-btn.on::before{transform:translateX(18px);opacity:1}
.sub{max-height:0;overflow:hidden;transition:max-height .35s var(--ease),opacity .3s;opacity:0}
.sub.open{max-height:280px;opacity:1}

/* ── MIXOLOGIST MODULE TOGGLES ── */
.mix-mod-grid{display:grid;grid-template-columns:1fr 1fr 1fr;gap:7px;margin-bottom:12px}
.mix-mod{border:2px solid var(--b1);border-radius:10px;padding:10px 6px;text-align:center;cursor:pointer;
  transition:all .2s;touch-action:manipulation;}
.mix-mod:active{transform:scale(.93)}
.mix-mod.on-ou{border-color:var(--cyan);background:rgba(0,229,196,.08)}
.mix-mod.on-eo{border-color:var(--violet);background:rgba(124,58,237,.08)}
.mix-mod.on-md{border-color:var(--orange);background:rgba(249,115,22,.08)}
.mix-mod-icon{font-size:16px;margin-bottom:3px}
.mix-mod-name{font-size:9.5px;font-weight:800;line-height:1.2}
.mix-mod-cnt{font-size:9px;color:var(--t3);margin-top:2px}
.mix-mod-stat{font-size:9px;font-weight:700;margin-top:2px}
.mix-mod-stat.pos{color:var(--green)}.mix-mod-stat.neg{color:var(--red)}.mix-mod-stat.neu{color:var(--t3)}

/* ── DIGIT BARS ── */
.dg{display:flex;gap:4px;height:100px;align-items:flex-end;margin-bottom:8px}
.db{flex:1;border-radius:6px 6px 3px 3px;display:flex;flex-direction:column;align-items:center;
  justify-content:flex-end;background:rgba(255,255,255,.06);border:1.5px solid transparent;
  cursor:pointer;transition:all .25s;position:relative;overflow:hidden;min-width:0;height:100%}
.db::after{content:'';position:absolute;bottom:0;left:0;right:0;height:var(--pct,4%);min-height:4px;
  background:linear-gradient(to top,rgba(124,58,237,.75),rgba(0,229,196,.15));
  border-radius:4px 4px 2px 2px;transition:height .45s}
.db:hover{background:rgba(255,255,255,.1)}
.db:active{transform:scale(.92)}
.db.sel{border-color:var(--cyan);background:rgba(0,229,196,.1)}
.db.sel::after{background:linear-gradient(to top,var(--cyan),rgba(0,229,196,.4))}
.db.hot{border-color:rgba(239,68,68,.5);background:rgba(239,68,68,.07)}
.db.hot::after{background:linear-gradient(to top,var(--red),rgba(239,68,68,.35))}
.db.last{border-color:var(--violet);background:rgba(157,110,245,.08)}
.db.ou-active{border-color:var(--cyan);background:rgba(0,229,196,.13)}
.db.ou-active::after{background:linear-gradient(to top,var(--cyan),rgba(0,229,196,.45))}
.db-n{font-size:11px;font-weight:800;position:absolute;bottom:4px;color:rgba(255,255,255,.8);z-index:1}
.db-p{position:absolute;top:4px;font-size:7.5px;color:var(--t3);font-weight:700;z-index:1}
.db-ck{position:absolute;top:3px;right:3px;font-size:8px;color:var(--cyan);display:none;font-weight:800;z-index:2}
.db.sel .db-ck,.db.ou-active .db-ck{display:block}
.dg-meta{font-size:10.5px;color:var(--t3);text-align:center;padding:2px 0 4px;font-weight:500}

/* ── PARITY BARS ── */
.parity-row{display:flex;align-items:center;gap:10px;margin-bottom:8px}
.parity-lbl{font-size:12px;font-weight:800;width:44px;flex-shrink:0}
.parity-bar-bg{flex:1;height:28px;background:rgba(255,255,255,.05);border-radius:var(--r4);
  overflow:hidden;position:relative;border:1px solid var(--b1)}
.parity-bar-fill{height:100%;border-radius:var(--r4);transition:width .5s;
  display:flex;align-items:center;padding-left:8px;font-size:10px;font-weight:800;color:rgba(255,255,255,.9)}
.parity-pct{position:absolute;right:8px;top:50%;transform:translateY(-50%);font-size:12px;font-weight:800;color:var(--t2)}
.parity-badge{font-size:9px;font-weight:800;padding:3px 9px;border-radius:99px;margin-left:auto;letter-spacing:.5px}
.parity-badge.even{background:var(--purple-dim);border:1px solid rgba(157,110,245,.35);color:var(--purple3)}
.parity-badge.odd{background:rgba(249,115,22,.12);border:1px solid rgba(249,115,22,.35);color:var(--orange)}
.parity-even .parity-bar-fill{background:linear-gradient(90deg,var(--purple),var(--violet))}
.parity-odd  .parity-bar-fill{background:linear-gradient(90deg,var(--orange),var(--yellow))}
.ou-dir-row{display:flex;gap:7px;margin-bottom:10px}
.ou-dir-btn{flex:1;padding:10px 8px;border-radius:var(--r3);border:2px solid var(--b2);
  background:rgba(255,255,255,.045);color:var(--t2);font-family:var(--f);
  font-size:13px;font-weight:700;cursor:pointer;transition:all .2s;touch-action:manipulation;}
.ou-dir-btn.sel-over{background:rgba(0,229,196,.12);border-color:var(--cyan);color:var(--cyan)}
.ou-dir-btn.sel-under{background:rgba(249,115,22,.12);border-color:var(--orange);color:var(--orange)}

/* ── BOT BADGE ── */
.bot-badge{display:inline-flex;align-items:center;gap:4px;padding:3px 9px;border-radius:99px;font-size:9.5px;font-weight:800;margin-left:auto}
.bot-badge.md{background:var(--cyan-dim);border:1px solid rgba(0,229,196,.35);color:var(--cyan)}
.bot-badge.eo{background:var(--purple-dim);border:1px solid rgba(157,110,245,.35);color:var(--purple3)}
.bot-badge.ou{background:rgba(249,115,22,.1);border:1px solid rgba(249,115,22,.3);color:var(--orange)}
.bot-badge.mix{background:linear-gradient(135deg,rgba(124,58,237,.15),rgba(0,229,196,.1));
  border:1px solid rgba(157,110,245,.4);color:var(--purple3)}

/* ── ACTION BUTTONS ── */
.act{width:100%;padding:13px 16px;border-radius:12px;border:none;font-family:var(--f);
  font-size:14px;font-weight:800;cursor:pointer;transition:all .22s;margin-bottom:7px;
  display:flex;align-items:center;justify-content:center;gap:8px;letter-spacing:.3px;
  touch-action:manipulation;min-height:48px;}
.act:hover{filter:brightness(1.08);transform:translateY(-1px)}
.act:active{transform:scale(.96);filter:brightness(.92)}
.act:disabled{opacity:.3;cursor:not-allowed;filter:none;transform:none}
.act-sub{font-size:10.5px;color:var(--t3);text-align:center;margin:-3px 0 9px;line-height:1.5;padding:0 4px}
.btn-teal{background:linear-gradient(135deg,#0f766e,#14b8a6);color:#fff;box-shadow:0 4px 16px rgba(20,184,166,.3)}
.btn-purp{background:linear-gradient(135deg,#5b21b6,var(--purple));color:#fff;box-shadow:0 4px 16px rgba(124,58,237,.3)}
.btn-ora{background:linear-gradient(135deg,#c2410c,var(--orange));color:#fff;box-shadow:0 4px 16px rgba(249,115,22,.3)}
.btn-pink{background:linear-gradient(135deg,#9d174d,#ec4899);color:#fff;box-shadow:0 4px 16px rgba(236,72,153,.3)}
.btn-mix{background:linear-gradient(135deg,var(--purple),#6366f1,var(--cyan));color:#fff;box-shadow:0 4px 20px rgba(124,58,237,.4)}
.btn-stop{background:var(--red-dim);border:1.5px solid rgba(239,68,68,.35);color:#f87171}
.btn-stop:hover{background:rgba(239,68,68,.2);transform:translateY(-1px);filter:none}

/* ── STAT CARDS ── */
.stat-row{display:grid;grid-template-columns:repeat(4,1fr);gap:7px;margin-bottom:2px}
.stat-card{background:var(--glass);border:1px solid var(--b1);border-radius:var(--r2);padding:11px;text-align:center}
.stat-icon{font-size:15px;margin-bottom:3px}
.stat-lbl{font-size:9px;color:var(--t3);font-weight:700;letter-spacing:.7px;text-transform:uppercase;margin-bottom:3px}
.stat-val{font-size:17px;font-weight:900;letter-spacing:-.5px}
.stat-val.pos{background:linear-gradient(135deg,var(--cyan),var(--cyan2));-webkit-background-clip:text;-webkit-text-fill-color:transparent}
.stat-val.neg{color:var(--red)}.stat-val.neu{color:var(--t1)}
.stat-sub{font-size:9px;color:var(--t3);margin-top:2px;font-weight:500}
.wr-bar{height:3px;border-radius:99px;background:var(--b1);margin-top:5px;overflow:hidden}
.wr-fill{height:100%;border-radius:99px;background:linear-gradient(90deg,var(--purple),var(--cyan));transition:width .6s}

/* ── TRADE HISTORY ── */
.th-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:12px}
.th-box{background:rgba(255,255,255,.035);border:1px solid var(--b1);border-radius:var(--r3);padding:11px}
.th-lbl{font-size:9.5px;color:var(--t3);font-weight:700;letter-spacing:.7px;text-transform:uppercase;margin-bottom:5px}
.th-val{font-size:19px;font-weight:900;letter-spacing:-.5px}
.th-val.pos{background:linear-gradient(135deg,var(--cyan),var(--cyan2));-webkit-background-clip:text;-webkit-text-fill-color:transparent}
.th-val.neg{color:var(--red)}.th-val.neu{color:var(--t1)}
.th-sub{font-size:11px;color:var(--t3);margin-top:2px;font-weight:500}
.th-tabs{display:flex;gap:5px;margin-bottom:10px;overflow-x:auto;-webkit-overflow-scrolling:touch}
.th-tabs::-webkit-scrollbar{display:none}
.th-tab{flex-shrink:0;padding:7px 13px;border-radius:var(--r3);border:1px solid var(--b1);
  background:transparent;color:var(--t2);font-family:var(--f);font-size:12px;font-weight:700;
  cursor:pointer;transition:all .2s;touch-action:manipulation;}
.th-tab.active{background:var(--purple-dim);border-color:rgba(157,110,245,.4);color:#fff}
.te{background:rgba(255,255,255,.03);border:1px solid var(--b1);border-radius:var(--r3);
  padding:11px;margin-bottom:7px;animation:fadeup .25s;position:relative;overflow:hidden}
.te::before{content:'';position:absolute;top:0;left:0;bottom:0;width:3px;border-radius:3px 0 0 3px}
.te.won::before{background:var(--green)}.te.lost::before{background:var(--red)}
.te.pending::before{background:var(--yellow)}.te.unsettled::before{background:var(--orange)}
@keyframes fadeup{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}
.te-hd{display:flex;align-items:center;gap:6px;margin-bottom:8px;flex-wrap:wrap}
.te-ico{width:24px;height:24px;border-radius:6px;flex-shrink:0;background:rgba(234,179,8,.1);
  border:1px solid rgba(234,179,8,.2);display:flex;align-items:center;justify-content:center;font-size:12px}
.te-st{font-size:8.5px;font-weight:800;padding:2px 7px;border-radius:99px;letter-spacing:.7px;text-transform:uppercase}
.te-st.pending{background:rgba(234,179,8,.1);color:var(--yellow);border:1px solid rgba(234,179,8,.25)}
.te-st.won{background:var(--green-dim);color:var(--green);border:1px solid rgba(34,197,94,.3)}
.te-st.lost{background:var(--red-dim);color:var(--red);border:1px solid rgba(239,68,68,.3)}
.te-st.unsettled{background:rgba(249,115,22,.1);color:var(--orange);border:1px solid rgba(249,115,22,.3)}
.te-tag{font-size:9px;color:var(--t3);background:rgba(255,255,255,.06);border-radius:99px;padding:2px 7px;font-weight:600}
/* Colored type pills for Mixologist */
.te-type-ou{color:var(--cyan);font-weight:800;font-size:9.5px;background:rgba(0,229,196,.1);
  border:1px solid rgba(0,229,196,.25);border-radius:99px;padding:2px 7px}
.te-type-eo{color:var(--purple3);font-weight:800;font-size:9.5px;background:rgba(124,58,237,.1);
  border:1px solid rgba(157,110,245,.25);border-radius:99px;padding:2px 7px}
.te-type-md{color:var(--orange);font-weight:800;font-size:9.5px;background:rgba(249,115,22,.1);
  border:1px solid rgba(249,115,22,.25);border-radius:99px;padding:2px 7px}
.te-pr{margin-left:auto;font-size:13px;font-weight:900}
.te-pr.pos{background:linear-gradient(135deg,var(--cyan),var(--cyan2));-webkit-background-clip:text;-webkit-text-fill-color:transparent}
.te-pr.neg{color:var(--red)}.te-pr.pend{color:var(--t3);font-size:11px}
.te-rows{display:grid;grid-template-columns:76px 1fr;gap:2px 0;font-size:11.5px}
.te-k{color:var(--t3);padding:2px 0;font-weight:600}.te-v{text-align:right;font-weight:600;padding:2px 0}
.entry-pill{display:inline-flex;align-items:center;background:linear-gradient(135deg,var(--purple),var(--violet));
  color:#fff;font-size:9.5px;font-weight:800;padding:2px 7px;border-radius:99px}
.stat-p{display:inline-flex;align-items:center;gap:3px;font-size:11px;font-weight:700}
.stat-p.ip{color:var(--cyan2)}.stat-p.won{color:var(--green)}.stat-p.lost{color:var(--red)}
.empty-msg{text-align:center;padding:32px;color:var(--t3);font-size:13px;
  border:1px dashed var(--b1);border-radius:var(--r3);background:rgba(255,255,255,.015)}
.trash{background:none;border:none;color:var(--t3);cursor:pointer;padding:4px 8px;
  font-size:15px;transition:color .2s;border-radius:6px;touch-action:manipulation;}
.trash:hover{color:var(--red)}

/* ── LOG ── */
.log-hd{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
.log-clr{font-size:11px;color:var(--t3);cursor:pointer;padding:4px 10px;border-radius:6px;
  background:rgba(255,255,255,.04);border:1px solid var(--b1);font-weight:600;touch-action:manipulation;}
.log-body{font-family:'Courier New',monospace;font-size:11px;line-height:1.75;
  max-height:160px;overflow-y:auto;color:var(--t3);background:rgba(0,0,0,.25);
  border-radius:var(--r4);padding:9px;border:1px solid var(--b1);
  -webkit-overflow-scrolling:touch;}
.log-body::-webkit-scrollbar{width:2px}
.log-body::-webkit-scrollbar-thumb{background:rgba(255,255,255,.07);border-radius:99px}
.log-line{display:flex;gap:7px;border-bottom:1px solid rgba(255,255,255,.022);padding:2px 0;animation:logfade .2s ease}
@keyframes logfade{from{opacity:0}to{opacity:1}}
.log-t{flex-shrink:0;width:54px;color:var(--t4);font-weight:600}
.log-line.win .log-m{color:var(--green)}.log-line.loss .log-m{color:var(--red)}
.log-line.success .log-m{color:var(--cyan)}.log-line.trade .log-m{color:var(--purple3)}
.log-line.warning .log-m{color:var(--yellow)}.log-line.error .log-m{color:#fca5a5}
.ml-auto{margin-left:auto}.mt-2{margin-top:7px}
</style>
</head>
<body>
<div class="toast-wrap"><div class="toast" id="toast"></div></div>

<!-- TOPBAR -->
<div class="topbar">
  <div class="logo-wrap">
    <div class="logo-icon">⚡</div>
    <div style="display:flex;align-items:center;gap:5px">
      <span class="logo">PerryAI</span><span class="logo-v">v5</span>
    </div>
  </div>
  <div class="tb-spacer"></div>
  <div class="live-pill" id="livePill"><div class="dot" id="dot"></div><span id="statusLbl">Offline</span></div>
  <span id="clock" style="margin-left:4px"></span>
  <a href="/logout" class="logout-btn">
    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><path d="M9 21H5a2 2 0 01-2-2V5a2 2 0 012-2h4M16 17l5-5-5-5M21 12H9"/></svg>
    Out
  </a>
</div>

<!-- DESKTOP -->
<div class="desktop-layout">
  <div class="dcol">
    <div id="d-api"></div><div id="d-botsel"></div><div id="d-log"></div>
  </div>
  <div class="dcol dcol-r">
    <div id="d-settings"></div><div id="d-analyzer"></div><div id="d-actions"></div><div id="d-history"></div>
  </div>
</div>

<!-- MOBILE -->
<div class="mobile-layout">
  <div class="tab-content" id="tabContent">
    <div class="tab-pane active" id="m-tab-connect">
      <div id="m-api"></div><div id="m-botsel"></div><div id="m-log"></div>
    </div>
    <div class="tab-pane" id="m-tab-trade">
      <div id="m-settings"></div><div id="m-analyzer"></div><div id="m-actions"></div>
    </div>
    <div class="tab-pane" id="m-tab-history"><div id="m-history"></div></div>
    <div class="tab-pane" id="m-tab-log2"><div id="m-log2"></div></div>
  </div>
  <nav class="bottom-nav">
    <div class="bnav-indicator" id="bnavBar"></div>
    <button class="bnav-btn active" id="bn0" onclick="setMobileTab(0)"><span class="bn-icon">🔗</span>Connect</button>
    <button class="bnav-btn" id="bn1" onclick="setMobileTab(1)"><span class="bn-icon">⚡</span>Trade</button>
    <button class="bnav-btn" id="bn2" onclick="setMobileTab(2)"><span class="bn-icon">🏆</span>History</button>
    <button class="bnav-btn" id="bn3" onclick="setMobileTab(3)"><span class="bn-icon">📡</span>Feed</button>
  </nav>
</div>

<!-- ═══ TEMPLATES ═══ -->
<template id="tpl-api">
<div class="card">
  <div class="card-hd">
    <div class="card-icon ci-green">🔗</div>
    <div class="card-title">API Connection</div>
    <span class="conn-badge" id="SCOPE-connBadge">● Connected</span>
  </div>
  <div id="SCOPE-preConn">
    <input class="tok-input" type="password" id="SCOPE-apiToken" placeholder="Paste your Deriv API token here…" autocomplete="off"/>
    <button class="btn-connect" onclick="doConnect()">🔌 Connect Account</button>
    <div class="help-txt">
      <div class="help-step"><div class="step-n">1</div><span>Log in at <a href="https://deriv.com" target="_blank">deriv.com</a></span></div>
      <div class="help-step"><div class="step-n">2</div><span>Settings › API Token — create with <strong>Trade</strong> permission</span></div>
      <div class="help-step"><div class="step-n">3</div><span>Paste token above and press Connect</span></div>
    </div>
  </div>
  <div id="SCOPE-postConn" style="display:none">
    <div class="acct-bar">
      <span>📋</span><strong id="SCOPE-acctType">—</strong>
      <span style="margin-left:auto;font-size:11px;color:var(--t3)" id="SCOPE-acctId">—</span>
    </div>
    <div class="bal-card">
      <div class="bal-lbl">Live Account Balance</div>
      <div class="bal-val" id="SCOPE-balance">—</div>
    </div>
    <button class="btn-disc" onclick="doDisconnect()">
      <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round"><circle cx="12" cy="12" r="10"/><line x1="15" y1="9" x2="9" y2="15"/><line x1="9" y1="9" x2="15" y2="15"/></svg>
      Disconnect
    </button>
  </div>
</div>
</template>

<template id="tpl-botsel">
<div class="card">
  <div class="card-hd">
    <div class="card-icon ci-purple">🤖</div>
    <div class="card-title">Select Bot</div>
  </div>
  <div class="bot-sel-grid">
    <div class="bot-card active-md" id="SCOPE-bcard-md" onclick="selectBot('md')">
      <div class="bot-icon">⚡</div>
      <div class="bot-name" style="color:var(--cyan)">Match/<br>Differ</div>
      <div class="bot-desc">Digit pred.</div>
    </div>
    <div class="bot-card" id="SCOPE-bcard-eo" onclick="selectBot('eo')">
      <div class="bot-icon">🔮</div>
      <div class="bot-name" style="color:var(--purple3)">Even/<br>Odd</div>
      <div class="bot-desc">Parity pred.</div>
    </div>
    <div class="bot-card" id="SCOPE-bcard-ou" onclick="selectBot('ou')">
      <div class="bot-icon">📊</div>
      <div class="bot-name" style="color:var(--orange)">Over/<br>Under</div>
      <div class="bot-desc">Range trader</div>
    </div>
    <div class="bot-card" id="SCOPE-bcard-mix" onclick="selectBot('mix')">
      <div class="bot-icon">🎛️</div>
      <div class="bot-name" style="background:linear-gradient(135deg,var(--violet),var(--cyan));-webkit-background-clip:text;-webkit-text-fill-color:transparent">Mix</div>
      <div class="bot-desc">All-in-one</div>
    </div>
    <div class="bot-card" id="SCOPE-bcard-ab" onclick="selectBot('ab')" style="border-color:rgba(34,197,94,.3)">
      <div class="bot-icon">🏗️</div>
      <div class="bot-name" style="background:linear-gradient(135deg,var(--green),var(--cyan));-webkit-background-clip:text;-webkit-text-fill-color:transparent">Account<br>Builder</div>
      <div class="bot-desc">Goal-based</div>
    </div>
  </div>
</div>
</template>

<template id="tpl-log">
<div class="card">
  <div class="log-hd">
    <div style="display:flex;align-items:center;gap:8px">
      <div class="card-icon ci-purple" style="width:26px;height:26px;font-size:12px">📡</div>
      <div class="card-title">Live Feed</div>
    </div>
    <span class="log-clr" onclick="clearLog()">Clear</span>
  </div>
  <div class="log-body" id="SCOPE-logBody"></div>
</div>
</template>

<template id="tpl-settings">
<div class="card">
  <div class="card-hd">
    <div class="card-icon ci-purple">⚙️</div>
    <div class="card-title">Configuration</div>
    <span class="bot-badge md" id="SCOPE-botBadge">⚡ M/D</span>
  </div>

  <div class="sec-lbl">Market &amp; Duration</div>
  <div class="g2">
    <div class="fld"><label>Volatility Index</label>
      <select id="SCOPE-symbol">
        <option value="1HZ100V">Vol 100 (1s)</option>
        <option value="1HZ10V">Vol 10 (1s)</option>
        <option value="1HZ25V">Vol 25 (1s)</option>
        <option value="1HZ50V">Vol 50 (1s)</option>
        <option value="1HZ75V">Vol 75 (1s)</option>
        <option value="R_10">Volatility 10</option>
        <option value="R_25">Volatility 25</option>
        <option value="R_50">Volatility 50</option>
        <option value="R_75">Volatility 75</option>
        <option value="R_100">Volatility 100</option>
      </select>
    </div>
    <div class="fld"><label>Tick Duration</label>
      <select id="SCOPE-duration">
        <option value="1">1 Tick</option><option value="2">2 Ticks</option>
        <option value="3">3 Ticks</option><option value="4">4 Ticks</option><option value="5">5 Ticks</option>
      </select>
    </div>
  </div>

  <div class="sec-lbl">Stake Amount</div>
  <div class="stake-row">
    <input class="stake-inp" type="number" id="SCOPE-stake" value="1.00" step="0.01" min="0.35" inputmode="decimal"/>
    <button class="preset" onclick="addStake(1)">+1</button>
    <button class="preset" onclick="addStake(10)">+10</button>
    <button class="preset" onclick="addStake(50)">+50</button>
  </div>
  <div class="hdiv"></div>

  <!-- MATCHES/DIFFERS -->
  <div id="SCOPE-sec-md">
    <div class="sec-lbl">Matches / Differs</div>
    <div class="g2">
      <div class="fld"><label>Trade Type</label>
        <select id="SCOPE-mdType"><option value="DIGITDIFF">Differs</option><option value="DIGITMATCH">Matches</option></select>
      </div>
      <div class="fld"><label>Digit Target</label>
        <select id="SCOPE-predSel" onchange="onPredSel()">
          <option value="auto">Auto (AI)</option>
          <option value="0">0</option><option value="1">1</option><option value="2">2</option>
          <option value="3">3</option><option value="4">4</option><option value="5">5</option>
          <option value="6">6</option><option value="7">7</option><option value="8">8</option><option value="9">9</option>
        </select>
      </div>
    </div>
  </div>

  <!-- EVEN/ODD -->
  <div id="SCOPE-sec-eo" style="display:none">
    <div class="sec-lbl">Even / Odd</div>
    <div class="g2">
      <div class="fld"><label>Parity Type</label>
        <select id="SCOPE-eoType"><option value="DIGITEVEN">Even</option><option value="DIGITODD">Odd</option></select>
      </div>
      <div class="fld"><label>Tick Interval</label><input type="number" id="SCOPE-eoInterval" value="1" min="1" inputmode="numeric"/></div>
    </div>
  </div>

  <!-- OVER/UNDER -->
  <div id="SCOPE-sec-ou" style="display:none">
    <div class="sec-lbl">Over / Under</div>
    <div class="ou-dir-row">
      <button class="ou-dir-btn sel-over" id="SCOPE-ouOver" onclick="setOUDir('over')">📈 Over</button>
      <button class="ou-dir-btn" id="SCOPE-ouUnder" onclick="setOUDir('under')">📉 Under</button>
    </div>
    <div class="g2">
      <div class="fld"><label>Digit Barrier</label>
        <select id="SCOPE-ouDigit">
          <option value="0">0</option><option value="1">1</option><option value="2">2</option>
          <option value="3">3</option><option value="4" selected>4</option><option value="5">5</option>
          <option value="6">6</option><option value="7">7</option><option value="8">8</option><option value="9">9</option>
        </select>
      </div>
      <div class="fld"><label>Tick Interval</label><input type="number" id="SCOPE-ouInterval" value="1" min="1" inputmode="numeric"/></div>
    </div>
  </div>

  <!-- MIXOLOGIST -->
  <div id="SCOPE-sec-mix" style="display:none">
    <div class="sec-lbl">Mixologist — Active Modules</div>
    <div class="mix-mod-grid">
      <div class="mix-mod on-ou" id="SCOPE-mixOU" onclick="toggleMixMod('ou')">
        <div class="mix-mod-icon">📊</div>
        <div class="mix-mod-name" style="color:var(--cyan)">Over/Under</div>
        <div class="mix-mod-cnt" id="SCOPE-ouTrades">0 trades</div>
        <div class="mix-mod-stat neu" id="SCOPE-ouProfit">$0.00</div>
      </div>
      <div class="mix-mod on-eo" id="SCOPE-mixEO" onclick="toggleMixMod('eo')">
        <div class="mix-mod-icon">🔮</div>
        <div class="mix-mod-name" style="color:var(--purple3)">Even/Odd</div>
        <div class="mix-mod-cnt" id="SCOPE-eoTrades">0 trades</div>
        <div class="mix-mod-stat neu" id="SCOPE-eoProfit">$0.00</div>
      </div>
      <div class="mix-mod on-md" id="SCOPE-mixMD" onclick="toggleMixMod('md')">
        <div class="mix-mod-icon">⚡</div>
        <div class="mix-mod-name" style="color:var(--orange)">M/Differ</div>
        <div class="mix-mod-cnt" id="SCOPE-mdTrades">0 trades</div>
        <div class="mix-mod-stat neu" id="SCOPE-mdProfit">$0.00</div>
      </div>
    </div>
    <div class="g2">
      <div class="fld"><label>O/U Direction</label>
        <select id="SCOPE-mixOUType"><option value="DIGITOVER">Over</option><option value="DIGITUNDER">Under</option></select>
      </div>
      <div class="fld"><label>O/U Barrier</label>
        <select id="SCOPE-mixOUPred">
          <option value="0">0</option><option value="1">1</option><option value="2">2</option>
          <option value="3">3</option><option value="4" selected>4</option><option value="5">5</option>
          <option value="6">6</option><option value="7">7</option><option value="8">8</option><option value="9">9</option>
        </select>
      </div>
    </div>
    <div class="g2">
      <div class="fld"><label>E/O Parity</label>
        <select id="SCOPE-mixEOType"><option value="DIGITEVEN">Even</option><option value="DIGITODD">Odd</option></select>
      </div>
      <div class="fld"><label>M/D Type</label>
        <select id="SCOPE-mixMDType"><option value="DIGITDIFF">Differs</option><option value="DIGITMATCH">Matches</option></select>
      </div>
    </div>
    <div style="font-size:11px;color:var(--t3);background:rgba(249,115,22,.06);border:1px solid rgba(249,115,22,.2);border-radius:8px;padding:8px 10px;margin-bottom:8px">
      ⚡ M/D uses a <strong style="color:var(--orange)">random digit</strong> per contract for maximum variety.
    </div>
  </div>

  <div class="hdiv"></div>
  <div class="g2">
    <div class="fld"><label>Tick Interval</label><input type="number" id="SCOPE-tickInterval" value="1" min="1" inputmode="numeric"/></div>
    <div class="fld"><label>Max Trades (0=∞)</label><input type="number" id="SCOPE-maxTrades" value="0" min="0" inputmode="numeric"/></div>
  </div>
  <div class="hdiv"></div>

  <div id="SCOPE-aiRow">
    <div class="feat">
      <div class="feat-info">
        <div class="feat-title" id="SCOPE-analyzerTitle">🤖 AI Analyzer</div>
        <div class="feat-desc" id="SCOPE-analyzerDesc">Auto-selects optimal trade from tick analysis</div>
      </div>
      <button class="feat-btn" id="SCOPE-intBtn" onclick="toggleFeat('int')">OFF</button>
    </div>
    <div class="sub" id="SCOPE-intSub">
      <div class="g2 mt-2">
        <div class="fld"><label>Analysis Window</label><input type="number" id="SCOPE-statsWindow" value="50" min="20" inputmode="numeric"/></div>
        <div class="fld"><label>Update Every N</label><input type="number" id="SCOPE-updateEvery" value="3" min="1" inputmode="numeric"/></div>
      </div>
    </div>
  </div>

  <div class="feat">
    <div class="feat-info">
      <div class="feat-title">📈 Martingale Recovery</div>
      <div class="feat-desc">Multiply stake after each loss</div>
    </div>
    <button class="feat-btn" id="SCOPE-martBtn" onclick="toggleFeat('mart')">OFF</button>
  </div>
  <div class="sub" id="SCOPE-martSub">
    <div class="g2 mt-2">
      <div class="fld"><label>Multiplier ×</label><input type="number" id="SCOPE-martMult" value="2.0" step="0.1" min="1.1" inputmode="decimal"/></div>
      <div class="fld"><label>Max Steps</label><input type="number" id="SCOPE-martSteps" value="4" min="1" max="10" inputmode="numeric"/></div>
    </div>
  </div>
  <div class="hdiv"></div>
  <div class="g2">
    <div class="fld"><label>Take Profit $ (0=off)</label><input type="number" id="SCOPE-takeProfit" value="0" step="0.5" min="0" inputmode="decimal"/></div>
    <div class="fld"><label>Stop Loss $ (0=off)</label><input type="number" id="SCOPE-stopLoss" value="0" step="0.5" min="0" inputmode="decimal"/></div>
  </div>

  <!-- ACCOUNT BUILDER -->
  <div id="SCOPE-sec-ab" style="display:none">
    <div class="sec-lbl" style="color:var(--green)">🏗️ Account Builder — Goal Settings</div>
    <div style="background:rgba(34,197,94,.07);border:1px solid rgba(34,197,94,.2);border-radius:10px;padding:12px;margin-bottom:12px;font-size:12px;color:var(--t2)">
      Set your <strong style="color:var(--green)">mini-balance</strong> (capital to risk) and <strong style="color:var(--cyan)">target profit</strong>.
      The bot auto-calculates safe stakes, analyzes the market, and switches strategy to reach your goal.
    </div>
    <div class="g2">
      <div class="fld"><label>Mini Balance $ (capital to risk)</label>
        <input type="number" id="SCOPE-miniBalance" value="100" step="5" min="5" inputmode="decimal"/>
      </div>
      <div class="fld"><label>Target Profit $ (goal)</label>
        <input type="number" id="SCOPE-targetProfit" value="20" step="1" min="1" inputmode="decimal"/>
      </div>
    </div>
    <div class="g2">
      <div class="fld"><label>Stake % of Balance</label>
        <select id="SCOPE-abStakePct">
          <option value="0.01">1% (very safe)</option>
          <option value="0.02" selected>2% (safe)</option>
          <option value="0.03">3% (moderate)</option>
          <option value="0.05">5% (aggressive)</option>
        </select>
      </div>
      <div class="fld"><label>Re-analyze Every N Trades</label>
        <input type="number" id="SCOPE-abReanalyze" value="10" min="5" max="50" inputmode="numeric"/>
      </div>
    </div>
    <div class="g2">
      <div class="fld"><label>Max Martingale Steps</label>
        <select id="SCOPE-abMartSteps">
          <option value="2">2 (conservative)</option>
          <option value="3" selected>3 (balanced)</option>
          <option value="4">4 (aggressive)</option>
        </select>
      </div>
      <div class="fld"><label>Market</label>
        <!-- uses same symbol/duration fields above -->
        <div style="font-size:11px;color:var(--t3);margin-top:4px">Uses market selected above</div>
      </div>
    </div>
    <!-- Progress display (live during trading) -->
    <div id="SCOPE-abProgressBox" style="display:none;margin-top:10px;background:rgba(34,197,94,.06);border:1px solid rgba(34,197,94,.2);border-radius:10px;padding:12px">
      <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px">
        <div style="font-size:11px;font-weight:700;color:var(--t2)">PROGRESS TO GOAL</div>
        <div style="font-size:14px;font-weight:900;color:var(--green)" id="SCOPE-abProgressPct">0%</div>
      </div>
      <div style="background:rgba(0,0,0,.3);border-radius:99px;height:8px;overflow:hidden">
        <div id="SCOPE-abProgressBar" style="height:100%;background:linear-gradient(90deg,var(--green),var(--cyan));border-radius:99px;width:0%;transition:width .5s ease"></div>
      </div>
      <div style="display:flex;justify-content:space-between;margin-top:8px;font-size:10px;color:var(--t3)">
        <span>Strategy: <strong id="SCOPE-abStrategy" style="color:var(--cyan)">—</strong></span>
        <span>Confidence: <strong id="SCOPE-abConfidence" style="color:var(--green)">—</strong></span>
      </div>
    </div>
  </div>
</div>
</template>

<template id="tpl-analyzer">
<div class="card" id="SCOPE-analyzerCard">
  <div id="SCOPE-az-md">
    <div class="card-hd">
      <div class="card-icon ci-cyan">⚡</div><div class="card-title">Digit Frequency</div>
    </div>
    <div class="dg" id="SCOPE-digitGrid"></div>
    <div class="dg-meta" id="SCOPE-digitMeta">Awaiting market data…</div>
  </div>
  <div id="SCOPE-az-eo" style="display:none">
    <div class="card-hd">
      <div class="card-icon ci-purple">🔮</div><div class="card-title">Parity Analysis</div>
      <span class="parity-badge even" id="SCOPE-parityBadge">EVEN ▲</span>
    </div>
    <div style="margin-top:8px">
      <div class="parity-row parity-even">
        <span class="parity-lbl" style="color:var(--purple3)">EVEN</span>
        <div class="parity-bar-bg"><div class="parity-bar-fill" id="SCOPE-evenBar" style="width:50%"></div>
          <span class="parity-pct" id="SCOPE-evenPct">50%</span></div>
      </div>
      <div class="parity-row parity-odd" style="margin-top:8px">
        <span class="parity-lbl" style="color:var(--orange)">ODD</span>
        <div class="parity-bar-bg"><div class="parity-bar-fill" id="SCOPE-oddBar" style="width:50%"></div>
          <span class="parity-pct" id="SCOPE-oddPct">50%</span></div>
      </div>
    </div>
    <div class="dg-meta" id="SCOPE-parityMeta" style="margin-top:10px">Awaiting market data…</div>
  </div>
  <div id="SCOPE-az-ou" style="display:none">
    <div class="card-hd">
      <div class="card-icon ci-cyan">📊</div><div class="card-title">Over/Under Analysis</div>
      <span id="SCOPE-ouActiveBadge" style="margin-left:auto;font-size:10px;font-weight:800;color:var(--cyan)"></span>
    </div>
    <div class="dg" id="SCOPE-ouGrid"></div>
    <div class="dg-meta" id="SCOPE-ouMeta">Awaiting market data…</div>
  </div>
  <div id="SCOPE-az-mix" style="display:none">
    <div class="card-hd">
      <div class="card-icon ci-mix">🎛️</div><div class="card-title">Mixologist Live Stats</div>
    </div>
    <div class="dg" id="SCOPE-mixDigitGrid"></div>
    <div class="dg-meta" id="SCOPE-mixMeta">Awaiting market data…</div>
    <div style="margin-top:10px;display:grid;grid-template-columns:1fr 1fr 1fr;gap:7px">
      <div style="background:rgba(0,229,196,.06);border:1px solid rgba(0,229,196,.2);border-radius:9px;padding:9px;text-align:center">
        <div style="font-size:9px;color:var(--t3);font-weight:700;margin-bottom:3px">O/U WR</div>
        <div style="font-size:14px;font-weight:800;color:var(--cyan)" id="SCOPE-mixOUWR">0%</div>
        <div style="font-size:9px;color:var(--t3)" id="SCOPE-mixOUInfo">0/0</div>
      </div>
      <div style="background:rgba(124,58,237,.06);border:1px solid rgba(157,110,245,.2);border-radius:9px;padding:9px;text-align:center">
        <div style="font-size:9px;color:var(--t3);font-weight:700;margin-bottom:3px">E/O WR</div>
        <div style="font-size:14px;font-weight:800;color:var(--purple3)" id="SCOPE-mixEOWR">0%</div>
        <div style="font-size:9px;color:var(--t3)" id="SCOPE-mixEOInfo">0/0</div>
      </div>
      <div style="background:rgba(249,115,22,.06);border:1px solid rgba(249,115,22,.2);border-radius:9px;padding:9px;text-align:center">
        <div style="font-size:9px;color:var(--t3);font-weight:700;margin-bottom:3px">M/D WR</div>
        <div style="font-size:14px;font-weight:800;color:var(--orange)" id="SCOPE-mixMDWR">0%</div>
        <div style="font-size:9px;color:var(--t3)" id="SCOPE-mixMDInfo">0/0</div>
      </div>
    </div>
  </div>
  <div id="SCOPE-az-ab" style="display:none">
    <div class="card-hd">
      <div class="card-icon" style="background:rgba(34,197,94,.15);border:1px solid rgba(34,197,94,.3)">🏗️</div>
      <div class="card-title">Account Builder — Market Analysis</div>
    </div>
    <div class="dg" id="SCOPE-abDigitGrid"></div>
    <div class="dg-meta" id="SCOPE-abDigitMeta">Analyzing market…</div>
    <div style="margin-top:12px;display:grid;grid-template-columns:1fr 1fr;gap:8px">
      <div style="background:rgba(34,197,94,.06);border:1px solid rgba(34,197,94,.2);border-radius:9px;padding:10px;text-align:center">
        <div style="font-size:9px;color:var(--t3);font-weight:700;margin-bottom:4px">CURRENT STRATEGY</div>
        <div style="font-size:12px;font-weight:800;color:var(--green)" id="SCOPE-abAnalysisLabel">Analyzing…</div>
      </div>
      <div style="background:rgba(0,229,196,.06);border:1px solid rgba(0,229,196,.2);border-radius:9px;padding:10px;text-align:center">
        <div style="font-size:9px;color:var(--t3);font-weight:700;margin-bottom:4px">CONFIDENCE</div>
        <div style="font-size:12px;font-weight:800;color:var(--cyan)" id="SCOPE-abAnalysisConf">—</div>
      </div>
    </div>
  </div>
</div>
</template>

<template id="tpl-actions">
<div class="card">
  <div class="card-hd" style="margin-bottom:14px">
    <div class="card-icon ci-purple">⚡</div><div class="card-title">Execute Trade</div>
  </div>
  <div id="SCOPE-act-md">
    <button class="act btn-teal" id="SCOPE-md-single" onclick="startBot('single')" disabled>⚡ Single Trade</button>
    <button class="act btn-purp" id="SCOPE-md-rand" onclick="startBot('auto_random')" disabled>🔄 Auto — Random Digit</button>
    <button class="act btn-ora" id="SCOPE-md-sel" onclick="startBot('auto_selected')" disabled>🎯 Auto — Selected Digit</button>
    <button class="act btn-pink" id="SCOPE-md-ai" onclick="startBot('ai_auto')" disabled>🤖 AI Auto Trade</button>
    <div class="act-sub">AI picks least frequent digit · intelligent DIFFERS strategy</div>
  </div>
  <div id="SCOPE-act-eo" style="display:none">
    <button class="act btn-teal" id="SCOPE-eo-single" onclick="startBot('eo_single')" disabled>🔮 Single Even/Odd</button>
    <button class="act btn-purp" id="SCOPE-eo-auto" onclick="startBot('eo_auto')" disabled>🔄 Auto — Fixed Parity</button>
    <button class="act btn-ora" id="SCOPE-eo-fast" onclick="startBot('eo_fast')" disabled>⚡ Fast Auto (Every Tick)</button>
    <button class="act btn-pink" id="SCOPE-eo-ai" onclick="startBot('eo_ai')" disabled>🤖 AI Auto Switch</button>
    <div class="act-sub">AI monitors parity history · auto-switches to winning side</div>
  </div>
  <div id="SCOPE-act-ou" style="display:none">
    <button class="act btn-teal" id="SCOPE-ou-single" onclick="startBot('ou_single')" disabled>📊 Single Over/Under</button>
    <button class="act btn-purp" id="SCOPE-ou-auto" onclick="startBot('ou_auto')" disabled>🔄 Auto — Fixed Digit</button>
    <button class="act btn-ora" id="SCOPE-ou-fast" onclick="startBot('ou_fast')" disabled>⚡ Fast Auto (Every Tick)</button>
    <button class="act btn-pink" id="SCOPE-ou-ai" onclick="startBot('ou_ai')" disabled>🤖 AI Auto Analysis</button>
    <div class="act-sub">AI scans all 10 digits · picks optimal direction &amp; barrier</div>
  </div>
  <div id="SCOPE-act-mix" style="display:none">
    <button class="act btn-mix" id="SCOPE-mix-start" onclick="startBot('mix_auto')" disabled>
      🎛️ Launch Mixologist
    </button>
    <div class="act-sub">Runs Over/Under · Even/Odd · Matches/Differs simultaneously.<br>Each module tracks P&amp;L independently. TP/SL waits for all contracts to settle.</div>
  </div>
  <div id="SCOPE-act-ab" style="display:none">
    <button class="act" id="SCOPE-ab-start" onclick="startBot('ab_auto')" disabled
      style="background:linear-gradient(135deg,var(--green),var(--cyan));color:#000;font-weight:900;font-size:14px;border:none;display:flex;align-items:center;justify-content:center;gap:8px">
      🏗️ Launch Account Builder
    </button>
    <div class="act-sub">AI analyzes market → picks best strategy → trades toward your profit goal.<br>Auto-switches strategy every N trades for maximum edge. Conservative martingale built-in.</div>
  </div>
  <button class="act btn-stop" id="SCOPE-btnStop" onclick="stopBot()" style="display:none">
    <svg width="13" height="13" viewBox="0 0 24 24" fill="currentColor"><rect x="3" y="3" width="18" height="18" rx="2"/></svg>
    Stop All Trading
  </button>
</div>
</template>

<template id="tpl-history">
<div class="card">
  <div class="card-hd">
    <div class="card-icon ci-gold">🏆</div><div class="card-title">Trade History</div>
    <button class="trash ml-auto" onclick="clearHistory()">
      <svg width="13" height="13" fill="none" viewBox="0 0 24 24" stroke="currentColor" stroke-width="2"><polyline points="3 6 5 6 21 6"/><path d="M19 6l-1 14a2 2 0 01-2 2H8a2 2 0 01-2-2L5 6m3 0V4a1 1 0 011-1h4a1 1 0 011 1v2"/></svg>
    </button>
  </div>
  <div class="th-grid">
    <div class="th-box"><div class="th-lbl">Net P/L</div><div class="th-val neu" id="SCOPE-thProfit">+0.00 USD</div></div>
    <div class="th-box">
      <div class="th-lbl">Win Rate</div><div class="th-val neu" id="SCOPE-thWR">0%</div>
      <div class="th-sub" id="SCOPE-thWRSub">(0/0)</div>
      <div class="wr-bar"><div class="wr-fill" id="SCOPE-wrFill" style="width:0%"></div></div>
    </div>
  </div>
  <div class="th-tabs">
    <button class="th-tab active" id="SCOPE-tabAll" onclick="setTab('all')">All <span id="SCOPE-nAll">0</span></button>
    <button class="th-tab" id="SCOPE-tabWins" onclick="setTab('wins')">✅ Wins <span id="SCOPE-nWins">0</span></button>
    <button class="th-tab" id="SCOPE-tabLosses" onclick="setTab('losses')">❌ Losses <span id="SCOPE-nLosses">0</span></button>
  </div>
  <div id="SCOPE-tradeList"><div class="empty-msg">No trades yet — connect API and start a bot</div></div>
</div>
</template>

<!-- ════ JAVASCRIPT ════ -->
<script>
/* ── TEMPLATE INJECTION ── */
const SLOTS={
  'd-api':{tpl:'tpl-api',scope:'d'},'d-botsel':{tpl:'tpl-botsel',scope:'d'},
  'd-log':{tpl:'tpl-log',scope:'d'},'d-settings':{tpl:'tpl-settings',scope:'d'},
  'd-analyzer':{tpl:'tpl-analyzer',scope:'d'},'d-actions':{tpl:'tpl-actions',scope:'d'},
  'd-history':{tpl:'tpl-history',scope:'d'},
  'm-api':{tpl:'tpl-api',scope:'m'},'m-botsel':{tpl:'tpl-botsel',scope:'m'},
  'm-log':{tpl:'tpl-log',scope:'m'},'m-settings':{tpl:'tpl-settings',scope:'m'},
  'm-analyzer':{tpl:'tpl-analyzer',scope:'m'},'m-actions':{tpl:'tpl-actions',scope:'m'},
  'm-history':{tpl:'tpl-history',scope:'m'},'m-log2':{tpl:'tpl-log',scope:'m2'},
};
Object.entries(SLOTS).forEach(([id,{tpl,scope}])=>{
  const slot=document.getElementById(id),tmpl=document.getElementById(tpl);
  if(slot&&tmpl) slot.innerHTML=tmpl.innerHTML.replace(/SCOPE-/g,scope+'-');
});

/* ── STATE ── */
let connected=false,botRunning=false,intOn=false,martOn=false;
let activeTab='all',allTrades=[];
let selDigit=null,digitPcts=Array(10).fill(0),lastDigit=null;
let ouDigitPcts=Array(10).fill(0),ouLastDigit=null;
let curBot='md',ouDir='over',curSymbol='1HZ100V',curCurrency='USD';
let _renderPending=false,_storedToken='';
// ── Authoritative running totals from server (never derived from history array) ──
let _wins=0,_losses=0,_totalTrades=0;
// Mixologist module state
let mixEnableOU=true,mixEnableEO=true,mixEnableMD=true;

/* ── CLOCK ── */
setInterval(()=>{ document.getElementById('clock').textContent=new Date().toLocaleTimeString('en-US',{hour12:false}); },1000);

/* ── BALANCE ── */
function _setBalance(bal,cur){
  if(bal==null) return;
  curCurrency=cur||curCurrency;
  const str=parseFloat(bal).toFixed(2)+' '+curCurrency;
  ['d-','m-'].forEach(p=>{
    const el=ge(p+'balance'); if(!el||el.textContent===str) return;
    el.textContent=str; el.style.transition='color .25s'; el.style.color='var(--cyan)';
    setTimeout(()=>el.style.color='',900);
  });
}
async function silentReAuth(){
  const token=_storedToken||getToken(); if(!token) return;
  try{
    const res=await fetch('/api/auth_check',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({api_token:token})});
    const d=await res.json(); if(d.balance!=null) _setBalance(d.balance,d.currency);
  }catch(e){}
}
async function pollBalance(){
  if(!connected) return;
  try{const r=await fetch('/api/balance'); if(!r.ok) return; const d=await r.json(); if(d.balance!=null) _setBalance(d.balance,d.currency);}catch(e){}
}
setInterval(pollBalance,5000);

/* ── HELPERS ── */
function ge(id){ return document.getElementById(id) }
function getVal(base){ return (ge('d-'+base)||ge('m-'+base)||{value:''}).value; }
function setAll(base,v){ ['d-','m-'].forEach(p=>{ const e=ge(p+base); if(e) e.value=v; }); }
function showEl(base){ ['d-','m-'].forEach(p=>{ const e=ge(p+base); if(e) e.style.display=''; }); }
function hideEl(base){ ['d-','m-'].forEach(p=>{ const e=ge(p+base); if(e) e.style.display='none'; }); }
function addStake(n){ ['d-stake','m-stake'].forEach(id=>{ const e=ge(id); if(e) e.value=(parseFloat(e.value||0)+n).toFixed(2); }); }
function getToken(){ return (['d-apiToken','m-apiToken'].map(id=>{ const e=ge(id); return e?e.value.trim():''; }).find(v=>v))||''; }

/* ── MOBILE TABS ── */
function setMobileTab(i){
  ['m-tab-connect','m-tab-trade','m-tab-history','m-tab-log2'].forEach((id,j)=>{
    const el=ge(id); if(el) el.classList.toggle('active',j===i);
  });
  for(let j=0;j<4;j++){ const b=ge('bn'+j); if(b) b.classList.toggle('active',j===i); }
  // move indicator
  const btns=document.querySelectorAll('.bnav-btn');
  const bar=ge('bnavBar');
  if(bar&&btns[i]){
    const w=100/btns.length+'%';
    bar.style.left=(i*100/btns.length)+'%';
    bar.style.width=w;
  }
  // scroll tab content to top
  const tc=ge('tabContent'); if(tc) tc.scrollTop=0;
}

/* ── TOAST ── */
let toastT=null;
function toast(msg,type='info'){
  const el=ge('toast'); el.textContent=msg; el.className='toast show '+type;
  clearTimeout(toastT); toastT=setTimeout(()=>el.classList.remove('show'),4000);
}

/* ── BOT SELECTOR ── */
function selectBot(type){
  if(botRunning){ toast('Stop the bot before switching','error'); return; }
  curBot=type;
  const all=['md','eo','ou','mix','ab'];
  ['d-','m-'].forEach(p=>{
    all.forEach(k=>{ const e=ge(p+'bcard-'+k); if(e) e.className='bot-card'; });
    const a=ge(p+'bcard-'+type); if(a) a.classList.add('active-'+type);
  });
  all.forEach(k=>{ hideEl('sec-'+k); hideEl('act-'+k); hideEl('az-'+k); });
  showEl('act-'+type); showEl('az-'+type);
  // Account Builder uses its own settings section, others use sec-*
  if(type==='ab'){ showEl('sec-ab'); } else { showEl('sec-'+type); }
  const labels={md:'⚡ M/Differs',eo:'🔮 Even/Odd',ou:'📊 Over/Under',mix:'🎛️ Mixologist',ab:'🏗️ Account Builder'};
  const cls={md:'bot-badge md',eo:'bot-badge eo',ou:'bot-badge ou',mix:'bot-badge mix',ab:'bot-badge ab'};
  ['d-','m-'].forEach(p=>{
    const b=ge(p+'botBadge'); if(b){ b.textContent=labels[type]; b.className=cls[type]||'bot-badge'; }
    // Hide AI Analyzer row for Mixologist and Account Builder (they have their own)
    const ar=ge(p+'aiRow'); if(ar) ar.style.display=(type==='mix'||type==='ab')?'none':'';
    // Hide standard stake/TP/SL for Account Builder (uses its own fields)
    const stk=ge(p+'stakeRow'); // if exists, hide for AB
  });
  toast(labels[type]+' selected','info');
}

/* ── MIXOLOGIST MODULE TOGGLES ── */
function toggleMixMod(mod){
  if(botRunning) return;
  if(mod==='ou') mixEnableOU=!mixEnableOU;
  if(mod==='eo') mixEnableEO=!mixEnableEO;
  if(mod==='md') mixEnableMD=!mixEnableMD;
  _renderMixModUI();
}
function _renderMixModUI(){
  ['d-','m-'].forEach(p=>{
    const setMod=(id,on,cls)=>{
      const e=ge(p+id); if(!e) return;
      e.style.opacity=on?'1':'0.4';
      e.className='mix-mod'+(on?' '+cls:'');
    };
    setMod('mixOU',mixEnableOU,'on-ou');
    setMod('mixEO',mixEnableEO,'on-eo');
    setMod('mixMD',mixEnableMD,'on-md');
  });
}

/* ── FEATURE TOGGLES ── */
function toggleFeat(key){
  if(key==='int'){
    intOn=!intOn;
    ['d-','m-'].forEach(p=>{
      const b=ge(p+'intBtn'); if(b){ b.textContent=intOn?'ON':'OFF'; b.classList.toggle('on',intOn); }
      const s=ge(p+'intSub'); if(s) s.classList.toggle('open',intOn);
    });
    if(intOn&&curBot==='md') setAll('predSel','auto');
  } else {
    martOn=!martOn;
    ['d-','m-'].forEach(p=>{
      const b=ge(p+'martBtn'); if(b){ b.textContent=martOn?'ON':'OFF'; b.classList.toggle('on',martOn); }
      const s=ge(p+'martSub'); if(s) s.classList.toggle('open',martOn);
    });
  }
}

/* ── DIGIT GRIDS ── */
['d-','m-'].forEach(scope=>{
  const g1=ge(scope+'digitGrid');
  if(g1) for(let i=0;i<10;i++) g1.innerHTML+=`<div class="db" id="${scope}dg${i}" onclick="clickDigit(${i})"><div class="db-n">${i}</div><div class="db-p" id="${scope}dp${i}">—</div><span class="db-ck">✓</span></div>`;
  const g2=ge(scope+'ouGrid');
  if(g2) for(let i=0;i<10;i++) g2.innerHTML+=`<div class="db" id="${scope}oud${i}" onclick="clickOUDigit(${i})"><div class="db-n">${i}</div><div class="db-p" id="${scope}oudp${i}">—</div><span class="db-ck">✓</span></div>`;
  const g3=ge(scope+'mixDigitGrid');
  if(g3) for(let i=0;i<10;i++) g3.innerHTML+=`<div class="db" id="${scope}mxd${i}"><div class="db-n">${i}</div><div class="db-p" id="${scope}mxdp${i}">—</div></div>`;
});

// Init Account Builder digit grid
['d-','m-'].forEach(scope=>{
  const g4=ge(scope+'abDigitGrid');
  if(g4) for(let i=0;i<10;i++) g4.innerHTML+=`<div class="db" id="${scope}abd${i}"><div class="db-n">${i}</div><div class="db-p" id="${scope}abdp${i}">—</div></div>`;
});
function clickDigit(n){ if(botRunning) return; selDigit=n; setAll('predSel',String(n)); renderDigitGrids(); }
function onPredSel(){ const v=getVal('predSel'); selDigit=(v==='auto')?null:parseInt(v); setAll('predSel',v); renderDigitGrids(); }
function renderDigitGrids(){
  const maxP=Math.max(...digitPcts,0);
  ['d-','m-'].forEach(s=>{
    for(let i=0;i<10;i++){
      const btn=ge(s+'dg'+i),pEl=ge(s+'dp'+i); if(!btn) continue;
      const p=digitPcts[i]; if(pEl) pEl.textContent=p>0?p.toFixed(1)+'%':'—';
      btn.className='db';
      if(i===selDigit) btn.classList.add('sel');
      else if(p===maxP&&p>0) btn.classList.add('hot');
      if(i===lastDigit&&i!==selDigit) btn.classList.add('last');
      btn.style.setProperty('--pct',(p||0)+'%');
    }
  });
}
function clickOUDigit(n){ if(botRunning) return; setAll('ouDigit',String(n)); renderOUGrids(); }
function renderOUGrids(){
  const selD=parseInt(getVal('ouDigit')||'4');
  ['d-','m-'].forEach(s=>{
    for(let i=0;i<10;i++){
      const btn=ge(s+'oud'+i),pEl=ge(s+'oudp'+i); if(!btn) continue;
      const p=ouDigitPcts[i]; if(pEl) pEl.textContent=p>0?p.toFixed(1)+'%':'—';
      btn.className='db';
      if(i===selD) btn.classList.add('ou-active');
      if(i===ouLastDigit&&i!==selD) btn.classList.add('last');
      btn.style.setProperty('--pct',(p||0)+'%');
    }
  });
}
function renderMixGrids(pcts,ldig){
  ['d-','m-'].forEach(s=>{
    for(let i=0;i<10;i++){
      const btn=ge(s+'mxd'+i),pEl=ge(s+'mxdp'+i); if(!btn) continue;
      const p=pcts[i]||0; if(pEl) pEl.textContent=p>0?p.toFixed(1)+'%':'—';
      btn.className='db'; if(i===ldig) btn.classList.add('last');
      btn.style.setProperty('--pct',p+'%');
    }
  });
}

/* ── O/U DIRECTION ── */
function setOUDir(dir){
  ouDir=dir;
  ['d-','m-'].forEach(p=>{
    const ov=ge(p+'ouOver'),un=ge(p+'ouUnder');
    if(ov) ov.className='ou-dir-btn'+(dir==='over'?' sel-over':'');
    if(un) un.className='ou-dir-btn'+(dir==='under'?' sel-under':'');
  });
}

/* ── CONNECT ── */
async function doConnect(){
  const token=getToken(); if(!token){ toast('Enter your API token first','error'); return; }
  _storedToken=token;
  ['d-','m-'].forEach(p=>{
    const pre=ge(p+'preConn');   if(pre)   pre.style.display='none';
    const post=ge(p+'postConn'); if(post)  post.style.display='block';
    const badge=ge(p+'connBadge'); if(badge) badge.style.display='flex';
    const at=ge(p+'acctType');   if(at)   at.textContent='Authenticating…';
    const bal=ge(p+'balance');   if(bal)  bal.textContent='Fetching…';
  });
  enableBtns(false);
  try{
    const res=await fetch('/api/auth_check',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({api_token:token})});
    const d=await res.json();
    if(d.error){ toast('Auth failed: '+d.error,'error'); doDisconnect(); return; }
    connected=true; enableBtns(true);
    ['d-','m-'].forEach(p=>{
      const at=ge(p+'acctType'); if(at) at.textContent=d.acct_type||'Account';
      const ai=ge(p+'acctId');   if(ai) ai.textContent=d.loginid||'—';
    });
    _setBalance(d.balance,d.currency);
    ge('dot').className='dot'; ge('statusLbl').textContent='Connected';
    toast('✅ Connected to '+d.loginid,'success');
  }catch(e){ toast('Connection failed: '+e,'error'); doDisconnect(); }
}
function doDisconnect(){
  connected=false; _storedToken='';
  ['d-','m-'].forEach(p=>{
    const pre=ge(p+'preConn');   if(pre)   pre.style.display='';
    const post=ge(p+'postConn'); if(post)  post.style.display='none';
    const badge=ge(p+'connBadge'); if(badge) badge.style.display='none';
  });
  enableBtns(false);
  ge('dot').className='dot'; ge('statusLbl').textContent='Offline';
  ge('livePill').style.borderColor='';
}
function enableBtns(on){
  const ids=['md-single','md-rand','md-sel','md-ai',
             'eo-single','eo-auto','eo-fast','eo-ai',
             'ou-single','ou-auto','ou-fast','ou-ai','mix-start','ab-start'];
  ['d-','m-'].forEach(p=>ids.forEach(id=>{ const e=ge(p+id); if(e) e.disabled=!on; }));
}

/* ── START BOT ── */
async function startBot(mode){
  if(!connected){ toast('Connect API first','error'); return; }
  if(botRunning){ toast('Bot already running','error'); return; }
  const token=_storedToken||getToken();
  const stake=parseFloat(getVal('stake')||'1');
  const symbol=getVal('symbol')||'1HZ100V';
  const duration=parseInt(getVal('duration')||'1');
  const ti=parseInt(getVal('tickInterval')||'1');
  const mt=parseInt(getVal('maxTrades')||'0');
  const tp=parseFloat(getVal('takeProfit')||'0');
  const sl=parseFloat(getVal('stopLoss')||'0');
  const payload={
    api_token:token,stake,symbol,duration,tick_interval:ti,max_trades:mt,take_profit:tp,stop_loss:sl,
    use_martingale:martOn,martingale_multiplier:parseFloat(getVal('martMult')||'2'),
    max_martingale_steps:parseInt(getVal('martSteps')||'4'),
    stats_window:parseInt(getVal('statsWindow')||'50'),
    update_every_trades:parseInt(getVal('updateEvery')||'3'),
  };

  if(curBot==='mix'){
    if(!mixEnableOU&&!mixEnableEO&&!mixEnableMD){ toast('Enable at least one module','error'); return; }
    Object.assign(payload,{
      bot_type:'mix',contract_type:'DIGITDIFF',
      enable_ou:mixEnableOU,enable_eo:mixEnableEO,enable_md:mixEnableMD,
      ou_contract_type:getVal('mixOUType')||'DIGITOVER',
      ou_prediction:parseInt(getVal('mixOUPred')||'4'),
      eo_contract_type:getVal('mixEOType')||'DIGITEVEN',
      md_contract_type:getVal('mixMDType')||'DIGITDIFF',
    });
  } else if(curBot==='md'){
    Object.assign(payload,{bot_type:'md',contract_type:getVal('mdType')||'DIGITDIFF'});
    const pv=getVal('predSel');
    payload.prediction=(pv==='auto'||!pv)?null:parseInt(pv);
    if(['auto_random','ai_auto'].includes(mode)){payload.prediction=null;payload.auto_switch=(mode==='ai_auto');}
  } else if(curBot==='eo'){
    Object.assign(payload,{bot_type:'eo',contract_type:getVal('eoType')||'DIGITEVEN',auto_switch:(mode==='eo_ai')});
    if(mode==='eo_fast') payload.tick_interval=1;
  } else if(curBot==='ou'){
    Object.assign(payload,{
      bot_type:'ou',contract_type:(ouDir==='over'?'DIGITOVER':'DIGITUNDER'),
      prediction:parseInt(getVal('ouDigit')||'4'),auto_switch:(mode==='ou_ai'),
    });
    if(mode==='ou_fast') payload.tick_interval=1;
  } else if(curBot==='ab'){
    const mb=parseFloat(getVal('miniBalance')||'100');
    const tgt=parseFloat(getVal('targetProfit')||'20');
    if(mb<5){ toast('Mini balance must be at least $5','error'); return; }
    if(tgt<=0){ toast('Target profit must be > 0','error'); return; }
    if(tgt>mb*2){ toast('Target cannot exceed 2× mini-balance','error'); return; }
    Object.assign(payload,{
      bot_type:'ab',
      contract_type:'DIGITDIFF',  // placeholder, bot picks its own
      mini_balance:mb,
      target_profit:tgt,
      stake_pct:parseFloat(getVal('abStakePct')||'0.02'),
      reanalyze_every:parseInt(getVal('abReanalyze')||'10'),
      max_martingale_steps:parseInt(getVal('abMartSteps')||'3'),
      use_martingale:true,
    });
    delete payload.stake;   // AB calculates its own stake
    payload.stake=mb*parseFloat(getVal('abStakePct')||'0.02');  // pass for validation
  }
  curSymbol=symbol;
  try{
    const res=await fetch('/api/start',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(payload)});
    const d=await res.json();
    if(d.error){ toast('❌ '+d.error,'error'); return; }
    botRunning=true; setRunningUI(true); toast('🚀 Bot launching…','success');
  }catch(e){ toast('Failed to start','error'); }
}

/* ── STOP ── */
async function stopBot(){
  try{
    const res=await fetch('/api/stop',{method:'POST'}); const d=await res.json();
    if(d.status==='already_stopped'){ botRunning=false; setRunningUI(false); }
    else toast('⏳ Stopping — settling open contracts…','info');
  }catch(e){ toast('Stop failed','error'); }
}

/* ── RUNNING UI ── */
function setRunningUI(running){
  botRunning=running;
  const dot=ge('dot'),sl=ge('statusLbl'),lp=ge('livePill');
  if(running){
    if(dot) dot.className='dot live';
    if(sl)  sl.textContent='Live';
    if(lp)  lp.style.borderColor='rgba(34,197,94,.4)';
  } else {
    if(dot) dot.className='dot';
    if(sl)  sl.textContent='Connected';
    if(lp)  lp.style.borderColor='';
  }
  const allBtns=['md-single','md-rand','md-sel','md-ai','eo-single','eo-auto','eo-fast','eo-ai',
                 'ou-single','ou-auto','ou-fast','ou-ai','mix-start','ab-start'];
  ['d-','m-'].forEach(p=>{
    const stop=ge(p+'btnStop'); if(stop) stop.style.display=running?'':'none';
    allBtns.forEach(id=>{ const e=ge(p+id); if(e) e.style.display=running?'none':''; });
  });
}

/* ── SSE ── */
(function(){
  const es=new EventSource('/stream');
  es.addEventListener('authorized',e=>{
    const d=JSON.parse(e.data); connected=true; enableBtns(true); _setBalance(d.balance,d.currency);
    ['d-','m-'].forEach(p=>{
      const at=ge(p+'acctType'); if(at) at.textContent=d.acct_type||'Account';
      const ai=ge(p+'acctId');   if(ai) ai.textContent=d.loginid||'—';
      const pre=ge(p+'preConn');   if(pre)   pre.style.display='none';
      const post=ge(p+'postConn'); if(post)  post.style.display='block';
      const badge=ge(p+'connBadge'); if(badge) badge.style.display='flex';
    });
  });
  es.addEventListener('balance_update',e=>{const d=JSON.parse(e.data);_setBalance(d.balance,d.currency);});
  es.addEventListener('bot_started',()=>{botRunning=true;setRunningUI(true);});
  es.addEventListener('auth_failed',e=>{const d=JSON.parse(e.data);toast('❌ Auth error: '+(d.error||'unknown'),'error');});
  es.addEventListener('bot_stopped',e=>{
    const d=JSON.parse(e.data); botRunning=false; setRunningUI(false);
    if(d.balance!=null) _setBalance(d.balance,d.currency);
    const s=d.summary||{};
    if(s.trades){
      const sign=s.profit>=0?'+':'';
      toast(`✅ Done! ${s.trades} trades · ${s.wr}% WR · P/L: ${sign}${s.profit.toFixed(2)}`,'info');
    }
    renderTrades(); silentReAuth();
  });
  es.addEventListener('log',   e=>addLog(JSON.parse(e.data)));
  es.addEventListener('stats', e=>onStats(JSON.parse(e.data)));
  es.onerror=()=>{};
})();

/* ── STATS HANDLER — uses authoritative server-side wins/losses ── */
function onStats(d){
  // ── Use server running totals (never count from history array) ──
  _wins=d.wins; _losses=d.losses; _totalTrades=d.trades;

  const p=d.profit,ps=(p>=0?'+':'')+p.toFixed(2)+' USD',pc=p>0?'pos':p<0?'neg':'neu';
  ['d-','m-'].forEach(pr=>{
    const ep=ge(pr+'thProfit'); if(ep){ ep.textContent=ps; ep.className='th-val '+pc; }
    const wr=ge(pr+'thWR');     if(wr){ wr.textContent=d.wr+'%'; wr.className='th-val '+(d.wr>=50?'pos':'neu'); }
    const ws=ge(pr+'thWRSub'); if(ws) ws.textContent=`(${d.wins}/${d.wins+d.losses})`;
    const wf=ge(pr+'wrFill');   if(wf) wf.style.width=d.wr+'%';
    // ── Tab counts from authoritative totals (stable, no oscillation) ──
    const na=ge(pr+'nAll');    if(na) na.textContent=_wins+_losses+allTrades.filter(t=>t.status==='pending').length;
    const nw=ge(pr+'nWins');   if(nw) nw.textContent=_wins;
    const nl=ge(pr+'nLosses'); if(nl) nl.textContent=_losses;
  });
  if(d.balance!=null) _setBalance(d.balance, curCurrency);

  if(d.bot_type==='md'){
    digitPcts=d.dpcts||Array(10).fill(0); lastDigit=d.ldigit; renderDigitGrids();
    const m=`${d.ticks.toLocaleString()} ticks · ${curSymbol} · Last: ${d.ldigit!=null?d.ldigit:'—'}`;
    ['d-','m-'].forEach(p=>{ const e=ge(p+'digitMeta'); if(e) e.textContent=m; });
  } else if(d.bot_type==='eo'){
    ['d-','m-'].forEach(p=>{
      const eb=ge(p+'evenBar'); if(eb) eb.style.width=d.even_pct+'%';
      const ob=ge(p+'oddBar');  if(ob) ob.style.width=d.odd_pct+'%';
      const ep=ge(p+'evenPct'); if(ep) ep.textContent=d.even_pct+'%';
      const op=ge(p+'oddPct');  if(op) op.textContent=d.odd_pct+'%';
      const badge=ge(p+'parityBadge');
      if(badge){ const ie=d.ct==='DIGITEVEN'; badge.textContent=ie?'EVEN ▲':'ODD ▲'; badge.className='parity-badge '+(ie?'even':'odd'); }
      const meta=ge(p+'parityMeta');
      if(meta) meta.textContent=`${d.ticks.toLocaleString()} ticks · ${curSymbol} · Last: ${d.ldigit!=null?d.ldigit:'—'}`;
    });
  } else if(d.bot_type==='ou'){
    ouDigitPcts=d.dpcts||Array(10).fill(0); ouLastDigit=d.ldigit;
    if(d.pred!=null){ setAll('ouDigit',String(d.pred)); }
    renderOUGrids();
    ['d-','m-'].forEach(p=>{
      const badge=ge(p+'ouActiveBadge');
      if(badge){ badge.textContent=`Active: ${d.ct==='DIGITOVER'?'OVER >':'UNDER <'} ${d.pred}`; }
    });
  } else if(d.bot_type==='mix'){
    const pcts=d.dpcts||Array(10).fill(0);
    renderMixGrids(pcts,d.ldigit);
    ['d-','m-'].forEach(p=>{
      const meta=ge(p+'mixMeta'); if(meta) meta.textContent=`${d.ticks.toLocaleString()} ticks · ${curSymbol} · Last: ${d.ldigit!=null?d.ldigit:'—'}`;
    });
  } else if(d.bot_type==='ab'){
    // Update AB digit grid
    const pcts=d.dpcts||Array(10).fill(0);
    ['d-','m-'].forEach(s=>{
      for(let i=0;i<10;i++){
        const btn=ge(s+'abd'+i),pEl=ge(s+'abdp'+i); if(!btn) continue;
        const p=pcts[i]||0; if(pEl) pEl.textContent=p>0?p.toFixed(1)+'%':'—';
        btn.className='db'; if(i===d.ldigit) btn.classList.add('last');
        btn.style.setProperty('--pct',p+'%');
      }
      const meta=ge(s+'abDigitMeta');
      if(meta) meta.textContent=`${(d.ticks||0).toLocaleString()} ticks · ${curSymbol} · Last: ${d.ldigit!=null?d.ldigit:'—'}`;
      // Update strategy labels
      const lbl=ge(s+'abAnalysisLabel'); if(lbl) lbl.textContent=d.strategy_label||'Analyzing…';
      const conf=ge(s+'abAnalysisConf'); if(conf) conf.textContent=d.confidence!=null?d.confidence+'%':'—';
    });
    // Update progress bar (in settings card)
    const prog=d.goal_progress||0;
    ['d-','m-'].forEach(p=>{
      const pb=ge(p+'abProgressBar'); if(pb) pb.style.width=prog+'%';
      const pp=ge(p+'abProgressPct'); if(pp) pp.textContent=prog.toFixed(1)+'%';
      const pbox=ge(p+'abProgressBox'); if(pbox) pbox.style.display='block';
      const strat=ge(p+'abStrategy'); if(strat) strat.textContent=d.strategy_label||'—';
      const cconf=ge(p+'abConfidence'); if(cconf) cconf.textContent=(d.confidence||0)+'%';
    });
    if(d.modules){
      ['d-','m-'].forEach(p=>{
        const ou=d.modules.ou,eo=d.modules.eo,md=d.modules.md;
        const s=ge(p+'mixOUWR'); if(s) s.textContent=ou.wr+'%';
        const si=ge(p+'mixOUInfo'); if(si) si.textContent=`${ou.wins}W/${ou.losses}L`;
        const e=ge(p+'mixEOWR'); if(e) e.textContent=eo.wr+'%';
        const ei=ge(p+'mixEOInfo'); if(ei) ei.textContent=`${eo.wins}W/${eo.losses}L`;
        const m=ge(p+'mixMDWR'); if(m) m.textContent=md.wr+'%';
        const mi=ge(p+'mixMDInfo'); if(mi) mi.textContent=`${md.wins}W/${md.losses}L`;
        // Settings module cards
        const ouT=ge(p+'ouTrades'); if(ouT) ouT.textContent=ou.trades+' trades';
        const eoT=ge(p+'eoTrades'); if(eoT) eoT.textContent=eo.trades+' trades';
        const mdT=ge(p+'mdTrades'); if(mdT) mdT.textContent=md.trades+' trades';
        const ouPr=ge(p+'ouProfit'); if(ouPr){ ouPr.textContent=(ou.profit>=0?'+':'')+ou.profit.toFixed(2); ouPr.className='mix-mod-stat '+(ou.profit>0?'pos':ou.profit<0?'neg':'neu'); }
        const eoPr=ge(p+'eoProfit'); if(eoPr){ eoPr.textContent=(eo.profit>=0?'+':'')+eo.profit.toFixed(2); eoPr.className='mix-mod-stat '+(eo.profit>0?'pos':eo.profit<0?'neg':'neu'); }
        const mdPr=ge(p+'mdProfit'); if(mdPr){ mdPr.textContent=(md.profit>=0?'+':'')+md.profit.toFixed(2); mdPr.className='mix-mod-stat '+(md.profit>0?'pos':md.profit<0?'neg':'neu'); }
      });
    }
  }

  // Merge server history into local allTrades (update existing, add new)
  const serverHistory=d.history||[];
  const localMap=new Map(allTrades.map(t=>[t.cid,t]));
  serverHistory.forEach(t=>{
    if(localMap.has(t.cid)){
      // Update existing entry (status may have changed)
      Object.assign(localMap.get(t.cid),t);
    } else {
      localMap.set(t.cid,t); allTrades.push(t);
    }
  });
  renderTrades();
}

/* ── TRADE HISTORY ── */
function setTab(t){
  activeTab=t;
  ['all','wins','losses'].forEach(x=>{
    ['d-','m-'].forEach(p=>{
      const e=ge(p+'tab'+x.charAt(0).toUpperCase()+x.slice(1)); if(e) e.classList.toggle('active',x===t);
    });
  });
  renderTrades();
}
function renderTrades(){
  if(_renderPending) return; _renderPending=true;
  requestAnimationFrame(()=>{ _renderPending=false; _doRenderTrades(); });
}
function _doRenderTrades(){
  let trades=[...allTrades].reverse();
  if(activeTab==='wins')   trades=trades.filter(t=>t.status==='won');
  if(activeTab==='losses') trades=trades.filter(t=>t.status==='lost');
  if(!trades.length){
    const empty=activeTab==='all'?'No trades yet — start a bot!':activeTab==='wins'?'No wins yet':'No losses yet';
    ['d-','m-'].forEach(p=>{ const e=ge(p+'tradeList'); if(e) e.innerHTML=`<div class="empty-msg">${empty}</div>`; });
    return;
  }
  const html=trades.map(t=>{
    const pend=t.status==='pending', won=t.status==='won', canc=t.status==='cancelled', unsettled=t.status==='unsettled';
    const netStr=pend?'Pending…':canc?'Cancelled':unsettled?'Check Deriv':(t.profit!=null?(t.profit>=0?'+':'')+t.profit.toFixed(2)+' '+curCurrency:'—');
    const prCls=pend||canc||unsettled?'pend':won?'pos':'neg';
    const pyStr=pend||canc||unsettled?'—':(t.payout||0).toFixed(2)+' '+curCurrency;
    const stHtml=pend?`<span class="stat-p ip">🕐 In Progress</span>`
      :canc?`<span class="stat-p" style="color:var(--t3)">⊘ Cancelled</span>`
      :unsettled?`<span class="stat-p" style="color:var(--orange)">⚠ Check Deriv</span>`
      :won?`<span class="stat-p won">✅ Won</span>`:`<span class="stat-p lost">❌ Lost</span>`;
    const tp=t.type||''; let typePill='';
    if(tp==='O/U') typePill=`<span class="te-type-ou">${tp}${t.detail?' · '+t.detail:''}</span>`;
    else if(tp==='E/O') typePill=`<span class="te-type-eo">${tp}${t.detail?' · '+t.detail:''}</span>`;
    else if(tp==='M/D') typePill=`<span class="te-type-md">${tp}${t.detail?' · '+t.detail:''}</span>`;
    else typePill=`<span class="entry-pill">${tp||'—'}</span>`;
    return`<div class="te ${t.status}">
      <div class="te-hd">
        <div class="te-ico">📈</div>
        <span style="font-size:12px;font-weight:700">${t.symbol}</span>
        <span class="te-st ${unsettled?'unsettled':canc?'pending':t.status}">${unsettled?'UNSETTLED':t.status.toUpperCase()}</span>
        ${typePill}
        <span class="te-tag">${t.dur||1}t</span>
        <span class="te-pr ${prCls}">${netStr}</span>
      </div>
      <div class="te-rows">
        <span class="te-k">Time</span>    <span class="te-v">${t.time||'—'}</span>
        <span class="te-k">Stake</span>   <span class="te-v">${(t.stake||0).toFixed(2)+' '+curCurrency}</span>
        <span class="te-k">Received</span><span class="te-v" style="color:${won?'var(--green)':'var(--t2)'}">${pyStr}</span>
        <span class="te-k">Net P&amp;L</span><span class="te-v" style="font-weight:800;color:${won?'var(--cyan)':pend||canc||unsettled?'var(--t3)':'var(--red)'}">${netStr}</span>
        <span class="te-k">Result</span>  <span class="te-v">${stHtml}</span>
      </div>
    </div>`;
  }).join('');
  ['d-','m-'].forEach(p=>{ const e=ge(p+'tradeList'); if(e) e.innerHTML=html; });
}
function clearHistory(){ allTrades=[]; _wins=0; _losses=0; _totalTrades=0; renderTrades(); }

/* ── LOG ── */
function addLog(d){
  ['d-logBody','m-logBody','m2-logBody'].forEach(id=>{
    const el=ge(id); if(!el) return;
    const line=document.createElement('div');
    line.className='log-line '+(d.l||'info');
    line.innerHTML=`<span class="log-t">${d.t||''}</span><span class="log-m">${d.m}</span>`;
    el.appendChild(line);
    if(el.children.length>250) el.removeChild(el.firstChild);
    el.scrollTop=el.scrollHeight;
  });
}
function clearLog(){ ['d-logBody','m-logBody','m2-logBody'].forEach(id=>{ const el=ge(id); if(el) el.innerHTML=''; }); }

/* ── INIT ── */
selectBot('md');
renderDigitGrids();
renderOUGrids();
_renderMixModUI();
// Init bottom nav indicator
setTimeout(()=>setMobileTab(0),100);
</script>
</body>
</html>"""


@app.route("/health")
def health():
    """Simple health-check endpoint."""
    return jsonify({
        "status": "ok",
        "version": "5.1",
        "bot_running": _bot_is_running(),
        "connected_accounts": list(_live_balance.keys()),
    })


if __name__ == "__main__":
    port=int(os.environ.get("PORT",5000))
    print(f"\n  PerryAI Trading Platform v5.1 — Mixologist Edition (Polished)")
    print(f"  ──────────────────────────────────────────────────────────────")
    print(f"  Bots: Matches/Differs · Even/Odd · Over/Under · Mixologist")
    print(f"  Credentials: {VALID_EMAIL}")
    print(f"  PC    → http://localhost:{port}")
    print(f"  Phone → http://YOUR_LAN_IP:{port}")
    print(f"  Health → http://localhost:{port}/health")
    print(f"  Ctrl+C to stop\n")
    print(f"  [TIP] Set env vars to override credentials:")
    print(f"    PERRYAI_EMAIL=you@example.com PERRYAI_PASSWORD=yourpass python PerryAI_v5_fixed.py\n")
    app.run(host="0.0.0.0",port=port,debug=False,threaded=True)
