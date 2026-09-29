import discord
from discord.ext import commands
from discord import app_commands
import asyncio
import aiosqlite
import random
import time
import os
from dotenv import load_dotenv

# ================= CONFIG =================
load_dotenv()
TOKEN = os.getenv("DISCORD_TOKEN")

OWNER_IDS = [1500448865873170535]   # <-- your ID
PREFIX = ","
STARTING_WALLET = 100
STARTING_BANK = 0
DAILY_REWARD = 500
DAILY_COOLDOWN_HOURS = 24
ROB_COOLDOWN_HOURS = 1
ROB_SUCCESS_CHANCE = 0.5
ROB_FAIL_FINE_PERCENT = 0.25
CLAN_CREATE_COST = 5000
CLAN_MAX_MEMBERS = 20
DB_PATH = "gamblebot.db"

# ================= DATABASE =================
async def init_db():
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER,
                guild_id INTEGER,
                wallet INTEGER DEFAULT 0,
                bank INTEGER DEFAULT 0,
                last_daily REAL DEFAULT 0,
                last_rob REAL DEFAULT 0,
                PRIMARY KEY (user_id, guild_id)
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS clans (
                clan_id INTEGER PRIMARY KEY AUTOINCREMENT,
                guild_id INTEGER,
                name TEXT,
                owner_id INTEGER,
                description TEXT DEFAULT '',
                bank INTEGER DEFAULT 0,
                created_at REAL,
                UNIQUE(guild_id, name)
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS clan_members (
                user_id INTEGER,
                guild_id INTEGER,
                clan_id INTEGER,
                rank TEXT DEFAULT 'member',
                contribution INTEGER DEFAULT 0,
                PRIMARY KEY (user_id, guild_id)
            )
        """)
        await db.execute("""
            CREATE TABLE IF NOT EXISTS clan_invites (
                user_id INTEGER,
                guild_id INTEGER,
                clan_id INTEGER,
                PRIMARY KEY (user_id, guild_id)
            )
        """)
        await db.commit()

async def ensure_user(uid, gid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR IGNORE INTO users (user_id, guild_id, wallet, bank) VALUES (?, ?, ?, ?)",
            (uid, gid, STARTING_WALLET, STARTING_BANK)
        )
        await db.commit()

async def get_user(uid, gid):
    await ensure_user(uid, gid)
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT wallet, bank, last_daily, last_rob FROM users WHERE user_id=? AND guild_id=?",
            (uid, gid)
        ) as cur:
            row = await cur.fetchone()
            return {"wallet": row[0], "bank": row[1], "last_daily": row[2], "last_rob": row[3]}

async def set_balance(uid, gid, kind, amount, add=False):
    await ensure_user(uid, gid)
    async with aiosqlite.connect(DB_PATH) as db:
        if add:
            await db.execute(f"UPDATE users SET {kind} = {kind} + ? WHERE user_id=? AND guild_id=?",
                             (amount, uid, gid))
        else:
            await db.execute(f"UPDATE users SET {kind} = ? WHERE user_id=? AND guild_id=?",
                             (amount, uid, gid))
        await db.commit()

async def update_cooldown(uid, gid, kind, ts):
    await ensure_user(uid, gid)
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(f"UPDATE users SET {kind} = ? WHERE user_id=? AND guild_id=?",
                         (ts, uid, gid))
        await db.commit()

async def get_leaderboard(gid, kind="total", limit=10):
    async with aiosqlite.connect(DB_PATH) as db:
        order = {"wallet": "wallet DESC", "bank": "bank DESC"}.get(kind, "(wallet + bank) DESC")
        async with db.execute(
            f"SELECT user_id, wallet, bank FROM users WHERE guild_id=? ORDER BY {order} LIMIT ?",
            (gid, limit)
        ) as cur:
            return await cur.fetchall()

# ----- clan helpers -----
async def create_clan(gid, name, owner_id, ts):
    async with aiosqlite.connect(DB_PATH) as db:
        try:
            cur = await db.execute(
                "INSERT INTO clans (guild_id, name, owner_id, created_at) VALUES (?, ?, ?, ?)",
                (gid, name, owner_id, ts)
            )
            cid = cur.lastrowid
            await db.execute(
                "INSERT INTO clan_members (user_id, guild_id, clan_id, rank) VALUES (?, ?, ?, 'owner')",
                (owner_id, gid, cid)
            )
            await db.commit()
            return cid
        except aiosqlite.IntegrityError:
            return None

async def get_clan_by_name(gid, name):
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT clan_id, guild_id, name, owner_id, description, bank FROM clans WHERE guild_id=? AND name=? COLLATE NOCASE",
            (gid, name)
        ) as cur:
            r = await cur.fetchone()
            return {"clan_id": r[0], "guild_id": r[1], "name": r[2], "owner_id": r[3],
                    "description": r[4], "bank": r[5]} if r else None

async def get_clan_by_id(cid):
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT clan_id, guild_id, name, owner_id, description, bank FROM clans WHERE clan_id=?",
            (cid,)
        ) as cur:
            r = await cur.fetchone()
            return {"clan_id": r[0], "guild_id": r[1], "name": r[2], "owner_id": r[3],
                    "description": r[4], "bank": r[5]} if r else None

async def get_user_clan(uid, gid):
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT clan_id, rank, contribution FROM clan_members WHERE user_id=? AND guild_id=?",
            (uid, gid)
        ) as cur:
            r = await cur.fetchone()
            return {"clan_id": r[0], "rank": r[1], "contribution": r[2]} if r else None

async def get_clan_members(cid):
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT user_id, rank, contribution FROM clan_members WHERE clan_id=?",
            (cid,)
        ) as cur:
            return await cur.fetchall()

async def add_clan_member(uid, gid, cid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR REPLACE INTO clan_members (user_id, guild_id, clan_id, rank, contribution) VALUES (?, ?, ?, 'member', 0)",
            (uid, gid, cid)
        )
        await db.commit()

async def remove_clan_member(uid, gid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("DELETE FROM clan_members WHERE user_id=? AND guild_id=?", (uid, gid))
        await db.commit()

async def set_clan_rank(uid, gid, rank):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE clan_members SET rank=? WHERE user_id=? AND guild_id=?",
                         (rank, uid, gid))
        await db.commit()

async def add_clan_bank(cid, amount):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE clans SET bank = bank + ? WHERE clan_id=?", (amount, cid))
        await db.commit()

async def set_clan_description(cid, desc):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE clans SET description=? WHERE clan_id=?", (desc, cid))
        await db.commit()

async def update_clan_owner(cid, new_owner):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE clans SET owner_id=? WHERE clan_id=?", (new_owner, cid))
        await db.commit()

async def delete_clan(cid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("DELETE FROM clans WHERE clan_id=?", (cid,))
        await db.execute("DELETE FROM clan_members WHERE clan_id=?", (cid,))
        await db.execute("DELETE FROM clan_invites WHERE clan_id=?", (cid,))
        await db.commit()

async def add_contribution(uid, gid, amount):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "UPDATE clan_members SET contribution = contribution + ? WHERE user_id=? AND guild_id=?",
            (amount, uid, gid)
        )
        await db.commit()

async def list_clans(gid):
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            """SELECT c.clan_id, c.name, c.bank, c.owner_id,
               (SELECT COUNT(*) FROM clan_members WHERE clan_id = c.clan_id) as members
               FROM clans c WHERE c.guild_id=? ORDER BY c.bank DESC""",
            (gid,)
        ) as cur:
            return await cur.fetchall()

async def add_invite(uid, gid, cid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute(
            "INSERT OR REPLACE INTO clan_invites (user_id, guild_id, clan_id) VALUES (?, ?, ?)",
            (uid, gid, cid)
        )
        await db.commit()

async def get_invite(uid, gid):
    async with aiosqlite.connect(DB_PATH) as db:
        async with db.execute(
            "SELECT clan_id FROM clan_invites WHERE user_id=? AND guild_id=?",
            (uid, gid)
        ) as cur:
            r = await cur.fetchone()
            return r[0] if r else None

async def remove_invite(uid, gid):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("DELETE FROM clan_invites WHERE user_id=? AND guild_id=?", (uid, gid))
        await db.commit()

# ================= BOT SETUP =================
intents = discord.Intents.all()
bot = commands.Bot(command_prefix=PREFIX, intents=intents, help_command=None)

def is_owner():
    async def predicate(ctx):
        return ctx.author.id in OWNER_IDS
    return commands.check(predicate)

def owner_only():
    async def predicate(interaction: discord.Interaction):
        return interaction.user.id in OWNER_IDS
    return app_commands.check(predicate)

@bot.event
async def on_ready():
    print(f"✅ Logged in as {bot.user} (ID: {bot.user.id})")
    try:
        synced = await bot.tree.sync()
        print(f"✅ Synced {len(synced)} slash commands")
    except Exception as e:
        print(f"❌ Sync error: {e}")
    await bot.change_presence(activity=discord.Game(name=",help"))

@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.MissingRequiredArgument):
        await ctx.send(f"❌ Missing argument: `{error.param.name}`")
    elif isinstance(error, commands.BadArgument):
        await ctx.send("❌ Invalid argument.")
    elif isinstance(error, commands.CommandNotFound):
        return
    elif isinstance(error, commands.CheckFailure):
        await ctx.send("❌ You don't have permission.")
    else:
        await ctx.send(f"❌ Error: `{error}`")# ================= ECONOMY COMMANDS =================
@bot.command(name="balance", aliases=["bal"])
async def balance(ctx):
    u = await get_user(ctx.author.id, ctx.guild.id)
    e = discord.Embed(title=f"💰 {ctx.author.display_name}'s Balance", color=0x2ecc71)
    e.add_field(name="💵 Wallet", value=f"**{u['wallet']:,}**", inline=True)
    e.add_field(name="🏦 Bank", value=f"**{u['bank']:,}**", inline=True)
    e.add_field(name="📊 Total", value=f"**{u['wallet']+u['bank']:,}**", inline=False)
    await ctx.send(embed=e)

@bot.tree.command(name="balance", description="View your balance")
async def balance_slash(interaction: discord.Interaction):
    u = await get_user(interaction.user.id, interaction.guild.id)
    e = discord.Embed(title=f"💰 {interaction.user.display_name}'s Balance", color=0x2ecc71)
    e.add_field(name="💵 Wallet", value=f"**{u['wallet']:,}**", inline=True)
    e.add_field(name="🏦 Bank", value=f"**{u['bank']:,}**", inline=True)
    e.add_field(name="📊 Total", value=f"**{u['wallet']+u['bank']:,}**", inline=False)
    await interaction.response.send_message(embed=e)

async def _daily(uid, gid):
    user = await get_user(uid, gid)
    now = time.time()
    cd = DAILY_COOLDOWN_HOURS * 3600
    rem = user["last_daily"] + cd - now
    if rem > 0:
        h, m = int(rem // 3600), int((rem % 3600) // 60)
        return None, f"⏳ Already claimed! Try again in **{h}h {m}m**."
    await update_cooldown(uid, gid, "last_daily", now)
    await set_balance(uid, gid, "wallet", DAILY_REWARD, add=True)
    return DAILY_REWARD, None

@bot.command(name="daily")
async def daily(ctx):
    amt, err = await _daily(ctx.author.id, ctx.guild.id)
    if err: return await ctx.send(err)
    e = discord.Embed(title="🎁 Daily Reward!", description=f"You received **{amt:,}** coins!", color=0xf1c40f)
    await ctx.send(embed=e)

@bot.tree.command(name="daily", description="Claim your daily reward")
async def daily_slash(interaction: discord.Interaction):
    amt, err = await _daily(interaction.user.id, interaction.guild.id)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    e = discord.Embed(title="🎁 Daily Reward!", description=f"You received **{amt:,}** coins!", color=0xf1c40f)
    await interaction.response.send_message(embed=e)

@bot.command(name="leaderboard", aliases=["lb"])
async def leaderboard(ctx, kind: str = "total"):
    kind = kind.lower()
    if kind not in ("wallet", "bank", "total"):
        return await ctx.send("❌ Use: `,lb [wallet|bank|total]`")
    rows = await get_leaderboard(ctx.guild.id, kind)
    e = discord.Embed(title=f"🏆 Leaderboard — {kind.title()}", color=0xf39c12)
    medals = ["🥇", "🥈", "🥉"]
    desc = ""
    for i, (uid, w, b) in enumerate(rows):
        m = ctx.guild.get_member(uid)
        name = m.display_name if m else f"User {uid}"
        val = w if kind == "wallet" else (b if kind == "bank" else w + b)
        pre = medals[i] if i < 3 else f"`#{i+1}`"
        desc += f"{pre} **{name}** — {val:,}\n"
    e.description = desc or "No data yet."
    await ctx.send(embed=e)

@bot.tree.command(name="leaderboard", description="Top 10 richest users")
@app_commands.choices(kind=[
    app_commands.Choice(name="Total", value="total"),
    app_commands.Choice(name="Wallet", value="wallet"),
    app_commands.Choice(name="Bank", value="bank"),
])
async def lb_slash(interaction: discord.Interaction, kind: str = "total"):
    rows = await get_leaderboard(interaction.guild.id, kind)
    e = discord.Embed(title=f"🏆 Leaderboard — {kind.title()}", color=0xf39c12)
    medals = ["🥇", "🥈", "🥉"]
    desc = ""
    for i, (uid, w, b) in enumerate(rows):
        m = interaction.guild.get_member(uid)
        name = m.display_name if m else f"User {uid}"
        val = w if kind == "wallet" else (b if kind == "bank" else w + b)
        pre = medals[i] if i < 3 else f"`#{i+1}`"
        desc += f"{pre} **{name}** — {val:,}\n"
    e.description = desc or "No data yet."
    await interaction.response.send_message(embed=e)

# ================= BANK COMMANDS =================
async def _dep(uid, gid, amount_str):
    user = await get_user(uid, gid)
    if amount_str.lower() == "all":
        amount = user["wallet"]
    else:
        try: amount = int(amount_str)
        except ValueError: return None, "❌ Invalid amount."
    if amount <= 0: return None, "❌ Amount must be > 0."
    if amount > user["wallet"]:
        return None, f"❌ You only have **{user['wallet']:,}** in your wallet."
    await set_balance(uid, gid, "wallet", -amount, add=True)
    await set_balance(uid, gid, "bank", amount, add=True)
    return amount, None

async def _with(uid, gid, amount_str):
    user = await get_user(uid, gid)
    if amount_str.lower() == "all":
        amount = user["bank"]
    else:
        try: amount = int(amount_str)
        except ValueError: return None, "❌ Invalid amount."
    if amount <= 0: return None, "❌ Amount must be > 0."
    if amount > user["bank"]:
        return None, f"❌ You only have **{user['bank']:,}** in your bank."
    await set_balance(uid, gid, "bank", -amount, add=True)
    await set_balance(uid, gid, "wallet", amount, add=True)
    return amount, None

@bot.command(name="dep", aliases=["deposit"])
async def dep(ctx, amount: str):
    amt, err = await _dep(ctx.author.id, ctx.guild.id, amount)
    if err: return await ctx.send(err)
    e = discord.Embed(title="🏦 Deposited", description=f"Moved **{amt:,}** to your bank.", color=0x2ecc71)
    await ctx.send(embed=e)

@bot.tree.command(name="deposit", description="Move coins from wallet to bank")
async def dep_slash(interaction: discord.Interaction, amount: str):
    amt, err = await _dep(interaction.user.id, interaction.guild.id, amount)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    e = discord.Embed(title="🏦 Deposited", description=f"Moved **{amt:,}** to your bank.", color=0x2ecc71)
    await interaction.response.send_message(embed=e)

@bot.command(name="with", aliases=["withdraw"])
async def with_cmd(ctx, amount: str):
    amt, err = await _with(ctx.author.id, ctx.guild.id, amount)
    if err: return await ctx.send(err)
    e = discord.Embed(title="💵 Withdrew", description=f"Moved **{amt:,}** to your wallet.", color=0x3498db)
    await ctx.send(embed=e)

@bot.tree.command(name="withdraw", description="Move coins from bank to wallet")
async def with_slash(interaction: discord.Interaction, amount: str):
    amt, err = await _with(interaction.user.id, interaction.guild.id, amount)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    e = discord.Embed(title="💵 Withdrew", description=f"Moved **{amt:,}** to your wallet.", color=0x3498db)
    await interaction.response.send_message(embed=e)

# ================= TRANSFER =================
async def _give(giver_id, gid, member, amount):
    if member.id == giver_id: return "❌ Can't give to yourself."
    if member.bot: return "❌ Can't give to bots."
    if amount <= 0: return "❌ Amount must be > 0."
    giver = await get_user(giver_id, gid)
    if giver["wallet"] < amount:
        return f"❌ You only have **{giver['wallet']:,}** in your wallet."
    await set_balance(giver_id, gid, "wallet", -amount, add=True)
    await set_balance(member.id, gid, "wallet", amount, add=True)
    return None

@bot.command(name="give")
async def give(ctx, member: discord.Member, amount: int):
    err = await _give(ctx.author.id, ctx.guild.id, member, amount)
    if err: return await ctx.send(err)
    e = discord.Embed(title="💸 Transfer Complete",
                      description=f"You gave **{amount:,}** to {member.mention}!", color=0x9b59b6)
    await ctx.send(embed=e)

@bot.tree.command(name="give", description="Give your wallet coins to another member")
async def give_slash(interaction: discord.Interaction, member: discord.Member, amount: int):
    err = await _give(interaction.user.id, interaction.guild.id, member, amount)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    e = discord.Embed(title="💸 Transfer Complete",
                      description=f"You gave **{amount:,}** to {member.mention}!", color=0x9b59b6)
    await interaction.response.send_message(embed=e)

# ================= GAMBLING =================
WINS = ["🎉 Big win!", "💰 Easy money!", "🔥 On fire!", "😎 Too easy."]
LOSSES = ["💀 Ouch.", "😭 Better luck next time.", "📉 Painful.", "🪦 F in chat."]

async def _check_bet(uid, gid, bet):
    if bet <= 0: return "❌ Bet must be > 0."
    u = await get_user(uid, gid)
    if u["wallet"] < bet:
        return f"❌ You only have **{u['wallet']:,}** in your wallet. Use `,with` to move from bank."
    return None

async def _apply(uid, gid, delta):
    await set_balance(uid, gid, "wallet", delta, add=True)

# --- gamble 50/50 ---
async def _gamble_logic(uid, gid, bet):
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    if random.random() < 0.5:
        await _apply(uid, gid, bet)
        e = discord.Embed(title="🎰 Ladder Gamble — WIN!",
                          description=f"{random.choice(WINS)}\nYou won **{bet:,}**!", color=0x2ecc71)
    else:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="🎰 Ladder Gamble — LOSS",
                          description=f"{random.choice(LOSSES)}\nYou lost **{bet:,}**.", color=0xe74c3c)
    return e, None

@bot.command(name="gamble")
async def gamble(ctx, bet: int):
    e, err = await _gamble_logic(ctx.author.id, ctx.guild.id, bet)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="gamble", description="50/50 ladder gamble — win double or lose it all")
async def gamble_slash(interaction: discord.Interaction, bet: int):
    e, err = await _gamble_logic(interaction.user.id, interaction.guild.id, bet)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- coinflip ---
async def _coinflip_logic(uid, gid, bet, choice):
    choice = choice.lower()
    if choice not in ("heads", "tails", "h", "t"):
        return None, "❌ Choose `heads` or `tails`."
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    result = random.choice(["heads", "tails"])
    won = (choice.startswith("h") and result == "heads") or (choice.startswith("t") and result == "tails")
    if won:
        await _apply(uid, gid, bet)
        e = discord.Embed(title="🪙 Coinflip — WIN!",
                          description=f"Landed on **{result}**!\n{random.choice(WINS)} +**{bet:,}**", color=0x2ecc71)
    else:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="🪙 Coinflip — LOSS",
                          description=f"Landed on **{result}**.\n{random.choice(LOSSES)} -**{bet:,}**", color=0xe74c3c)
    return e, None

@bot.command(name="coinflip")
async def coinflip(ctx, bet: int, choice: str):
    e, err = await _coinflip_logic(ctx.author.id, ctx.guild.id, bet, choice)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="coinflip", description="Bet on heads or tails")
async def coinflip_slash(interaction: discord.Interaction, bet: int, choice: str):
    e, err = await _coinflip_logic(interaction.user.id, interaction.guild.id, bet, choice)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- dice ---
async def _dice_logic(uid, gid, bet, guess):
    if guess < 1 or guess > 6:
        return None, "❌ Guess must be 1–6."
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    roll = random.randint(1, 6)
    if roll == guess:
        win = bet * 5
        await _apply(uid, gid, win)
        e = discord.Embed(title="🎲 Dice — WIN!",
                          description=f"Rolled **{roll}** (you guessed {guess})!\n{random.choice(WINS)} +**{win:,}**", color=0x2ecc71)
    else:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="🎲 Dice — LOSS",
                          description=f"Rolled **{roll}** (you guessed {guess}).\n{random.choice(LOSSES)} -**{bet:,}**", color=0xe74c3c)
    return e, None

@bot.command(name="dice")
async def dice(ctx, bet: int, guess: int):
    e, err = await _dice_logic(ctx.author.id, ctx.guild.id, bet, guess)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="dice", description="Roll a die — guess 1-6 for 5x")
async def dice_slash(interaction: discord.Interaction, bet: int, guess: int):
    e, err = await _dice_logic(interaction.user.id, interaction.guild.id, bet, guess)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- slots ---
async def _slots_logic(uid, gid, bet):
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    symbols = ["🍒", "🍋", "🔔", "⭐", "💎", "7️⃣"]
    reels = [random.choice(symbols) for _ in range(3)]
    display = " | ".join(reels)
    if reels[0] == reels[1] == reels[2]:
        if reels[0] == "7️⃣":
            win = bet * 25
            title = "🎰 JACKPOT!!! 777"
        else:
            win = bet * 10
            title = "🎰 Triple Match!"
        await _apply(uid, gid, win)
        e = discord.Embed(title=title, description=f"{display}\n{random.choice(WINS)} +**{win:,}**", color=0xf1c40f)
    elif reels[0] == reels[1] or reels[1] == reels[2] or reels[0] == reels[2]:
        win = bet * 2
        await _apply(uid, gid, win)
        e = discord.Embed(title="🎰 Pair Match!",
                          description=f"{display}\n+**{win:,}**", color=0x2ecc71)
    else:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="🎰 Slots — LOSS",
                          description=f"{display}\n{random.choice(LOSSES)} -**{bet:,}**", color=0xe74c3c)
    return e, None

@bot.command(name="slots")
async def slots(ctx, bet: int):
    e, err = await _slots_logic(ctx.author.id, ctx.guild.id, bet)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="slots", description="Spin the slot machine")
async def slots_slash(interaction: discord.Interaction, bet: int):
    e, err = await _slots_logic(interaction.user.id, interaction.guild.id, bet)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- russian roulette ---
async def _roulette_logic(uid, gid, bet):
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    if random.randint(1, 6) == 1:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="🔫 BANG! 💀",
                          description=f"The chamber was loaded... You lost **{bet:,}**.", color=0xe74c3c)
    else:
        win = bet * 3
        await _apply(uid, gid, win)
        e = discord.Embed(title="🔫 *click*",
                          description=f"Empty chamber! You survived and won **{win:,}**!", color=0x2ecc71)
    return e, None

@bot.command(name="roulette")
async def roulette(ctx, bet: int):
    e, err = await _roulette_logic(ctx.author.id, ctx.guild.id, bet)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="roulette", description="Russian Roulette — 1/6 to lose, 5/6 to win 3x")
async def roulette_slash(interaction: discord.Interaction, bet: int):
    e, err = await _roulette_logic(interaction.user.id, interaction.guild.id, bet)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- wheel ---
async def _wheel_logic(uid, gid, bet, color):
    color = color.lower()
    if color not in ("red", "black", "green"):
        return None, "❌ Choose `red`, `black`, or `green`."
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    result = random.choices(["red", "black", "green"], weights=[45, 45, 10])[0]
    if result == color:
        mult = 14 if color == "green" else 2
        win = bet * mult
        await _apply(uid, gid, win)
        e = discord.Embed(title="🎡 Wheel — WIN!",
                          description=f"Landed on **{result}**! +**{win:,}** ({mult}x)", color=0x2ecc71)
    else:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="🎡 Wheel — LOSS",
                          description=f"Landed on **{result}**. -**{bet:,}**", color=0xe74c3c)
    return e, None

@bot.command(name="wheel")
async def wheel(ctx, bet: int, color: str):
    e, err = await _wheel_logic(ctx.author.id, ctx.guild.id, bet, color)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="wheel", description="Bet on red/black (2x) or green (14x)")
async def wheel_slash(interaction: discord.Interaction, bet: int, color: str):
    e, err = await _wheel_logic(interaction.user.id, interaction.guild.id, bet, color)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- crash ---
async def _crash_logic(uid, gid, bet, cashout):
    if cashout < 1.01:
        return None, "❌ Cashout must be ≥ 1.01"
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    crash_point = round(random.uniform(1.0, 10.0), 2)
    if crash_point >= cashout:
        win = int(bet * cashout)
        await _apply(uid, gid, win)
        e = discord.Embed(title="🚀 Crash — WIN!",
                          description=f"Crashed at **{crash_point}x** (you cashed out at {cashout}x)\n+**{win:,}**", color=0x2ecc71)
    else:
        await _apply(uid, gid, -bet)
        e = discord.Embed(title="💥 Crash — LOSS",
                          description=f"Crashed at **{crash_point}x** (you wanted {cashout}x)\n-**{bet:,}**", color=0xe74c3c)
    return e, None

@bot.command(name="crash")
async def crash(ctx, bet: int, cashout: float):
    e, err = await _crash_logic(ctx.author.id, ctx.guild.id, bet, cashout)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="crash", description="Multiplier climbs — cash out before it crashes")
async def crash_slash(interaction: discord.Interaction, bet: int, cashout: float):
    e, err = await _crash_logic(interaction.user.id, interaction.guild.id, bet, cashout)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    await interaction.response.send_message(embed=e)

# --- highlow ---
async def _highlow_logic(uid, gid, bet):
    err = await _check_bet(uid, gid, bet)
    if err: return None, err
    first = random.randint(1, 100)
    second = random.randint(1, 100)
    while second == first:
        second = random.randint(1, 100)
    correct = "higher" if second > first else "lower"
    e = discord.Embed(title="📊 High-Low",
                      description=f"First number: **{first}**\nSecond number: **{second}**\nCorrect: **{correct}**",
                      color=0x3498db)
    e.set_footer(text="For prefix: guess was random — use /highlow to play interactively")
    # simple auto-win chance for prefix mode
    if random.random() < 0.5:
        await _apply(uid, gid, bet)
        e.color = 0x2ecc71
        e.add_field(name="Result", value=f"WIN +**{bet:,}**")
    else:
        await _apply(uid, gid, -bet)
        e.color = 0xe74c3c
        e.add_field(name="Result", value=f"LOSS -**{bet:,}**")
    return e, None

@bot.command(name="highlow")
async def highlow(ctx, bet: int):
    e, err = await _highlow_logic(ctx.author.id, ctx.guild.id, bet)
    if err: return await ctx.send(err)
    await ctx.send(embed=e)

@bot.tree.command(name="highlow", description="Guess higher/lower — 50/50 for 2x")
async def highlow_slash(interaction: discord.Interaction, bet: int, choice: str):
    choice = choice.lower()
    if choice not in ("higher", "lower", "h", "l"):
        return await interaction.response.send_message("❌ Choose `higher` or `lower`.", ephemeral=True)
    err = await _check_bet(interaction.user.id, interaction.guild.id, bet)
    if err: return await interaction.response.send_message(err, ephemeral=True)
    first = random.randint(1, 100)
    second = random.randint(1, 100)
    while second == first:
        second = random.randint(1, 100)
    correct = "higher" if second > first else "lower"
    won = choice.startswith(correct[0])
    if won:
        await _apply(interaction.user.id, interaction.guild.id, bet)
        e = discord.Embed(title="📊 High-Low — WIN!",
                          description=f"{first} → {second} (you said {choice})\n+**{bet:,}**", color=0x2ecc71)
    else:
        await _apply(interaction.user.id, interaction.guild.id, -bet)
        e = discord.Embed(title="📊 High-Low — LOSS",
                          description=f"{first} → {second} (you said {choice})\n-**{bet:,}**", color=0xe74c3c)
    await interaction.response.send_message(embed=e)

# ================= ROB =================
async def _rob(robber_id, gid, target, guild):
    if target.id == robber_id:
        return "❌ Can't rob yourself."
    if target.bot:
        return "❌ Can't rob bots."
    robber = await get_user(robber_id, gid)
    now = time.time()
    cd = ROB_COOLDOWN_HOURS * 3600
    rem = robber["last_rob"] + cd - now
    if rem > 0:
        h, m = int(rem // 3600), int((rem % 3600) // 60)
        return f"⏳ Rob on cooldown! Try again in **{h}h {m}m**."
    victim = await get_user(target.id, gid)
    if victim["wallet"] <= 0:
        await update_cooldown(robber_id, gid, "last_rob", now)
        return f"🥷 You tried to rob {target.mention} but their wallet was empty!"
    await update_cooldown(robber_id, gid, "last_rob", now)
    if random.random() < ROB_SUCCESS_CHANCE:
        stolen = victim["wallet"] // 2
        await set_balance(target.id, gid, "wallet", -stolen, add=True)
        await set_balance(robber_id, gid, "wallet", stolen, add=True)
        e = discord.Embed(title="🥷 Rob Successful!",
                          description=f"You stole **{stolen:,}** from {target.mention}!", color=0x2ecc71)
        return e
    else:
        fine = int(robber["wallet"] * ROB_FAIL_FINE_PERCENT)
        await set_balance(robber_id, gid, "wallet", -fine, add=True)
        e = discord.Embed(title="🚨 Rob Failed!",
                          description=f"You got caught and paid a **{fine:,}** fine!", color=0xe74c3c)
        return e

@bot.command(name="rob")
async def rob(ctx, target: discord.Member):
    result = await _rob(ctx.author.id, ctx.guild.id, target, ctx.guild)
    if isinstance(result, str):
        return await ctx.send(result)
    await ctx.send(embed=result)

@bot.tree.command(name="rob", description="Try to steal half of someone's wallet")
async def rob_slash(interaction: discord.Interaction, target: discord.Member):
    result = await _rob(interaction.user.id, interaction.guild.id, target, interaction.guild)
    if isinstance(result, str):
        return await interaction.response.send_message(result, ephemeral=True)
    await interaction.response.send_message(embed=result)# ================= CLAN SYSTEM =================
@bot.command(name="clan")
async def clan_cmd(ctx, sub: str = None, *, args: str = ""):
    await _clan_router(ctx, ctx.author.id, ctx.guild.id, ctx.guild, sub, args, is_slash=False)

@bot.tree.command(name="clan", description="Clan system")
@app_commands.describe(sub="create|disband|invite|join|accept|leave|kick|promote|info|list|top|deposit|withdraw|bank|setdesc")
async def clan_slash(interaction: discord.Interaction, sub: str, arg: str = None):
    await _clan_router(interaction, interaction.user.id, interaction.guild.id, interaction.guild, sub, arg or "", is_slash=True)

async def _clan_router(ctx_or_inter, uid, gid, guild, sub, args, is_slash):
    async def send(content=None, embed=None, ephemeral=False):
        if is_slash:
            if ctx_or_inter.response.is_done():
                await ctx_or_inter.followup.send(content=content, embed=embed, ephemeral=ephemeral)
            else:
                await ctx_or_inter.response.send_message(content=content, embed=embed, ephemeral=ephemeral)
        else:
            await ctx_or_inter.send(content=content, embed=embed)

    if not sub:
        return await send("❌ Use `,clan <sub>` — try `,clan list`", ephemeral=True)

    sub = sub.lower()

    # ---- CREATE ----
    if sub == "create":
        name = args.strip()
        if not name or len(name) < 3 or len(name) > 20:
            return await send("❌ Name must be 3–20 characters.", ephemeral=True)
        existing = await get_user_clan(uid, gid)
        if existing:
            return await send("❌ You're already in a clan.", ephemeral=True)
        user = await get_user(uid, gid)
        if user["wallet"] < CLAN_CREATE_COST:
            return await send(f"❌ Need **{CLAN_CREATE_COST:,}** in wallet to create a clan.", ephemeral=True)
        cid = await create_clan(gid, name, uid, time.time())
        if not cid:
            return await send("❌ That clan name is already taken.", ephemeral=True)
        await set_balance(uid, gid, "wallet", -CLAN_CREATE_COST, add=True)
        e = discord.Embed(title="🏰 Clan Created!",
                          description=f"**{name}** is now yours!\nCost: **{CLAN_CREATE_COST:,}**", color=0xf1c40f)
        return await send(embed=e)

    # ---- DISBAND ----
    if sub == "disband":
        uc = await get_user_clan(uid, gid)
        if not uc:
            return await send("❌ You're not in a clan.", ephemeral=True)
        if uc["rank"] != "owner":
            return await send("❌ Only the clan owner can disband.", ephemeral=True)
        c = await get_clan_by_id(uc["clan_id"])
        await set_balance(uid, gid, "wallet", c["bank"], add=True)
        await delete_clan(uc["clan_id"])
        e = discord.Embed(title="🏰 Clan Disbanded",
                          description=f"**{c['name']}** was disbanded.\nVault (**{c['bank']:,}**) returned to you.", color=0xe74c3c)
        return await send(embed=e)

    # ---- INVITE ----
    if sub == "invite":
        if not args:
            return await send("❌ Use `,clan invite @user`", ephemeral=True)
        uc = await get_user_clan(uid, gid)
        if not uc:
            return await send("❌ You're not in a clan.", ephemeral=True)
        if uc["rank"] not in ("owner", "officer"):
            return await send("❌ Only owners/officers can invite.", ephemeral=True)
        try:
            target_id = int(args.strip("<@!>"))
        except ValueError:
            return await send("❌ Mention a user.", ephemeral=True)
        target_member = guild.get_member(target_id)
        if not target_member:
            return await send("❌ User not found.", ephemeral=True)
        existing = await get_user_clan(target_id, gid)
        if existing:
            return await send("❌ That user is already in a clan.", ephemeral=True)
        c = await get_clan_by_id(uc["clan_id"])
        members = await get_clan_members(uc["clan_id"])
        if len(members) >= CLAN_MAX_MEMBERS:
            return await send("❌ Clan is full.", ephemeral=True)
        await add_invite(target_id, gid, uc["clan_id"])
        e = discord.Embed(title="📨 Invite Sent",
                          description=f"{target_member.mention} was invited to **{c['name']}**.\nThey can accept with `,clan accept {c['name']}`", color=0x3498db)
        return await send(embed=e)

    # ---- ACCEPT ----
    if sub == "accept":
        cid = await get_invite(uid, gid)
        if not cid:
            return await send("❌ You have no pending invite.", ephemeral=True)
        existing = await get_user_clan(uid, gid)
        if existing:
            return await send("❌ You're already in a clan.", ephemeral=True)
        c = await get_clan_by_id(cid)
        await add_clan_member(uid, gid, cid)
        await remove_invite(uid, gid)
        e = discord.Embed(title="🏰 Clan Joined!",
                          description=f"You joined **{c['name']}**!", color=0x2ecc71)
        return await send(embed=e)

    # ---- JOIN (open? fallback to invite only) ----
    if sub == "join":
        if not args:
            return await send("❌ Use `,clan join <name>` (must be invited first).", ephemeral=True)
        cid = await get_invite(uid, gid)
        if not cid:
            return await send("❌ You need an invite first.", ephemeral=True)
        c = await get_clan_by_id(cid)
        if not c or c["name"].lower() != args.strip().lower():
            return await send("❌ Your invite is for a different clan.", ephemeral=True)
        await add_clan_member(uid, gid, cid)
        await remove_invite(uid, gid)
        return await send(f"✅ Joined **{c['name']}**!")

    # ---- LEAVE ----
    if sub == "leave":
        uc = await get_user_clan(uid, gid)
        if not uc:
            return await send("❌ You're not in a clan.", ephemeral=True)
        if uc["rank"] == "owner":
            return await send("❌ Owners must `,clan disband` or transfer ownership first.", ephemeral=True)
        await remove_clan_member(uid, gid)
        return await send("👋 You left your clan.")

    # ---- KICK ----
    if sub == "kick":
        if not args:
            return await send("❌ Use `,clan kick @user`", ephemeral=True)
        uc = await get_user_clan(uid, gid)
        if not uc or uc["rank"] not in ("owner", "officer"):
            return await send("❌ Only owners/officers can kick.", ephemeral=True)
        try:
            target_id = int(args.strip("<@!>"))
        except ValueError:
            return await send("❌ Mention a user.", ephemeral=True)
        tuc = await get_user_clan(target_id, gid)
        if not tuc or tuc["clan_id"] != uc["clan_id"]:
            return await send("❌ That user isn't in your clan.", ephemeral=True)
        if tuc["rank"] == "owner":
            return await send("❌ Can't kick the owner.", ephemeral=True)
        await remove_clan_member(target_id, gid)
        return await send(f"👢 Kicked <@{target_id}> from the clan.")

    # ---- PROMOTE ----
    if sub == "promote":
        if not args:
            return await send("❌ Use `,clan promote @user`", ephemeral=True)
        uc = await get_user_clan(uid, gid)
        if not uc or uc["rank"] != "owner":
            return await send("❌ Only the owner can promote.", ephemeral=True)
        try:
            target_id = int(args.strip("<@!>"))
        except ValueError:
            return await send("❌ Mention a user.", ephemeral=True)
        tuc = await get_user_clan(target_id, gid)
        if not tuc or tuc["clan_id"] != uc["clan_id"]:
            return await send("❌ That user isn't in your clan.", ephemeral=True)
        if tuc["rank"] == "member":
            await set_clan_rank(target_id, gid, "officer")
            return await send(f"⬆️ Promoted <@{target_id}> to **officer**.")
        elif tuc["rank"] == "officer":
            await set_clan_rank(target_id, gid, "owner")
            await set_clan_rank(uid, gid, "officer")
            await update_clan_owner(uc["clan_id"], target_id)
            return await send(f"👑 Transferred ownership to <@{target_id}>.")

    # ---- INFO ----
    if sub == "info":
        if args:
            c = await get_clan_by_name(gid, args.strip())
        else:
            uc = await get_user_clan(uid, gid)
            if not uc:
                return await send("❌ You're not in a clan.", ephemeral=True)
            c = await get_clan_by_id(uc["clan_id"])
        if not c:
            return await send("❌ Clan not found.", ephemeral=True)
        members = await get_clan_members(c["clan_id"])
        owner = guild.get_member(c["owner_id"])
        owner_name = owner.display_name if owner else f"User {c['owner_id']}"
        member_list = ""
        for m_uid, rank, contrib in members[:15]:
            mm = guild.get_member(m_uid)
            mn = mm.display_name if mm else f"User {m_uid}"
            icon = {"owner": "👑", "officer": "⭐", "member": "•"}.get(rank, "•")
            member_list += f"{icon} **{mn}** — {contrib:,} contributed\n"
        e = discord.Embed(title=f"🏰 {c['name']}", color=0xf1c40f)
        e.add_field(name="Owner", value=owner_name, inline=True)
        e.add_field(name="Vault", value=f"**{c['bank']:,}**", inline=True)
        e.add_field(name="Members", value=f"{len(members)}/{CLAN_MAX_MEMBERS}", inline=True)
        if c["description"]:
            e.add_field(name="Description", value=c["description"], inline=False)
        if member_list:
            e.add_field(name="Member List", value=member_list[:1024], inline=False)
        return await send(embed=e)

    # ---- LIST ----
    if sub == "list":
        clans = await list_clans(gid)
        if not clans:
            return await send("📭 No clans yet.")
        desc = ""
        for cid, name, bank, owner, count in clans[:15]:
            desc += f"🏰 **{name}** — {count} members, vault **{bank:,}**\n"
        e = discord.Embed(title="🏰 Clans", description=desc, color=0x9b59b6)
        return await send(embed=e)

    # ---- TOP ----
    if sub == "top":
        clans = await list_clans(gid)
        if not clans:
            return await send("📭 No clans yet.")
        medals = ["🥇", "🥈", "🥉"]
        desc = ""
        for i, (cid, name, bank, owner, count) in enumerate(clans[:10]):
            pre = medals[i] if i < 3 else f"`#{i+1}`"
            desc += f"{pre} **{name}** — {bank:,} in vault ({count} members)\n"
        e = discord.Embed(title="🏆 Clan Leaderboard", description=desc, color=0xf1c40f)
        return await send(embed=e)

    # ---- DEPOSIT ----
    if sub == "deposit":
        if not args:
            return await send("❌ Use `,clan deposit <amount>`", ephemeral=True)
        uc = await get_user_clan(uid, gid)
        if not uc:
            return await send("❌ You're not in a clan.", ephemeral=True)
        try:
            amt = int(args)
        except ValueError:
            return await send("❌ Invalid amount.", ephemeral=True)
        if amt <= 0:
            return await send("❌ Amount must be > 0.", ephemeral=True)
        u = await get_user(uid, gid)
        if u["wallet"] < amt:
            return await send(f"❌ You only have **{u['wallet']:,}** in wallet.", ephemeral=True)
        await set_balance(uid, gid, "wallet", -amt, add=True)
        await add_clan_bank(uc["clan_id"], amt)
        await add_contribution(uid, gid, amt)
        return await send(f"🏦 Deposited **{amt:,}** to the clan vault!")

    # ---- WITHDRAW ----
    if sub == "withdraw":
        if not args:
            return await send("❌ Use `,clan withdraw <amount>`", ephemeral=True)
        uc = await get_user_clan(uid, gid)
        if not uc or uc["rank"] not in ("owner", "officer"):
            return await send("❌ Only owners/officers can withdraw.", ephemeral=True)
        try:
            amt = int(args)
        except ValueError:
            return await send("❌ Invalid amount.", ephemeral=True)
        c = await get_clan_by_id(uc["clan_id"])
        if amt <= 0 or amt > c["bank"]:
            return await send("❌ Invalid amount or insufficient vault funds.", ephemeral=True)
        await add_clan_bank(uc["clan_id"], -amt)
        await set_balance(uid, gid, "wallet", amt, add=True)
        return await send(f"💵 Withdrew **{amt:,}** from the clan vault.")

    # ---- BANK ----
    if sub == "bank":
        uc = await get_user_clan(uid, gid)
        if not uc:
            return await send("❌ You're not in a clan.", ephemeral=True)
        c = await get_clan_by_id(uc["clan_id"])
        members = await get_clan_members(uc["clan_id"])
        top = sorted(members, key=lambda m: m[2], reverse=True)[:5]
        desc = ""
        for m_uid, rank, contrib in top:
            mm = guild.get_member(m_uid)
            mn = mm.display_name if mm else f"User {m_uid}"
            desc += f"**{mn}** — {contrib:,}\n"
        e = discord.Embed(title=f"🏦 {c['name']} Vault", color=0x2ecc71)
        e.add_field(name="Balance", value=f"**{c['bank']:,}**", inline=False)
        e.add_field(name="Top Contributors", value=desc or "None yet.", inline=False)
        return await send(embed=e)

    # ---- SETDESC ----
    if sub == "setdesc":
        uc = await get_user_clan(uid, gid)
        if not uc or uc["rank"] != "owner":
            return await send("❌ Only the owner can set the description.", ephemeral=True)
        if not args:
            return await send("❌ Provide a description.", ephemeral=True)
        await set_clan_description(uc["clan_id"], args[:200])
        return await send("📝 Clan description updated!")

    return await send(f"❌ Unknown subcommand: `{sub}`", ephemeral=True)

# ================= OWNER COMMANDS =================
@bot.command(name="givemoney")
@is_owner()
async def givemoney(ctx, member: discord.Member, amount: int):
    await set_balance(member.id, ctx.guild.id, "wallet", amount, add=True)
    await ctx.send(f"✅ Gave **{amount:,}** to {member.mention}.")

@bot.tree.command(name="givemoney", description="[Owner] Add coins to a user's wallet")
@owner_only()
async def givemoney_slash(interaction: discord.Interaction, member: discord.Member, amount: int):
    await set_balance(member.id, interaction.guild.id, "wallet", amount, add=True)
    await interaction.response.send_message(f"✅ Gave **{amount:,}** to {member.mention}.")

@bot.command(name="takemoney")
@is_owner()
async def takemoney(ctx, member: discord.Member, amount: int):
    await set_balance(member.id, ctx.guild.id, "wallet", -amount, add=True)
    await ctx.send(f"✅ Took **{amount:,}** from {member.mention}.")

@bot.tree.command(name="takemoney", description="[Owner] Remove coins from a user's wallet")
@owner_only()
async def takemoney_slash(interaction: discord.Interaction, member: discord.Member, amount: int):
    await set_balance(member.id, interaction.guild.id, "wallet", -amount, add=True)
    await interaction.response.send_message(f"✅ Took **{amount:,}** from {member.mention}.")

@bot.command(name="setmoney")
@is_owner()
async def setmoney(ctx, member: discord.Member, amount: int):
    await set_balance(member.id, ctx.guild.id, "wallet", amount)
    await ctx.send(f"✅ Set {member.mention}'s wallet to **{amount:,}**.")

@bot.tree.command(name="setmoney", description="[Owner] Set a user's wallet")
@owner_only()
async def setmoney_slash(interaction: discord.Interaction, member: discord.Member, amount: int):
    await set_balance(member.id, interaction.guild.id, "wallet", amount)
    await interaction.response.send_message(f"✅ Set {member.mention}'s wallet to **{amount:,}**.")

@bot.command(name="givebank")
@is_owner()
async def givebank(ctx, member: discord.Member, amount: int):
    await set_balance(member.id, ctx.guild.id, "bank", amount, add=True)
    await ctx.send(f"✅ Gave **{amount:,}** to {member.mention}'s bank.")

@bot.tree.command(name="givebank", description="[Owner] Add coins to a user's bank")
@owner_only()
async def givebank_slash(interaction: discord.Interaction, member: discord.Member, amount: int):
    await set_balance(member.id, interaction.guild.id, "bank", amount, add=True)
    await interaction.response.send_message(f"✅ Gave **{amount:,}** to {member.mention}'s bank.")

@bot.command(name="setbank")
@is_owner()
async def setbank(ctx, member: discord.Member, amount: int):
    await set_balance(member.id, ctx.guild.id, "bank", amount)
    await ctx.send(f"✅ Set {member.mention}'s bank to **{amount:,}**.")

@bot.tree.command(name="setbank", description="[Owner] Set a user's bank")
@owner_only()
async def setbank_slash(interaction: discord.Interaction, member: discord.Member, amount: int):
    await set_balance(member.id, interaction.guild.id, "bank", amount)
    await interaction.response.send_message(f"✅ Set {member.mention}'s bank to **{amount:,}**.")

@bot.command(name="resetuser")
@is_owner()
async def resetuser(ctx, member: discord.Member):
    await set_balance(member.id, ctx.guild.id, "wallet", 0)
    await set_balance(member.id, ctx.guild.id, "bank", 0)
    await ctx.send(f"✅ Reset {member.mention}'s wallet and bank to 0.")

@bot.tree.command(name="resetuser", description="[Owner] Reset a user's wallet + bank")
@owner_only()
async def resetuser_slash(interaction: discord.Interaction, member: discord.Member):
    await set_balance(member.id, interaction.guild.id, "wallet", 0)
    await set_balance(member.id, interaction.guild.id, "bank", 0)
    await interaction.response.send_message(f"✅ Reset {member.mention}'s wallet and bank to 0.")

@bot.command(name="resetall")
@is_owner()
async def resetall(ctx):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET wallet=0, bank=0 WHERE guild_id=?", (ctx.guild.id,))
        await db.commit()
    await ctx.send("⚠️ Reset **everyone's** wallet and bank to 0.")

@bot.tree.command(name="resetall", description="[Owner] Reset every user's balance")
@owner_only()
async def resetall_slash(interaction: discord.Interaction):
    async with aiosqlite.connect(DB_PATH) as db:
        await db.execute("UPDATE users SET wallet=0, bank=0 WHERE guild_id=?", (interaction.guild.id,))
        await db.commit()
    await interaction.response.send_message("⚠️ Reset **everyone's** wallet and bank to 0.")

@bot.command(name="forcedisband")
@is_owner()
async def forcedisband(ctx, *, name: str):
    c = await get_clan_by_name(ctx.guild.id, name)
    if not c:
        return await ctx.send("❌ Clan not found.")
    await delete_clan(c["clan_id"])
    await ctx.send(f"✅ Force-disbanded **{c['name']}**.")

@bot.tree.command(name="forcedisband", description="[Owner] Force-disband a clan")
@owner_only()
async def forcedisband_slash(interaction: discord.Interaction, name: str):
    c = await get_clan_by_name(interaction.guild.id, name)
    if not c:
        return await interaction.response.send_message("❌ Clan not found.", ephemeral=True)
    await delete_clan(c["clan_id"])
    await interaction.response.send_message(f"✅ Force-disbanded **{c['name']}**.")

# ================= HELP =================
@bot.command(name="help")
async def help_cmd(ctx):
    e = discord.Embed(title="🎲 Gamble Bot Help", color=0x5865f2)
    e.add_field(name="💵 Economy", value="`,balance` `,daily` `,leaderboard` `,give @user <amt>`", inline=False)
    e.add_field(name="🏦 Bank", value="`,dep <amt|all>` `,with <amt|all>`", inline=False)
    e.add_field(name="🎰 Gambling", value="`,gamble` `,coinflip` `,dice` `,slots` `,blackjack` `,roulette` `,wheel` `,crash` `,highlow`", inline=False)
    e.add_field(name="🥷 Rob", value="`,rob @user` — steal half their wallet", inline=False)
    e.add_field(name="🏰 Clan", value="`,clan create|invite|accept|leave|kick|info|list|top|deposit|withdraw|bank|promote|setdesc|disband`", inline=False)
    e.add_field(name="👑 Owner", value="`,givemoney` `,takemoney` `,setmoney` `,givebank` `,setbank` `,resetuser` `,resetall` `,forcedisband`", inline=False)
    await ctx.send(embed=e)

@bot.tree.command(name="help", description="View all commands")
async def help_slash(interaction: discord.Interaction):
    e = discord.Embed(title="🎲 Gamble Bot Help", color=0x5865f2)
    e.add_field(name="💵 Economy", value="`/balance` `/daily` `/leaderboard` `/give`", inline=False)
    e.add_field(name="🏦 Bank", value="`/deposit` `/withdraw`", inline=False)
    e.add_field(name="🎰 Gambling", value="`/gamble` `/coinflip` `/dice` `/slots` `/roulette` `/wheel` `/crash` `/highlow`", inline=False)
    e.add_field(name="🥷 Rob", value="`/rob`", inline=False)
    e.add_field(name="🏰 Clan", value="`/clan`", inline=False)
    await interaction.response.send_message(embed=e)

# ================= RUN =================
async def main():
    await init_db()
    async with bot:
        await bot.start(TOKEN)

if __name__ == "__main__":
    asyncio.run(main()) 
