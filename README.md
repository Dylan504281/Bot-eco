import discord
from discord.ext import commands
import random
import os
import asyncio
import youtube_dl

# Configuration Constants
DEFAULT_PREFIX = "#"
BOT_ACTIVITY = "EcoBot | #ayuda para comandos"
TOKEN_ENV_VAR = "DISCORD_TOKEN"

# Intents Configuration
def setup_intents():
    intents = discord.Intents.default()
    intents.members = True
    intents.message_content = True
    intents.reactions = True
    return intents

# EcoBot Messages
class EcoMessages:
    RECYCLING_TIPS = [
        "Apaga las luces cuando salgas de una habitación.",
        "Usa bolsas reutilizables en lugar de bolsas de plástico.",
        "Reduce el uso de agua al ducharte y al lavar los platos.",
        "Recicla el papel, plástico y vidrio siempre que sea posible.",
        "Planta un árbol o cuida de las plantas en tu área."
    ]
    
    RECYCLABLE_ITEMS = (
        "✅ **Puedes reciclar:**\n"
        "- Papel y cartón limpios\n"
        "- Plásticos con símbolo de reciclaje (botellas, envases)\n"
        "- Vidrio (botellas, frascos)\n"
        "- Metales (latas de aluminio, latas de acero)"
    )
    
    NON_RECYCLABLE_ITEMS = (
        "❌ **No se deben reciclar:**\n"
        "- Plásticos sin símbolo de reciclaje\n"
        "- Residuos orgánicos o comida\n"
        "- Cartón sucio o húmedo\n"
        "- Bolsas de plástico de un solo uso"
    )
    
    CRAFTS_IDEAS = (
        "♻ **Manualidades con material reciclado:**\n"
        "- Macetas con botellas de plástico\n"
        "- Organizador de escritorio con cajas de cartón\n"
        "- Adornos navideños con papel reciclado\n"
        "- Juguetes para mascotas con rollos de papel higiénico\n"
        "- Bolsas reutilizables con camisetas viejas"
    )
    
    RECYCLING_BENEFITS = (
        "🌍 **Beneficios del reciclaje:**\n"
        "- Reduce residuos en vertederos\n"
        "- Conserva recursos naturales\n"
        "- Disminuye la contaminación\n"
        "- Fomenta la economía circular\n"
        "- Combate el cambio climático"
    )

# Trivia Questions
TRIVIA_QUESTIONS = [
    ("¿Cuál es la capital de Francia?", "París"),
    ("¿Cuál es la montaña más alta del mundo?", "Monte Everest"),
    ("¿Cuál es el planeta más grande de nuestro sistema solar?", "Júpiter"),
    ("¿En qué año se lanzó el primer iPhone?", "2007"),
    ("¿Quién pintó la Mona Lisa?", "Leonardo da Vinci"),
]

# Emoticon Dictionary
EMOTICON_DICT = {
    '$feliz': '',
    '$triste': '☹️',
    '$enojado': '',
    '$sorprendido': '',
    '$pensativo': '',
}

# YouTube DL Configuration
YDL_OPTIONS = {
    'format': 'bestaudio/best',
    'outtmpl': '%(extractor)s-%(id)s-%(title)s.%(ext)s',
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
    'options': '-vn',
}

# Bot Initialization
class EcoBot(commands.Bot):
    def __init__(self):
        super().__init__(
            command_prefix=DEFAULT_PREFIX,
            intents=setup_intents(),
            help_command=None
        )
    
    async def on_ready(self):
        print(f'Bot conectado como: {self.user} (ID: {self.user.id})')
        activity = discord.Game(
            name=f"{BOT_ACTIVITY} | Con agradecimiento a la Bióloga Karla Aparicio"
        )
        await self.change_presence(activity=activity)

