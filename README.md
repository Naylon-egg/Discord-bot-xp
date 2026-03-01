# Discord-bot-xp
# Bot de xp para o discord movido a discord.py

import discord
from discord.ext import commands
import random
import json
import os

# =========================
# CONFIGURAÇÃO DO BOT
# =========================

TOKEN = ""  # Coloque seu token aqui

intents = discord.Intents.default()
intents.message_content = True

bot = commands.Bot(command_prefix="?", intents=intents)

# =========================
# SISTEMA DE DADOS
# =========================

DATA_FILE = "xp_data.json"

def load_data():
    if not os.path.exists(DATA_FILE):
        return {}
    with open(DATA_FILE, "r") as f:
        return json.load(f)

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f, indent=4)

data = load_data()

def get_guild_data(guild_id):
    guild_id = str(guild_id)
    if guild_id not in data:
        data[guild_id] = {
            "users": {},
            "xp_per_level": 100,  # XP base para o nível 1
            "xp_channel": None,
            "level_channel": None
        }
    return data[guild_id]

# =========================
# BARRA DE XP BONITA
# =========================

def create_xp_bar(current_xp, xp_needed):
    total_blocks = 20
    progress_ratio = current_xp / xp_needed
    filled_blocks = int(progress_ratio * total_blocks)
    empty_blocks = total_blocks - filled_blocks
    bar = "🟩" * filled_blocks + "⬜" * empty_blocks
    percent = round(progress_ratio * 100, 1)
    remaining = xp_needed - current_xp
    return bar, percent, remaining

# =========================
# SISTEMA DE XP POR CARACTERES
# =========================

@bot.event
async def on_message(message):
    if message.author.bot or not message.guild:
        return

    guild_data = get_guild_data(message.guild.id)

    if guild_data["xp_channel"] is None or message.channel.id != guild_data["xp_channel"]:
        await bot.process_commands(message)
        return

    if len(message.content) < 10:
        await bot.process_commands(message)
        return

    user_id = str(message.author.id)

    if user_id not in guild_data["users"]:
        guild_data["users"][user_id] = {"xp": 0, "level": 1}

    user_data = guild_data["users"][user_id]

    # XP ganho por caracteres
    xp_gain = len(message.content) // 5
    user_data["xp"] += xp_gain

    leveled_up = False

    # XP necessário aumenta naturalmente a cada nível
    while True:
        xp_needed = int(guild_data["xp_per_level"] * (1.2 ** (user_data["level"] - 1)))
        if user_data["xp"] < xp_needed:
            break
        user_data["xp"] -= xp_needed
        user_data["level"] += 1
        leveled_up = True

    if leveled_up:
        bar, percent, remaining = create_xp_bar(user_data["xp"], xp_needed)
        embed = discord.Embed(
            title="✨ LEVEL UP ✨",
            description=(
                f"{message.author.mention} alcançou o **Nível {user_data['level']}**!\n\n"
                f"📊 **Progresso:**\n{bar} {percent}%\n"
                f"🔹 XP: {user_data['xp']} / {xp_needed}\n"
                f"🔸 Faltam: {remaining} XP"
            ),
            color=0x00ff99
        )
        if guild_data["level_channel"]:
            channel = bot.get_channel(guild_data["level_channel"])
            if channel:
                await channel.send(embed=embed)
        else:
            await message.channel.send(embed=embed)

    save_data(data)
    await bot.process_commands(message)

# =========================
# COMANDOS ADMIN
# =========================

@bot.command()
@commands.has_permissions(administrator=True)
async def setxp(ctx, quantidade: int):
    guild_data = get_guild_data(ctx.guild.id)
    guild_data["xp_per_level"] = quantidade
    save_data(data)
    await ctx.send(f"XP base necessário para upar definido como {quantidade}.")

@bot.command()
@commands.has_permissions(administrator=True)
async def setxpchannel(ctx, canal: discord.TextChannel):
    guild_data = get_guild_data(ctx.guild.id)
    guild_data["xp_channel"] = canal.id
    save_data(data)
    await ctx.send(f"Canal de XP definido para {canal.mention}.")

@bot.command()
@commands.has_permissions(administrator=True)
async def setlevelchannel(ctx, canal: discord.TextChannel):
    guild_data = get_guild_data(ctx.guild.id)
    guild_data["level_channel"] = canal.id
    save_data(data)
    await ctx.send(f"Canal de level up definido para {canal.mention}.")

@bot.command()
@commands.has_permissions(administrator=True)
async def addxp(ctx, membro: discord.Member, quantidade: int):
    guild_data = get_guild_data(ctx.guild.id)
    user_id = str(membro.id)
    if user_id not in guild_data["users"]:
        guild_data["users"][user_id] = {"xp": 0, "level": 1}
    guild_data["users"][user_id]["xp"] += quantidade
    save_data(data)
    await ctx.send(f"{quantidade} XP adicionados para {membro.mention}.")

