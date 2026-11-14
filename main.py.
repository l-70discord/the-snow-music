import os
import asyncio
import discord
from discord import app_commands
from discord.ext import commands
import yt_dlp

TOKEN = os.getenv("BOT_TOKEN")

intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)

queues = {}
text_channels = {}
synced = False

YDL_OPTIONS = {
    "format": "bestaudio/best",
    "quiet": True,
    "noplaylist": True,
    "default_search": "ytsearch1:"
}

def extract_audio(query):
    with yt_dlp.YoutubeDL(YDL_OPTIONS) as ydl:
        return ydl.extract_info(query, download=False)

async def get_audio(query):
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, extract_audio, query)

async def play_next(guild):
    vc = guild.voice_client
    if not vc or not vc.is_connected():
        return

    queue = queues.get(guild.id, [])
    if not queue:
        return

    item = queue.pop(0)

    try:
        info = await get_audio(item["url"])

        if "entries" in info:
            info = info["entries"][0]

        source = discord.FFmpegPCMAudio(
            info["url"],
            before_options="-reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5",
            options="-vn"
        )

        def after_play(error):
            if error:
                print("Audio error:", error)

            asyncio.run_coroutine_threadsafe(
                play_next(guild), bot.loop
            )

        vc.play(source, after=after_play)

        channel = text_channels.get(guild.id)
        if channel:
            await channel.send(
                f"🎵 الآن يتم تشغيل: **{info.get('title', 'Unknown')}**"
            )

    except Exception as error:
        print("Playback error:", error)
        channel = text_channels.get(guild.id)
        if channel:
            await channel.send("تعذّر تشغيل هذا المقطع، سأحاول التالي.")
        await play_next(guild)


@bot.event
async def on_ready():
    global synced
    if not synced:
        await bot.tree.sync()
        synced = True

    print(f"Logged in as {bot.user}")


@bot.tree.command(name="play", description="تشغيل أغنية")
@app_commands.describe(
    query="اسم الأغنية أو رابطها",
    platform="مصدر البحث"
)
@app_commands.choices(platform=[
    app_commands.Choice(name="YouTube", value="youtube"),
    app_commands.Choice(name="SoundCloud", value="soundcloud")
])
async def play(
    interaction: discord.Interaction,
    query: str,
    platform: app_commands.Choice[str]
):
    await interaction.response.defer(thinking=True)

    if not interaction.user.voice:
        await interaction.followup.send(
            "يجب أن تدخل رومًا صوتيًا أولًا."
        )
        return

    guild = interaction.guild
    channel = interaction.user.voice.channel

    vc = guild.voice_client

    if vc is None:
        vc = await channel.connect()
    elif vc.channel != channel:
        await vc.move_to(channel)

    text_channels[guild.id] = interaction.channel

    if query.startswith("http"):
        search = query
    elif platform.value == "soundcloud":
        search = "scsearch1:" + query
    else:
        search = "ytsearch1:" + query

    queue = queues.setdefault(guild.id, [])
    queue.append({"url": search})

    await interaction.followup.send(
        f"🎶 تمت إضافة **{query}** إلى قائمة التشغيل."
    )

    if not vc.is_playing() and not vc.is_paused():
        await play_next(guild)


@bot.tree.command(name="skip", description="تخطي الأغنية")
async def skip(interaction: discord.Interaction):
    vc = interaction.guild.voice_client

    if vc and vc.is_playing():
        vc.stop()
        await interaction.response.send_message("⏭️ تم التخطي.")
    else:
        await interaction.response.send_message("لا توجد أغنية تعمل.")


@bot.tree.command(name="pause", description="إيقاف مؤقت")
async def pause(interaction: discord.Interaction):
    vc = interaction.guild.voice_client

    if vc and vc.is_playing():
        vc.pause()
        await interaction.response.send_message("⏸️ تم الإيقاف المؤقت.")
    else:
        await interaction.response.send_message("لا توجد أغنية تعمل.")


@bot.tree.command(name="resume", description="استكمال الأغنية")
async def resume(interaction: discord.Interaction):
    vc = interaction.guild.voice_client

    if vc and vc.is_paused():
        vc.resume()
        await interaction.response.send_message("▶️ تم الاستكمال.")
    else:
        await interaction.response.send_message("لا توجد أغنية متوقفة.")


@bot.tree.command(name="stop", description="إيقاف الموسيقى")
async def stop(interaction: discord.Interaction):
    vc = interaction.guild.voice_client
    queues[interaction.guild.id] = []

    if vc:
        vc.stop()

    await interaction.response.send_message("⏹️ تم إيقاف الموسيقى ومسح القائمة.")


@bot.tree.command(name="queue", description="عرض قائمة الأغاني")
async def show_queue(interaction: discord.Interaction):
    queue = queues.get(interaction.guild.id, [])

    if not queue:
        await interaction.response.send_message("قائمة الانتظار فارغة.")
        return

    items = "\n".join(
        f"{i+1}. {item['url']}"
        for i, item in enumerate(queue[:10])
    )

    await interaction.response.send_message(
        f"🎶 **قائمة الانتظار:**\n{items}"
    )


@bot.tree.command(name="leave", description="إخراج البوت من الروم")
async def leave(interaction: discord.Interaction):
    vc = interaction.guild.voice_client

    if vc:
        queues[interaction.guild.id] = []
        await vc.disconnect()
        await interaction.response.send_message("👋 غادرت الروم الصوتي.")
    else:
        await interaction.response.send_message("أنا لست في روم صوتي.")


if not TOKEN:
    raise RuntimeError("BOT_TOKEN is missing!")

bot.run(TOKEN)