# Music Cog
class Music(commands.Cog):
    def __init__(self, bot):
        self.bot = bot
        self.ydl = youtube_dl.YoutubeDL(YDL_OPTIONS)

    async def ensure_voice(self, ctx):
        if ctx.voice_client is None:
            if ctx.author.voice:
                await ctx.author.voice.channel.connect()
            else:
                await ctx.send("No estás conectado a un canal de voz.")
                raise commands.CommandError("Author not connected to a voice channel.")
        elif ctx.voice_client.is_playing():
            ctx.voice_client.stop()

    @commands.command()
    async def join(self, ctx, *, channel: discord.VoiceChannel):
        """Únete a un canal de voz"""
        if ctx.voice_client is not None:
            return await ctx.voice_client.move_to(channel)
        await channel.connect()

    @commands.command()
    async def play(self, ctx, *, query):
        """Reproduce un archivo local"""
        source = discord.PCMVolumeTransformer(discord.FFmpegPCMAudio(query))
        ctx.voice_client.play(source, after=lambda e: print(f'Error: {e}') if e else None)
        await ctx.send(f'Reproduciendo: {query}')

    @commands.command()
    async def yt(self, ctx, *, url):
        """Reproduce desde una URL de YouTube"""
        async with ctx.typing():
            player = await YTDLSource.from_url(url, loop=self.bot.loop)
            ctx.voice_client.play(player, after=lambda e: print(f'Error: {e}') if e else None)
        await ctx.send(f'Reproduciendo: {player.title}')

    @commands.command()
    async def stop(self, ctx):
        """Detiene la reproducción y desconecta el bot"""
        await ctx.voice_client.disconnect()

class YTDLSource(discord.PCMVolumeTransformer):
    def __init__(self, source, *, data, volume=0.5):
        super().__init__(source, volume)
        self.data = data
        self.title = data.get('title')
        self.url = data.get('url')

    @classmethod
    async def from_url(cls, url, *, loop=None, stream=False):
        loop = loop or asyncio.get_event_loop()
        data = await loop.run_in_executor(None, lambda: ytdl.extract_info(url, download=not stream))
        
        if 'entries' in data:
            data = data['entries'][0]
        
        filename = data['url'] if stream else ytdl.prepare_filename(data)
        return cls(discord.FFmpegPCMAudio(filename, **FFMPEG_OPTIONS), data=data)

# Utility Functions
def get_trivia_question():
    question, answer = random.choice(TRIVIA_QUESTIONS)
    return f"❓ Trivia: {question}"

def get_joke():
    jokes = [
        "¿Por qué los pájaros no usan Facebook? Porque ya tienen Twitter.",
        "¿Qué hace una abeja en el gimnasio? ¡Zum-ba!",
        "¿Cómo se llama el primo vegano de Bruce Lee? Broco Lee."
    ]
    return random.choice(jokes)

def has_role(member, role_name):
    return any(role.name == role_name for role in member.roles)

