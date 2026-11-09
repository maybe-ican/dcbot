import discord
from discord.ext import commands
import asyncio
import random
from datetime import datetime, timedelta

intents = discord.Intents.default()
intents.message_content = True
intents.members = True # Üye giriş/çıkışlarını takip edebilmek için şart!

bot = commands.Bot(command_prefix='!', intents=intents, help_command=None)

# Giriş-çıkış kanallarını tutmak için basit bir sözlük (Sunucu ID -> Kanal ID)
giris_cikis_kanallari = {}

@bot.event
async def on_ready():
    print(f'✅ {bot.user} aktif ve tüm sistemler kusursuz çalışıyor!')

# ------------------- GENEL YARDIM KOMUTU (!help / !yardim) -------------------
@bot.command(aliases=['yardim'])
async def help(ctx):
    embed = discord.Embed(
        title="✨ **BOXPVP BOT KOMUT MENÜSÜ** ✨",
        description="Sunucumuzda herkesin kullanabileceği genel komutlar:",
        color=0x3498db # Mavi
    )
    embed.add_field(
        name="🎮 **Genel Komutlar**",
        value="`!ip` veya `!sunucu` ➔ Sunucu IP adresini gösterir.\n`!help` veya `!yardim` ➔ Bu yardım menüsünü açar.",
        inline=False
    )
    embed.set_footer(text=f"{ctx.guild.name} • Keyifli Oyunlar Dileriz!", icon_url=ctx.guild.icon.url if ctx.guild and ctx.guild.icon else None)
    await ctx.send(embed=embed)

# ------------------- YETKİLİ YARDIM KOMUTU (!helpyetkili) -------------------
@bot.command()
@commands.has_permissions(administrator=True)
async def helpyetkili(ctx):
    embed = discord.Embed(
        title="🛠️ **YETKİLİ KOMUT MENÜSÜ** 🛠️",
        description="Sadece yöneticilerin kullanabileceği özel komutlar:",
        color=0xe67e22 # Turuncu
    )
    embed.add_field(
        name="🎁 **Çekiliş & Yönetim**",
        value="`!cekilis <saniye> <ödül>` ➔ Saniye saniye sayan canlı çekiliş başlatır.\n*Örnek:* `!cekilis 180 VIP Üyelik`",
        inline=False
    )
    embed.add_field(
        name="🚪 **Giriş-Çıkış Sistemi**",
        value="`!giriscikisayarla` ➔ Bulunduğun kanalı giriş-çıkış kanalı yapar.\n`!giriscikisunset` ➔ Giriş-çıkış sistemini kapatır/siler.\n`!giriscikistest gir/cik` ➔ Sistemi test etmeni sağlar.",
        inline=False
    )
    embed.set_footer(text="BoxPvP Yönetim Paneli")
    await ctx.send(embed=embed)

@helpyetkili.error
async def helpyetkili_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("⛔ **Hata:** Bu komutu görmek için `Yönetici` yetkisine sahip olmalısın!", delete_after=5)

# ------------------- IP KOMUTU -------------------
@bot.command(aliases=['sunucu'])
async def ip(ctx):
    embed = discord.Embed(
        title="⚔️ **BOXPVP SUNUCU BİLGİLERİ** ⚔️",
        description="Aşağıdaki IP adresini kopyalayarak sunucumuza hemen katılabilirsin!",
        color=0x9b59b6 # Mor
    )
    embed.add_field(name="📌 **Sunucu IP Adresi**", value="```boxpvp.marthost.net```", inline=False)
    embed.add_field(name="⚡ **Desteklenen Sürüm**", value="`1.18.x - 1.26.x`", inline=True)
    embed.add_field(name="🔥 **Oyun Modu**", value="`BoxPvP`", inline=True)
    await ctx.send(embed=embed)

