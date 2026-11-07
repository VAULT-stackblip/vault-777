import os
import json
import sqlite3
import random
import asyncio
import secrets
import traceback
from datetime import datetime, timedelta

import discord
from discord import app_commands
from discord.ext import commands, tasks
from dotenv import load_dotenv

# นำเข้า yt-dlp สำหรับสตรีมเสียงจาก YouTube
try:
    import yt_dlp
except ImportError:
    yt_dlp = None

# ==============================
# CONFIG & ENVIRONMENT
# ==============================
load_dotenv()

TOKEN = os.getenv("DISCORD_TOKEN")
OWNER_ID = int(os.getenv("VAULT_OWNER_ID", "0"))

OWNER_KEY = os.getenv("VAULT_OWNER_KEY", "").strip()
ADMIN_KEY = os.getenv("VAULT_ADMIN_KEY", "").strip()
USER_KEY = os.getenv("VAULT_USER_KEY", "").strip()

ANNOUNCEMENT_CHANNEL_ID = int(os.getenv("ANNOUNCEMENT_CHANNEL_ID", "0"))

VOICE_CONFIG_FILE = "voice_config.json"
DB_FILE = "vault_bot.db"

intents = discord.Intents.default()
intents.members = True
intents.message_content = True
intents.voice_states = True

bot = commands.Bot(command_prefix="!", intents=intents)