# Bot Commands
def setup_commands(bot):
    @bot.command(name="hola", help="Saluda al bot")
    async def greet(ctx):
        await ctx.send(f"¡Hola {ctx.author.mention}! ¿En qué puedo ayudarte hoy?")

    @bot.command(name="ayuda", help="Muestra todos los comandos disponibles")
    async def show_help(ctx):
        embed = discord.Embed(
            title="🛠 Comandos disponibles de EcoBot",
            description="Desarrollado con el asesoramiento científico de la Bióloga y Profesora Karla Aparicio",
            color=0x2ecc71
        )
        
        # Add command categories
        embed.add_field(
            name="🌱 Comandos Ecológicos",
            value="\n".join(
                f"`{DEFAULT_PREFIX}{cmd.name}` - {cmd.help}"
                for cmd in bot.commands
                if cmd.name in ["consejo", "reciclaje", "ecologia", "manualidades", "beneficios_reciclaje"]
            ),
            inline=False
        )
        
        embed.add_field(
            name="🎵 Comandos de Música",
            value="\n".join(
                f"`{DEFAULT_PREFIX}{cmd.name}` - {cmd.help}"
                for cmd in bot.commands
                if cmd.name in ["play", "yt", "stop", "join"]
            ),
            inline=False
        )
        
        embed.add_field(
            name="🎲 Comandos Diversión",
            value="\n".join(
                f"`{DEFAULT_PREFIX}{cmd.name}` - {cmd.help}"
                for cmd in bot.commands
                if cmd.name in ["trivia", "chiste", "moneda", "dado", "elegir"]
            ),
            inline=False
        )
        
        embed.set_footer(
            text="Con agradecimiento especial a la Bióloga Karla Aparicio por su asesoramiento científico"
        )
        
        await ctx.send(embed=embed)

    @bot.command(name="reciclaje", help="Explica la importancia del reciclaje")
    async def explain_recycling(ctx):
        """Proporciona información básica sobre reciclaje"""
        await ctx.send(
            "♻ **El reciclaje** es fundamental para:\n"
            "- Reducir la cantidad de residuos\n"
            "- Conservar recursos naturales\n"
            "- Minimizar nuestro impacto ambiental\n"
            "¡Cada pequeño gesto cuenta!"
        )

    @bot.command(name="consejo", help="Da un consejo ecológico aleatorio")
    async def random_tip(ctx):
        embed = discord.Embed(
            title="💡 Consejo ecológico",
            description=random.choice(EcoMessages.RECYCLING_TIPS),
            color=0x3498db
        )
        await ctx.send(embed=embed)

    @bot.command(name="ecologia", help="Explica el concepto de ecología")
    async def ecology_info(ctx):
        embed = discord.Embed(
            title="🌱 ¿Qué es la Ecología?",
            description=(
                "La **ecología** es la ciencia que estudia las interacciones entre "
                "los organismos y su entorno. Es fundamental para entender cómo "
                "nuestras acciones afectan el medio ambiente y cómo podemos vivir "
                "de manera más sostenible."
            ),
            color=0x27ae60
        )
        await ctx.send(embed=embed)

    @bot.command(name="manualidades", help="Muestra ideas para manualidades con material reciclado")
    async def recycling_crafts(ctx):
        embed = discord.Embed(
            title="✂ Manualidades Ecológicas",
            description=EcoMessages.CRAFTS_IDEAS,
            color=0xe67e22
        )
        await ctx.send(embed=embed)

    @bot.command(name="beneficios_reciclaje", help="Enumera los beneficios del reciclaje")
    async def recycling_benefits(ctx):
        embed = discord.Embed(
            title="🌟 Beneficios del Reciclaje",
            description=EcoMessages.RECYCLING_BENEFITS,
            color=0x9b59b6
        )
        await ctx.send(embed=embed)

    @bot.command(name="trivia", help="Pregunta de trivia aleatoria")
    async def trivia(ctx):
        await ctx.send(get_trivia_question())

    @bot.command(name="chiste", help="Cuenta un chiste aleatorio")
    async def joke(ctx):
        if has_role(ctx.author, "Joke Teller"):
            await ctx.send(get_joke())
        else:
            await ctx.send("No tienes permiso para usar este comando.")

    @bot.command(name="moneda", help="Lanza una moneda")
    async def coin_flip(ctx):
        await ctx.send("Cara" if random.randint(1, 2) == 1 else "Cruz")

    @bot.command(name="dado", help="Tira un dado (ej. #dado 2d6)")
    async def roll(ctx, dice: str):
        try:
            rolls, limit = map(int, dice.split('d'))
            result = ', '.join(str(random.randint(1, limit)) for _ in range(rolls))
            await ctx.send(result)
        except Exception:
            await ctx.send('Formato debe ser NdN (ej. 2d6)')

    @bot.command(name="elegir", help="Elige entre varias opciones")
    async def choose(ctx, *choices: str):
        await ctx.send(random.choice(choices))

# Error Handling
def setup_error_handling(bot):
    @bot.event
    async def on_command_error(ctx, error):
        if isinstance(error, commands.CommandNotFound):
            await ctx.send(f"Comando no encontrado. Usa `{DEFAULT_PREFIX}ayuda` para ver comandos.")
        elif isinstance(error, commands.MissingRequiredArgument):
            await ctx.send(f"Faltan argumentos. Usa `{DEFAULT_PREFIX}ayuda {ctx.command.name}` para ayuda.")
        else:
            await ctx.send("⚠ Error inesperado.")
            print(f"Error: {error}")

# Main Function
async def main():
    bot = EcoBot()
    setup_commands(bot)
    setup_error_handling(bot)
    await bot.add_cog(Music(bot))
    
    token = os.getenv(TOKEN_ENV_VAR)
    if not token:
        print(f"Error: Configura la variable de entorno {TOKEN_ENV_VAR}")
        return
    
    await bot.start(token)

if __name__ == "__main__":
    youtube_dl.utils.bug_reports_message = lambda: ''
    asyncio.run(main())