@bot.command()
@commands.has_permissions(administrator=True)
async def removexp(ctx, membro: discord.Member, quantidade: int):
    guild_data = get_guild_data(ctx.guild.id)
    user_id = str(membro.id)
    if user_id not in guild_data["users"]:
        await ctx.send("Usuário não possui XP.")
        return
    guild_data["users"][user_id]["xp"] -= quantidade
    if guild_data["users"][user_id]["xp"] < 0:
        guild_data["users"][user_id]["xp"] = 0
    save_data(data)
    await ctx.send(f"{quantidade} XP removidos de {membro.mention}.")

@bot.command()
@commands.has_permissions(administrator=True)
async def setlevel(ctx, membro: discord.Member, nivel: int):
    guild_data = get_guild_data(ctx.guild.id)
    user_id = str(membro.id)
    if user_id not in guild_data["users"]:
        guild_data["users"][user_id] = {"xp": 0, "level": 1}
    guild_data["users"][user_id]["level"] = max(1, nivel)
    save_data(data)
    await ctx.send(f"Nível de {membro.mention} definido para {nivel}.")

# =========================
# COMANDO RANK
# =========================

@bot.command()
async def rank(ctx):
    guild_data = get_guild_data(ctx.guild.id)
    user_id = str(ctx.author.id)
    if user_id not in guild_data["users"]:
        await ctx.send("Você ainda não tem XP.")
        return
    user_data = guild_data["users"][user_id]
    xp_needed = int(guild_data["xp_per_level"] * (1.2 ** (user_data["level"] - 1)))
    bar, percent, remaining = create_xp_bar(user_data["xp"], xp_needed)
    embed = discord.Embed(
        title=f"🏆 Rank de {ctx.author.name}",
        description=(
            f"🎖️ **Nível:** {user_data['level']}\n\n"
            f"📊 **Progresso:**\n{bar} {percent}%\n"
            f"🔹 XP Atual: {user_data['xp']}\n"
            f"🔸 XP Necessário: {xp_needed}\n"
            f"✨ Faltam: {remaining} XP"
        ),
        color=0x3498db
    )
    await ctx.send(embed=embed)

# =========================
# COMANDOS DE AJUDA
# =========================

@bot.command()
async def ajuda(ctx):
    embed = discord.Embed(title="📜 Lista de Comandos", color=0x9b59b6)
    embed.add_field(
        name="👥 Públicos",
        value=(
            "`?rank`\n"
            "`?d20, ?d40, ?d60, ?d100`\n"
            "`?ajuda`"
        ),
        inline=False
    )
    await ctx.send(embed=embed)

@bot.command()
@commands.has_permissions(administrator=True)
async def akuma(ctx): # pode ser modificado
    embed = discord.Embed(title="🛠️ Comandos de Administrador", color=0xe74c3c)
    embed.add_field(
        name="⚙️ Administração de XP e Nível",
        value=(
            "`?setxp <quantidade>` - Define XP base para upar\n"
            "`?setxpchannel #canal` - Define canal que conta XP\n"
            "`?setlevelchannel #canal` - Define canal de level up\n"
            "`?addxp @usuario <quantidade>` - Adiciona XP\n"
            "`?removexp @usuario <quantidade>` - Remove XP\n"
            "`?setlevel @usuario <nivel>` - Define nível de um usuário"
        ),
        inline=False
    )
    await ctx.send(embed=embed)

# =========================
# COMANDOS DADOS FIXOS
# =========================

@bot.command()
async def d20(ctx):
    resultado = random.randint(1, 20)
    embed = discord.Embed(
        title=f"🎲 {ctx.author.name} rolou 1d20",
        description=f"Resultado: **{resultado}**",
        color=0x1abc9c
    )
    await ctx.send(embed=embed)

@bot.command()
async def d40(ctx):
    resultado = random.randint(1, 40)
    embed = discord.Embed(
        title=f"🎲 {ctx.author.name} rolou 1d40",
        description=f"Resultado: **{resultado}**",
        color=0x1abc9c
    )
    await ctx.send(embed=embed)

@bot.command()
async def d60(ctx):
    resultado = random.randint(1, 60)
    embed = discord.Embed(
        title=f"🎲 {ctx.author.name} rolou 1d60",
        description=f"Resultado: **{resultado}**",
        color=0x1abc9c
    )
    await ctx.send(embed=embed)

@bot.command()
async def d100(ctx):
    resultado = random.randint(1, 100)
    embed = discord.Embed(
        title=f"🎲 {ctx.author.name} rolou 1d100",
        description=f"Resultado: **{resultado}**",
        color=0x1abc9c
    )
    await ctx.send(embed=embed)

# =========================
# BOT ONLINE
# =========================

@bot.event
async def on_ready():
    print(f"Conectado como {bot.user}")

bot.run(TOKEN)