# ==============================
# DATABASE SETUP (SQLite)
# ==============================
def init_db():
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS user_levels (
            guild_id INTEGER,
            user_id INTEGER,
            xp INTEGER DEFAULT 0,
            level INTEGER DEFAULT 1,
            voice_minutes INTEGER DEFAULT 0,
            PRIMARY KEY (guild_id, user_id)
        )
    """)
    conn.commit()
    conn.close()

init_db()

# ==============================
# HELPER FUNCTIONS & KEY CHECK
# ==============================
def load_voice_config():
    try:
        with open(VOICE_CONFIG_FILE, "r", encoding="utf-8") as file:
            return json.load(file)
    except (FileNotFoundError, json.JSONDecodeError):
        return {}

def save_voice_config(data):
    with open(VOICE_CONFIG_FILE, "w", encoding="utf-8") as file:
        json.dump(data, file, indent=4)

def verify_key(user_id: int, supplied_key: str, required_level: str) -> bool:
    if not supplied_key:
        return False
    
    clean_key = supplied_key.strip()

    if required_level == "owner":
        if not OWNER_KEY:
            return False
        key_matches = secrets.compare_digest(clean_key, OWNER_KEY)
        if OWNER_ID != 0:
            return key_matches and (user_id == OWNER_ID)
        return key_matches
        
    elif required_level == "admin":
        if not ADMIN_KEY:
            return False
        return secrets.compare_digest(clean_key, ADMIN_KEY)
        
    elif required_level == "user":
        if not USER_KEY:
            return False
        return secrets.compare_digest(clean_key, USER_KEY)
        
    return False

class VaultKeyModal(discord.ui.Modal):
    def __init__(self, required_level: str, action_func):
        super().__init__(title="Vault 777 • Key Verification", timeout=180)
        self.required_level = required_level
        self.action_func = action_func
        
        self.key_input = discord.ui.TextInput(
            label=f"กรอกรหัส {required_level.upper()} KEY",
            placeholder="ใส่ Vault Key เพื่อดำเนินการ...",
            required=True,
            min_length=1,
            max_length=128
        )
        self.add_item(self.key_input)

    async def on_submit(self, interaction: discord.Interaction):
        valid = verify_key(interaction.user.id, self.key_input.value, self.required_level)

        if not valid:
            await interaction.response.send_message(
                "⛔ Vault Key ไม่ถูกต้อง หรือไม่ได้ตั้งค่าคีย์ในระบบ",
                ephemeral=True
            )
            return

        if not interaction.response.is_done():
            await interaction.response.defer(ephemeral=True)

        try:
            await self.action_func(interaction)
        except discord.Forbidden:
            await interaction.followup.send("⛔ บอทไม่มีสิทธิ์ในการดำเนินการนี้", ephemeral=True)
        except Exception as error:
            print(f"[ACTION ERROR] {error}")
            traceback.print_exc()
            await interaction.followup.send("เกิดข้อผิดพลาดระหว่างทำรายการ", ephemeral=True)

# ==============================
# 1. ROLE SYSTEM (PERSISTENT BUTTONS)
# ==============================
class RoleButton(discord.ui.Button):
    def __init__(self, role_id: int, role_name: str = "บทบาท"):
        super().__init__(
            label=f"รับ/คืนยศ: {role_name}",
            style=discord.ButtonStyle.primary,
            custom_id=f"persistent_role_{role_id}"
        )
        self.role_id = role_id

    async def callback(self, interaction: discord.Interaction):
        async def action(i: discord.Interaction):
            guild = i.guild
            role = guild.get_role(self.role_id)
            
            if not role:
                await i.followup.send("⛔ ไม่พบบทบาทนี้ในเซิร์ฟเวอร์", ephemeral=True)
                return

            bot_member = guild.me
            if bot_member.top_role <= role:
                await i.followup.send(
                    f"⛔ บอทไม่สามารถจัดการยศ **{role.name}** ได้ เนื่องจากยศบอทอยู่ต่ำกว่าหรือเท่ากับยศนี้",
                    ephemeral=True
                )
                return

            if not bot_member.guild_permissions.manage_roles:
                await i.followup.send("⛔ บอทไม่มีสิทธิ์ `Manage Roles`", ephemeral=True)
                return

            if role in i.user.roles:
                await i.user.remove_roles(role)
                await i.followup.send(f"➖ ถอดยศ **{role.name}** เรียบร้อยแล้ว", ephemeral=True)
            else:
                await i.user.add_roles(role)
                await i.followup.send(f"➕ เพิ่มยศ **{role.name}** เรียบร้อยแล้ว", ephemeral=True)

        await interaction.response.send_modal(VaultKeyModal("user", action))

class DynamicRoleView(discord.ui.View):
    def __init__(self, roles=None):
        super().__init__(timeout=None)
        if roles:
            for r in roles:
                self.add_item(RoleButton(r.id, r.name))

@bot.tree.command(name="create_role_panel", description="สร้างแผงเลือกรับยศ (ใส่ยศได้สูงสุด 5 ยศ)")
@app_commands.checks.has_permissions(administrator=True)
async def create_role_panel(
    interaction: discord.Interaction,
    role1: discord.Role,
    role2: discord.Role = None,
    role3: discord.Role = None,
    role4: discord.Role = None,
    role5: discord.Role = None
):
    async def action(i: discord.Interaction):
        roles = [r for r in [role1, role2, role3, role4, role5] if r is not None]
        view = DynamicRoleView(roles=roles)
        
        embed = discord.Embed(
            title="🛡️ เลือกรับบทบาท (Role Selection)",
            description="กดปุ่มด้านล่างเพื่อรับหรือถอดยศที่คุณต้องการ (ต้องกรอก USER KEY)",
            color=discord.Color.blue()
        )
        await i.channel.send(embed=embed, view=view)
        await i.followup.send("✅ สร้างแผงเลือกรับยศเรียบร้อยแล้ว!", ephemeral=True)

    await interaction.response.send_modal(VaultKeyModal("admin", action))

async def setup_hook():
    @bot.event
    async def on_interaction(interaction: discord.Interaction):
        if interaction.type == discord.InteractionType.component:
            custom_id = interaction.data.get("custom_id", "")
            if custom_id.startswith("persistent_role_"):
                try:
                    role_id = int(custom_id.replace("persistent_role_", ""))
                    button = RoleButton(role_id=role_id, role_name="บทบาท")
                    await button.callback(interaction)
                except Exception as e:
                    print(f"[PERSISTENT BUTTON ERROR] {e}")

bot.setup_hook = setup_hook

# ==============================
# 2. SCHEDULER & TIME COMMAND SYSTEM
# ==============================
last_announced_minute = ""

@tasks.loop(seconds=10)
async def uptime_scheduler():
    global last_announced_minute
    now = datetime.now()
    current_time = now.strftime("%H:%M")

    if current_time == last_announced_minute:
        return

    is_sunday = (now.weekday() == 6)
    should_announce = False
    message_text = ""

    if current_time == "21:55" and not is_sunday:
        should_announce = True
        message_text = "📢 **[แจ้งเตือนระบบ]** ระบบกำลังจะปิดบริการในอีก 5 นาที (เวลา 22:00 น.)"
    elif current_time == "19:55" and is_sunday:
        should_announce = True
        message_text = "📢 **[แจ้งเตือนตารางพิเศษวันอาทิตย์]** ระบบกำลังจะปิดบริการในอีก 5 นาที (เวลา 20:00 น.)"
    elif current_time == "08:00":
        should_announce = True
        message_text = "☀️ **[แจ้งเตือนระบบ]** ระบบเปิดให้บริการตามปกติเรียบร้อยแล้ว!"

    if should_announce:
        last_announced_minute = current_time
        if ANNOUNCEMENT_CHANNEL_ID != 0:
            channel = bot.get_channel(ANNOUNCEMENT_CHANNEL_ID)
            if channel:
                try:
                    await channel.send(message_text)
                except Exception as e:
                    print(f"[SCHEDULER ERROR] ส่งข้อความไม่สำเร็จ: {e}")

def get_schedule_embed():
    now = datetime.now()
    weekday = now.weekday()
    current_time_str = now.strftime("%H:%M")

    current_slot = "🔴 ปิดให้บริการ (ไม่อยู่ในช่วงเวลาทำการ)"
    next_slot = "ยังไม่มีข้อมูลตารางถัดไป"

    if weekday < 5:
        if "17:10" <= current_time_str <= "18:30":
            current_slot = "🟢 วันจันทร์ – ศุกร์ (17:10 – 18:30)"
            next_slot = "วันเสาร์ (10:00 – 18:00)"
        elif current_time_str < "17:10":
            next_slot = "วันจันทร์ – ศุกร์ (17:10 – 18:30)"
        else:
            if weekday == 4:
                next_slot = "วันเสาร์ (10:00 – 18:00)"
            else:
                next_slot = "วันถัดไป (17:10 – 18:30)"
    elif weekday == 5:
        if "10:00" <= current_time_str <= "18:00":
            current_slot = "🟢 วันเสาร์ (10:00 – 18:00)"
            next_slot = "วันอาทิตย์ (08:00 – 12:00)"
        elif current_time_str < "10:00":
            next_slot = "วันเสาร์ (10:00 – 18:00)"
        else:
            next_slot = "วันอาทิตย์ (08:00 – 12:00)"
    elif weekday == 6:
        if "08:00" <= current_time_str <= "12:00":
            current_slot = "🟢 วันอาทิตย์ (08:00 – 12:00)"
            next_slot = "วันอาทิตย์พิเศษ (13:00 – 18:00)"
        elif "13:00" <= current_time_str <= "18:00":
            current_slot = "🟢 วันอาทิตย์พิเศษ (13:00 – 18:00)"
            next_slot = "วันจันทร์ (17:10 – 18:30)"
        elif current_time_str < "08:00":
            next_slot = "วันอาทิตย์ (08:00 – 12:00)"
        elif "12:00" <= current_time_str < "13:00":
            next_slot = "วันอาทิตย์พิเศษ (13:00 – 18:00)"
        else:
            next_slot = "วันจันทร์ (17:10 – 18:30)"

    embed = discord.Embed(
        title="Vault 777 』 — Operating Schedule",
        description="ตารางเวลาเปิด–ปิดให้บริการของระบบประจำสัปดาห์",
        color=discord.Color.gold()
    )
    embed.add_field(
        name="📅 รายละเอียดตารางเวลาทั้งหมด",
        value=(
            "**วันจันทร์ – ศุกร์**\n17:10 – 18:30\n\n"
            "**วันเสาร์**\n10:00 – 18:00\n\n"
            "**วันอาทิตย์**\n08:00 – 12:00\n\n"
            "**วันอาทิตย์พิเศษ**\n13:00 – 18:00"
        ),
        inline=False
    )
    embed.add_field(name="📍 <(ตารางปัจจุบัน)>", value=f"**{current_slot}**", inline=False)
    embed.add_field(name="⏳ [ตารางที่กำลังจะถึง]", value=f"**{next_slot}**", inline=False)
    embed.set_footer(text=f"เวลาปัจจุบันบนระบบ: {current_time_str} น.")
    return embed

@bot.tree.command(name="time", description="ดูตารางเวลาทำการของ Vault 777 (ไม่ต้องใช้คีย์)")
async def time_command(interaction: discord.Interaction):
    embed = get_schedule_embed()
    await interaction.response.send_message(embed=embed, ephemeral=True)

# ==============================
# 3. VOICE XP & TIME TRACKER SYSTEM
# ==============================
@tasks.loop(minutes=1)
async def voice_xp_tracker():
    for guild in bot.guilds:
        for vc in guild.voice_channels:
            if not vc.members or (guild.afk_channel and vc.id == guild.afk_channel.id):
                continue

            for member in vc.members:
                if member.bot:
                    continue

                guild_id = guild.id
                user_id = member.id

                conn = sqlite3.connect(DB_FILE)
                cursor = conn.cursor()
                cursor.execute("""
                    SELECT xp, level, voice_minutes FROM user_levels 
                    WHERE guild_id = ? AND user_id = ?
                """, (guild_id, user_id))
                row = cursor.fetchone()

                if row:
                    xp, lvl, v_mins = row
                    v_mins += 1  
                    
                    new_xp = xp
                    if v_mins >= 120:
                        v_mins = 0  
                        new_xp += 1 

                    needed_xp = lvl * 100
                    if new_xp >= needed_xp:
                        lvl += 1
                        new_xp -= needed_xp

                    cursor.execute("""
                        UPDATE user_levels 
                        SET xp = ?, level = ?, voice_minutes = ? 
                        WHERE guild_id = ? AND user_id = ?
                    """, (new_xp, lvl, v_mins, guild_id, user_id))
                else:
                    cursor.execute("""
                        INSERT INTO user_levels (guild_id, user_id, xp, level, voice_minutes) 
                        VALUES (?, ?, 0, 1, 1)
                    """, (guild_id, user_id))

                conn.commit()
                conn.close()

# ==============================
# 4. MUSIC & QUEUE SYSTEM
# ==============================
YTDL_OPTIONS = {
    'format': 'bestaudio/best',
    'restrictfilenames': True,
    'noplaylist': True,
    'nocheckcertificate': True,
    'ignoreerrors': False,
    'logtostderr': False,
    'quiet': True,
    'no_warnings': True,
    'default_search': 'auto',
    'source_address': '0.0.0.0',
}

FFMPEG_OPTIONS = {
    'before_options': '-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5',
    'options': '-vn',
}

ytdl = yt_dlp.YoutubeDL(YTDL_OPTIONS) if yt_dlp else None

class YTDLSource(discord.PCMVolumeTransformer):
    def __init__(self, source, *, data, volume=0.5):
        super().__init__(source, volume)
        self.data = data
        self.title = data.get('title')
        self.url = data.get('url')

    @classmethod
    async def create_source(cls, search: str, *, loop=None):
        loop = loop or asyncio.get_event_loop()
        data = await loop.run_in_executor(None, lambda: ytdl.extract_info(search, download=False))
        
        if 'entries' in data:
            data = data['entries'][0]

        filename = data['url']
        return cls(discord.FFmpegPCMAudio(filename, **FFMPEG_OPTIONS), data=data)

music_queues = {}

def play_next(guild, bot_instance):
    guild_id = guild.id
    if guild_id in music_queues and len(music_queues[guild_id]) > 0:
        next_source = music_queues[guild_id].pop(0)
        voice_client = guild.voice_client
        if voice_client and voice_client.is_connected():
            voice_client.play(next_source, after=lambda e: play_next(guild, bot_instance))

@bot.tree.command(name="play", description="เล่นเพลงจาก YouTube หรือเพิ่มลงคิว (ต้องใช้ USER KEY)")
async def play(interaction: discord.Interaction, search: str):
    async def action(i: discord.Interaction):
        if not i.user.voice:
            await i.followup.send("⛔ คุณต้องอยู่ในห้องเสียงก่อนใช้คำสั่งนี้", ephemeral=True)
            return

        if ytdl is None:
            await i.followup.send("⛔ ยังไม่ได้ติดตั้ง `yt-dlp` บนระบบบอท", ephemeral=True)
            return

        channel = i.user.voice.channel
        guild_id = i.guild_id
        vc = i.guild.voice_client

        if not vc or not vc.is_connected():
            voice_client = await channel.connect()
        else:
            voice_client = vc

        try:
            player = await YTDLSource.create_source(search, loop=bot.loop)
        except Exception as e:
            await i.followup.send(f"⛔ เกิดข้อผิดพลาดในการโหลดเพลง: {e}", ephemeral=True)
            return

        if guild_id not in music_queues:
            music_queues[guild_id] = []

        if voice_client.is_playing() or voice_client.is_paused():
            music_queues[guild_id].append(player)
            await i.followup.send(f"📥 เพิ่มเพลงลงในคิวแล้ว: **{player.title}** (ลำดับที่ {len(music_queues[guild_id])})")
        else:
            voice_client.play(player, after=lambda e: play_next(i.guild, bot))
            await i.followup.send(f"🎶 กำลังเล่นเพลง: **{player.title}**")

    await interaction.response.send_modal(VaultKeyModal("user", action))

@bot.tree.command(name="skip", description="ข้ามเพลงที่กำลังเล่นอยู่ไปยังเพลงถัดไปในคิว (ต้องใช้ USER KEY)")
async def skip(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        voice_client = i.guild.voice_client
        if voice_client and voice_client.is_playing():
            voice_client.stop()
            await i.followup.send("⏭️ ข้ามเพลงเรียบร้อยแล้ว!")
        else:
            await i.followup.send("⛔ ไม่มีเพลงกำลังเล่นอยู่", ephemeral=True)
    await interaction.response.send_modal(VaultKeyModal("user", action))

@bot.tree.command(name="stop", description="หยุดเล่นเพลงและเคลียร์คิว แต่บอทยังอยู่ในห้องเสียง (ต้องใช้ USER KEY)")
async def stop(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        guild_id = i.guild_id
        voice_client = i.guild.voice_client

        if not voice_client or not voice_client.is_connected():
            await i.followup.send("⛔ บอทไม่ได้อยู่ในห้องเสียงครับ", ephemeral=True)
            return

        if guild_id in music_queues:
            music_queues[guild_id].clear()

        if voice_client.is_playing() or voice_client.is_paused():
            voice_client.stop()

        await i.followup.send("⏹️ หยุดเล่นเพลงและล้างคิวทั้งหมดแล้ว (บอทยังคงสแตนด์บายอยู่ในห้องเสียง)")
    await interaction.response.send_modal(VaultKeyModal("user", action))

@bot.tree.command(name="queue", description="ดูรายการเพลงที่รออยู่ในคิวขณะนี้ (ไม่ต้องใช้คีย์)")
async def queue_command(interaction: discord.Interaction):
    guild_id = interaction.guild_id
    if guild_id not in music_queues or len(music_queues[guild_id]) == 0:
        await interaction.response.send_message("📭 ไม่มีเพลงรออยู่ในคิวขณะนี้ครับ", ephemeral=True)
        return

    queue_list = music_queues[guild_id]
    description = ""
    for i, song in enumerate(queue_list[:10], start=1):
        description += f"`{i}.` {song.title}\n"

    embed = discord.Embed(
        title="🎶 รายการคิวเพลงทั้งหมดในเซิร์ฟเวอร์",
        description=description,
        color=discord.Color.purple()
    )
    await interaction.response.send_message(embed=embed, ephemeral=True)

@bot.tree.command(name="stopmusic", description="หยุดเล่นเพลงและให้บอทออกจากห้องเสียง (ต้องใช้ USER KEY)")
async def stopmusic(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        vc = i.guild.voice_client
        if vc and vc.is_connected():
            if vc.is_playing():
                vc.stop()
            await vc.disconnect()
            await i.followup.send("⏹️ หยุดเล่นเพลงและออกจากห้องเสียงเรียบร้อยแล้ว")
        else:
            await i.followup.send("⛔ บอทไม่ได้อยู่ในห้องเสียง", ephemeral=True)

    await interaction.response.send_modal(VaultKeyModal("user", action))

# ==============================
# VOICE CONFIG SYSTEM
# ==============================
@bot.tree.command(name="setvoice", description="ตั้งค่าห้องเสียงประจำบอท")
@app_commands.checks.has_permissions(administrator=True)
async def setvoice(interaction: discord.Interaction, channel: discord.VoiceChannel):
    async def action(i: discord.Interaction):
        data = load_voice_config()
        data[str(i.guild.id)] = channel.id
        save_voice_config(data)
        await i.followup.send(f"🔊 ตั้งค่าห้องเสียงประจำบอทเป็น {channel.mention} เรียบร้อยแล้ว")
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="voiceon", description="สั่งให้บอทเข้าห้องเสียงประจำ")
@app_commands.checks.has_permissions(administrator=True)
async def voiceon(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        data = load_voice_config()
        cid = data.get(str(i.guild.id))
        if not cid:
            await i.followup.send("⛔ ยังไม่ได้ตั้งค่าห้องเสียงประจำ กรุณาใช้ `/setvoice` ก่อน", ephemeral=True)
            return

        channel = i.guild.get_channel(cid)
        if isinstance(channel, discord.VoiceChannel):
            await channel.connect()
            await i.followup.send(f"🔊 เข้าห้องเสียง {channel.mention} เรียบร้อยแล้ว")
        else:
            await i.followup.send("⛔ ไม่พบห้องเสียงที่ตั้งค่าไว้", ephemeral=True)
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="voiceoff", description="สั่งให้บอทออกจากห้องเสียง")
@app_commands.checks.has_permissions(administrator=True)
async def voiceoff(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        vc = i.guild.voice_client
        if vc and vc.is_connected():
            await vc.disconnect()
            await i.followup.send("🔇 ออกจากห้องเสียงเรียบร้อยแล้ว")
        else:
            await i.followup.send("⛔ บอทไม่ได้อยู่ในห้องเสียง", ephemeral=True)
    await interaction.response.send_modal(VaultKeyModal("admin", action))

# ==============================
# BOT EVENTS & ANTI-SPAM & LEVEL SYSTEM
# ==============================
user_message_logs = {}
xp_cooldowns = {}

@bot.event
async def on_ready():
    print(f"Vault 777 System Ready: {bot.user}")
    try:
        # ซิงก์คำสั่งลงทุก Guild โดยตรงทันที ไม่เคลียร์ทิ้งเพื่อป้องกันปัญหา 0 คำสั่ง
        for guild in bot.guilds:
            bot.tree.copy_global_to(guild=guild)
            synced = await bot.tree.sync(guild=guild)
            print(f"Synced {len(synced)} slash commands to Guild: {guild.name}")
            
    except Exception as e:
        print(f"Error syncing commands: {e}")
        
    if not uptime_scheduler.is_running():
        uptime_scheduler.start()
    if not voice_xp_tracker.is_running():
        voice_xp_tracker.start()

@bot.event
async def on_message(message: discord.Message):
    if message.author.bot or not message.guild:
        return

    guild_id = message.guild.id
    user_id = message.author.id
    now = datetime.utcnow()
    key = (guild_id, user_id)

    # ANTI-SPAM SYSTEM
    timestamps = user_message_logs.get(key, [])
    timestamps = [t for t in timestamps if (now - t).total_seconds() < 5]
    timestamps.append(now)
    user_message_logs[key] = timestamps

    if len(timestamps) >= 5:
        try:
            await message.delete()
            await message.author.timeout(timedelta(minutes=10), reason="Anti-Spam Auto Timeout")
            await message.channel.send(
                f"🚨 {message.author.mention} ถูกจำกัดเวลา 10 นาที เนื่องจากส่งข้อความถี่เกินไป",
                delete_after=10
            )
            return
        except Exception as e:
            print(f"Anti-spam action error: {e}")

    # CHAT XP SYSTEM (60s Cooldown)
    last_xp = xp_cooldowns.get(key)
    if not last_xp or (now - last_xp).total_seconds() >= 60:
        xp_cooldowns[key] = now
        xp_gained = random.randint(15, 25)

        conn = sqlite3.connect(DB_FILE)
        cursor = conn.cursor()
        cursor.execute("SELECT xp, level FROM user_levels WHERE guild_id = ? AND user_id = ?", (guild_id, user_id))
        row = cursor.fetchone()

        if row:
            current_xp, current_lvl = row
            new_xp = current_xp + xp_gained
            needed_xp = current_lvl * 100
            
            if new_xp >= needed_xp:
                new_lvl = current_lvl + 1
                new_xp -= needed_xp
                cursor.execute("UPDATE user_levels SET xp = ?, level = ? WHERE guild_id = ? AND user_id = ?",
                               (new_xp, new_lvl, guild_id, user_id))
                await message.channel.send(
                    f"🎉 ยินดีด้วย {message.author.mention}! คุณเลเวลอัปเป็น **Level {new_lvl}** แล้ว!",
                    delete_after=15
                )
            else:
                cursor.execute("UPDATE user_levels SET xp = ? WHERE guild_id = ? AND user_id = ?",
                               (new_xp, guild_id, user_id))
        else:
            cursor.execute("INSERT INTO user_levels (guild_id, user_id, xp, level, voice_minutes) VALUES (?, ?, ?, 1, 0)",
                           (guild_id, user_id, xp_gained))

        conn.commit()
        conn.close()

    await bot.process_commands(message)

# ==============================
# OWNER COMMAND (SET LEVEL & XP)
# ==============================
@bot.tree.command(name="setlevel", description="ตั้งค่าเลเวลและ XP ให้สมาชิก (ต้องใช้ OWNER KEY)")
async def setlevel(interaction: discord.Interaction, member: discord.Member, level: int, xp: int = 0):
    async def action(i: discord.Interaction):
        conn = sqlite3.connect(DB_FILE)
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO user_levels (guild_id, user_id, xp, level, voice_minutes)
            VALUES (?, ?, ?, ?, 0)
            ON CONFLICT(guild_id, user_id) 
            DO UPDATE SET level = ?, xp = ?
        """, (i.guild.id, member.id, xp, level, level, xp))
        conn.commit()
        conn.close()

        await i.followup.send(f"👑 ตั้งค่าให้ {member.mention} เป็น **Level {level}** (XP: {xp}) เรียบร้อยแล้ว!", ephemeral=True)

    await interaction.response.send_modal(VaultKeyModal("owner", action))