# ------------------- GİRİŞ - ÇIKIŞ SİSTEMİ -------------------
@bot.command()
@commands.has_permissions(administrator=True)
async def giriscikisayarla(ctx):
    giris_cikis_kanallari[ctx.guild.id] = ctx.channel.id
    await ctx.send(f"✅ Başarılı! Artık tüm giriş ve çıkışlar bu kanalda ({ctx.channel.mention}) duyurulacak.")

@bot.command()
@commands.has_permissions(administrator=True)
async def giriscikisunset(ctx):
    if ctx.guild.id in giris_cikis_kanallari:
        del giris_cikis_kanallari[ctx.guild.id]
        await ctx.send("🗑️ Giriş-çıkış sistemi bu sunucu için başarıyla devre dışı bırakıldı.")
    else:
        await ctx.send("⚠️ Zaten aktif bir giriş-çıkış kanalı ayarlanmamış.")

@bot.command()
@commands.has_permissions(administrator=True)
async def giriscikistest(ctx, tur: str = "gir"):
    if tur.lower() == "gir":
        await on_member_join(ctx.author)
    elif tur.lower() == "cik":
        await on_member_remove(ctx.author)
    else:
        await ctx.send("Kullanım: `!giriscikistest gir` veya `!giriscikistest cik`")

@bot.event
async def on_member_join(member):
    kanal_id = giris_cikis_kanallari.get(member.guild.id)
    if kanal_id:
        kanal = member.guild.get_channel(kanal_id)
        if kanal:
            embed = discord.Embed(
                title="🎉 YENİ BİR OYUNCU KATILDI!",
                description=f"Sunucumuza Hoş Geldin {member.mention} Seninle Birlikte **{member.guild.member_count} Kişiyiz** 💗!",
                color=0x2ecc71 # Yeşil
            )
            embed.set_thumbnail(url=member.display_avatar.url)
            await kanal.send(embed=embed)

@bot.event
async def on_member_remove(member):
    kanal_id = giris_cikis_kanallari.get(member.guild.id)
    if kanal_id:
        kanal = member.guild.get_channel(kanal_id)
        if kanal:
            embed = discord.Embed(
                title="😢 BİR OYUNCU AYRILDI",
                description=f"Sunucumuzdan Ayrıldı {member.mention} Sensin **{member.guild.member_count} Kişiyiz** 😭",
                color=0xe74c3c # Kırmızı
            )
            embed.set_thumbnail(url=member.display_avatar.url)
            await kanal.send(embed=embed)

# ------------------- ÇEKİLİŞ BUTONU & 1 SANİYELİK MANTIK -------------------
class CekilisView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.katilimcilar = set()

    @discord.ui.button(label="Çekilişe Katıl", style=discord.ButtonStyle.blurple, emoji="🎉", custom_id="katil_btn")
    async def katil(self, interaction: discord.Interaction, button: discord.ui.Button):
        user_id = interaction.user.id
        
        if user_id in self.katilimcilar:
            # Çekilişten çıkması engellendi, bilgi veriliyor
            await interaction.response.send_message("⚠️ Zaten çekilişe katıldın! 🚀", ephemeral=True)
        else:
            self.katilimcilar.add(user_id)
            await interaction.response.send_message("🎉 Çekilişe başarıyla katıldın! Bol şans!", ephemeral=True)
            
            # Katılımcı sayısını anlık güncelle
            embed = interaction.message.embeds[0]
            for idx, field in enumerate(embed.fields):
                if "Katılımcı Sayısı" in field.name:
                    embed.set_field_at(idx, name="👥 **Katılımcı Sayısı**", value=f"`{len(self.katilimcilar)} Kişi`", inline=True)
                    break
            await interaction.message.edit(embed=embed)