# ==============================
# USER COMMANDS (NO KEY REQUIRED)
# ==============================
@bot.tree.command(name="rank", description="ตรวจสอบเลเวลและ XP ของคุณ (ไม่ต้องใช้คีย์)")
async def rank(interaction: discord.Interaction, member: discord.Member = None):
    target = member or interaction.user
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("SELECT xp, level FROM user_levels WHERE guild_id = ? AND user_id = ?",
                   (interaction.guild.id, target.id))
    row = cursor.fetchone()
    conn.close()

    if row:
        xp, lvl = row
        needed = lvl * 100
        await interaction.response.send_message(f"📊 **{target.display_name}** | Level: `{lvl}` | XP: `{xp}/{needed}`", ephemeral=True)
    else:
        await interaction.response.send_message(f"📊 **{target.display_name}** ยังไม่มีข้อมูลเลเวล", ephemeral=True)

@bot.tree.command(name="voicetime", description="ดูเวลาที่คุณอยู่ในห้องเสียง (ไม่ต้องใช้คีย์)")
async def voicetime(interaction: discord.Interaction, member: discord.Member = None):
    target = member or interaction.user
    conn = sqlite3.connect(DB_FILE)
    cursor = conn.cursor()
    cursor.execute("SELECT voice_minutes FROM user_levels WHERE guild_id = ? AND user_id = ?",
                   (interaction.guild.id, target.id))
    row = cursor.fetchone()
    conn.close()

    total_minutes = row[0] if row else 0
    hours = total_minutes // 60
    minutes = total_minutes % 60

    await interaction.response.send_message(
        f"🎙️ **สถิติห้องเสียงของ {target.display_name}**\n🕒 อยู่ในห้องเสียงสะสมรวม: **{hours} ชั่วโมง {minutes} นาที** (สะสมครบทุกๆ 2 ชั่วโมงจะได้ 1 XP)",
        ephemeral=True
    )

# ==============================
# MODERATION & ADMIN COMMANDS (REQUIRE KEY)
# ==============================
@bot.tree.command(name="ping", description="ตรวจสอบสถานะบอท (ใช้ USER KEY)")
async def ping(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        latency = round(bot.latency * 1000)
        await i.followup.send(f"🏓 Pong! ความเร็วตอบสนอง: `{latency}ms` (Vault Key Confirmed)")
    await interaction.response.send_modal(VaultKeyModal("user", action))

@bot.tree.command(name="kick", description="เตะสมาชิกออกจากเซิร์ฟเวอร์")
@app_commands.checks.has_permissions(kick_members=True)
async def kick(interaction: discord.Interaction, member: discord.Member, reason: str = "ไม่ระบุ"):
    async def action(i: discord.Interaction):
        await member.kick(reason=reason)
        await i.followup.send(f"✅ เตะ {member.mention} เรียบร้อยแล้ว (เหตุผล: {reason})")
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="ban", description="แบนสมาชิก")
@app_commands.checks.has_permissions(ban_members=True)
async def ban(interaction: discord.Interaction, member: discord.Member, reason: str = "ไม่ระบุ"):
    async def action(i: discord.Interaction):
        await member.ban(reason=reason)
        await i.followup.send(f"✅ แบน {member.mention} เรียบร้อยแล้ว (เหตุผล: {reason})")
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="banid", description="แบนสมาชิกด้วย Discord ID")
@app_commands.checks.has_permissions(ban_members=True)
async def banid(interaction: discord.Interaction, user_id: str, reason: str = "ไม่ระบุ"):
    async def action(i: discord.Interaction):
        try:
            uid = int(user_id)
            user = await bot.fetch_user(uid)
            await interaction.guild.ban(user, reason=reason)
            await i.followup.send(f"✅ แบนผู้ใช้ ID `{uid}` ({user.name}) เรียบร้อยแล้ว")
        except ValueError:
            await i.followup.send("⛔ กรุณากรอก Discord ID เป็นตัวเลขเท่านั้น", ephemeral=True)
        except discord.NotFound:
            await i.followup.send("⛔ ไม่พบผู้ใช้จาก ID ดังกล่าว", ephemeral=True)
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="timeout", description="จำกัดเวลาสมาชิก")
@app_commands.checks.has_permissions(moderate_members=True)
async def timeout(interaction: discord.Interaction, member: discord.Member, minutes: int, reason: str = "ไม่ระบุ"):
    async def action(i: discord.Interaction):
        await member.timeout(timedelta(minutes=minutes), reason=reason)
        await i.followup.send(f"⏰ จำกัดเวลา {member.mention} เป็นเวลา {minutes} นาที")
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="setnick", description="เปลี่ยนชื่อเล่นของสมาชิก")
@app_commands.checks.has_permissions(manage_nicknames=True)
async def setnick(interaction: discord.Interaction, member: discord.Member, new_nickname: str):
    async def action(i: discord.Interaction):
        await member.edit(nick=new_nickname)
        await i.followup.send(f"🏷️ เปลี่ยนชื่อ {member.mention} เป็น **{new_nickname}** เรียบร้อยแล้ว")
    await interaction.response.send_modal(VaultKeyModal("admin", action))