# ------------------- ÇEKİLİŞ KOMUTU (!cekilis) -------------------
@bot.command()
@commands.has_permissions(administrator=True)
async def cekilis(ctx, saniye: int = None, *, odul: str = "Büyük Ödül"):
    if saniye is None or saniye <= 0:
        await ctx.send("⚠️ Lütfen geçerli bir süre girin! Örnek: `!cekilis 180 VIP Üyelik`")
        return

    bitis_zamani = datetime.now() + timedelta(seconds=saniye)
    tarih_format = bitis_zamani.strftime("%d.%m.%Y %H:%M")

    view = CekilisView()

    def make_embed(kalan_sn):
        saat = kalan_sn // 3600
        dakika = (kalan_sn % 3600) // 60
        saniye_kal = kalan_sn % 60
        sure_str = f"{saat:02d}:{dakika:02d}:{saniye_kal:02d}"

        emb = discord.Embed(
            title=f"🎁 **ÇEKİLİŞ: {odul.upper()}** 🎁",
            description="Çekilişe katılmak için aşağıdaki **🎉 Çekilişe Katıl** butonuna tıkla!",
            color=0xf1c40f # Altın Sarısı
        )
        emb.add_field(name="🕒 **Sonuçlanacağı Tarih/Saat**", value=f"`{tarih_format}`", inline=False)
        emb.add_field(name="⏳ **Kalan Süre**", value=f"`{sure_str}`", inline=True)
        emb.add_field(name="👥 **Katılımcı Sayısı**", value=f"`{len(view.katilimcilar)} Kişi`", inline=True)
        emb.add_field(name="👑 **Çekilişi Yapan**", value=ctx.author.mention, inline=False)
        emb.set_footer(text="Bol şanslar! • BoxPvP Yönetimi")
        return emb

    cekilis_msg = await ctx.send(embed=make_embed(saniye), view=view)
    await ctx.message.delete()

    # 1 saniye aralıklarla tıkır tıkır güncelleyen döngü
    kalan = saniye
    while kalan > 0:
        await asyncio.sleep(1)
        kalan -= 1
        try:
            await cekilis_msg.edit(embed=make_embed(kalan), view=view)
        except Exception:
            pass

    # --- ÇEKİLİŞ BİTİŞİ ---
    for item in view.children:
        item.disabled = True
    
    if len(view.katilimcilar) > 0:
        kazanan_id = random.choice(list(view.katilimcilar))
        kazanan = ctx.guild.get_member(kazanan_id)
        kazanan_text = kazanan.mention if kazanan else f"<@{kazanan_id}>"

        sonuc_embed = discord.Embed(
            title=f"🎉 **ÇEKİLİŞ SONUÇLANDI: {odul.upper()}** 🎉",
            description=f"🏆 **Kazanan:** {kazanan_text}\n\nTebrikler! Ödülünü almak için yetkililere ulaşabilirsin.",
            color=0x2ecc71
        )
        sonuc_embed.add_field(name="👥 **Toplam Katılan**", value=f"`{len(view.katilimcilar)} Kişi`", inline=True)
        sonuc_embed.add_field(name="🕒 **Bitiş Zamanı**", value=f"`{tarih_format}`", inline=True)
        
        await cekilis_msg.edit(embed=sonuc_embed, view=view)
        await ctx.send(f"🎊 Tebrikler {kazanan_text}! **{odul}** çekilişini sen kazandın!")
    else:
        iptal_embed = discord.Embed(
            title=f"❌ **ÇEKİLİŞ İPTAL EDİLDİ: {odul.upper()}**",
            description="Yeterli katılım olmadığı için (0 kişi) çekiliş kazanan belirlenmeden bitti.",
            color=0xe74c3c
        )
        await cekilis_msg.edit(embed=iptal_embed, view=view)

# Hata Yakalamalar
@cekilis.error
async def cekilis_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("⛔ **Hata:** Çekiliş başlatmak için `Yönetici` yetkisine sahip olmalısın!", delete_after=5)

@giriscikisayarla.error
async def gc_ayar_error(ctx, error):
    if isinstance(error, commands.MissingPermissions):
        await ctx.send("⛔ **Hata:** Bu komutu kullanmak için `Yönetici` yetkisine sahip olmalısın!", delete_after=5)

import os
bot.run(os.environ.get('DISCORD_TOKEN'))