@bot.tree.command(name="leaderboard", description="ดูตารางอันดับผู้ที่มีเลเวลสูงสุดในเซิร์ฟเวอร์")
async def leaderboard(interaction: discord.Interaction):
    async def action(i: discord.Interaction):
        conn = sqlite3.connect(DB_FILE)
        cursor = conn.cursor()
        cursor.execute("SELECT user_id, level, xp FROM user_levels WHERE guild_id = ? ORDER BY level DESC, xp DESC LIMIT 10",
                       (i.guild.id,))
        rows = cursor.fetchall()
        conn.close()

        if not rows:
            await i.followup.send("🏆 ยังไม่มีข้อมูลตารางอันดับในเซิร์ฟเวอร์นี้", ephemeral=True)
            return

        embed = discord.Embed(title=f"🏆 Top 10 Leaderboard - {i.guild.name}", color=discord.Color.gold())
        for idx, (uid, lvl, xp) in enumerate(rows, start=1):
            user = i.guild.get_member(uid)
            name = user.display_name if user else f"User ID {uid}"
            embed.add_field(name=f"#{idx} {name}", value=f"Level `{lvl}` | XP `{xp}`", inline=False)

        await i.followup.send(embed=embed, ephemeral=True)

    await interaction.response.send_modal(VaultKeyModal("user", action))

# ==============================
# START BOT
# ==============================
if __name__ == "__main__":
    if not TOKEN:
        raise RuntimeError("กรุณาตั้งค่า DISCORD_TOKEN ในไฟล์ .env ก่อนเริ่มทำงาน")
    bot.run(TOKEN)
