import discord
import aiohttp
from discord.ext import commands
from discord.ui import View, Button
import json
import datetime

# Token du bot Discord
DISCORD_TOKEN = ''

allowed_channel_ids = [1264734425137283223]

# URL de l'API du bot Telegram
TELEGRAM_API_URL = ''

# ID utilisateur de l'administrateur principal (vous)
ADMIN_USER_ID = '560087705753485313'

# Chemin des fichiers pour sauvegarder les données
USER_CREDITS_FILE = 'user_credits.json'
ADMIN_USERS_FILE = 'admin_users.json'
CLAIMED_FILE = 'claimed.json'

# ID du salon de logs
LOGS_CHANNEL_ID = 1259852287946260581  # Remplacez par l'ID de votre salon de logs Discord

# Configurer les intents
intents = discord.Intents.default()
intents.message_content = True
intents.members = True

# Initialiser le bot avec les intents
bot = commands.Bot(command_prefix='!', intents=intents, help_command=None)

# Charger les crédits des utilisateurs depuis un fichier (s'il existe)
try:
    with open(USER_CREDITS_FILE, 'r') as f:
        user_credits = json.load(f)
except FileNotFoundError:
    user_credits = {}

# Charger les administrateurs depuis un fichier (s'il existe)
try:
    with open(ADMIN_USERS_FILE, 'r') as f:
        admin_users = json.load(f)
except FileNotFoundError:
    admin_users = []

# Charger les données de réclamation de crédits depuis un fichier (s'il existe)
try:
    with open(CLAIMED_FILE, 'r') as f:
        claimed_credits = json.load(f)
except FileNotFoundError:
    claimed_credits = {}

# Événement pour notifier quand le bot est prêt
@bot.event
async def on_ready():
    print(f'{bot.user} est connecté.')

# Fonction pour envoyer un message de log
async def send_log_message(ctx, command, message):
    log_channel = await bot.fetch_channel(LOGS_CHANNEL_ID)

    embed = discord.Embed(
        title=f"Commande {command} exécutée",
        description=message,
        color=0x00ff00,
        timestamp=datetime.datetime.now(datetime.timezone.utc)  # Updated
    )
    embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur

    await log_channel.send(embed=embed)

# Commande pour afficher les commandes disponibles
@bot.command(name='help')
async def commands_command(ctx):
    embed = discord.Embed(title="Searcher #2k24 - HELP", description="Liste des commandes disponibles", color=0xff0000)
    embed.add_field(name="- 🔎 **!search <query>**", value="• Recherche des informations via l'API", inline=False)
    embed.add_field(name="- 💰 **!balance**", value="• Affiche le nombre de crédits restants", inline=False)
    embed.add_field(name="- 🏆 **!leaderboard**", value="• Affiche le classement des 10 premier utilisateurs en fonction des crédits", inline=False)
    embed.add_field(name="- 💸 **!claim**", value="• Réclame 10 crédits une fois par jour", inline=False)
    embed.add_field(name="- ➕**!addcredits @user amount**", value="• ajouter des crédits à un utilisateur (Admin uniquement)", inline=False)
    embed.add_field(name="- 🔑 **!wl @user**", value="• Ajouter un utilisateur à la liste des administrateurs (Admin uniquement)", inline=False)
    embed.add_field(name="- 🚫 **!unwl @user**", value="• Retirer un utilisateur de la liste des administrateurs (Admin uniquement)", inline=False)
    embed.add_field(name="- 🗑 **!clear <nombre>**", value="• Supprime le nombre spécifié de messages (Admin uniquement)", inline=False)
    
    embed.add_field(name="🔎 Les données suivantes sont disponibles pour la recherche:", value=(
        "📩 E-mail:                23.783.443.383\n"
        "👤 Nom et prénom:         9.702.624.213\n"
        "📞 Téléphone:             8.407.695.879\n"
        "👤 pseudo:                4.589.069.938\n"
        "🃏 numéro de document:    2.949.787.070\n"
        "🔑 Mot de passe:          2.777.624.717\n"
        "ⓕ Facebook ID:            824.934.156\n"
        "🎯 IP:                    502.264.741\n"
        "🚘 Numéro de véhicule:    393.199.156\n"
        "🔗 Lien:                  301.684.327\n"
        "🏢 Nom de l'entreprise:   297.446.674\n"
        "✈️ ID de télégramme:      154.968.917\n"
        "🌐 Domaine:               84.443.741\n"
        "📷 Identifiant Instagram: 45.089.233"
    ), inline=False)
    await ctx.send(embed=embed)

# Mapping des types de données à leurs emojis
data_emojis = {
    "Email": "📧",
    "Password": "🔑",
    "NickName": "👤",
    "RegDate": "📅",
    "Name": "👤",
    "Phone": "📞",
    "Document Number": "🃏",
    "Facebook ID": "ⓕ",
    "IP": "🎯",
    "Vehicle Number": "🚘",
    "Link": "🔗",
    "Company Name": "🏢",
    "Telegram ID": "✈️",
    "Domain": "🌐",
    "Instagram ID": "📷"
}

@bot.command(name='search')
async def search_command(ctx, *, query):
    await send_log_message(ctx, "search", f"Recherche effectuée : {query}")
    user_id = str(ctx.author.id)

    # Vérifier les crédits de l'utilisateur
    if user_credits.get(user_id, 0) <= 0:
        embed = discord.Embed(
            title="Crédits Insuffisants",
            description="Vous n'avez pas assez de crédits pour effectuer cette recherche.",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur
        await ctx.send(embed=embed)
        return

    # Préparer les données de la requête pour l'API de Telegram
    data = {
        "token": "",  # Remplacez par votre propre token Telegram
        "request": query,
        "limit": 100,
        "lang": "fr"
    }

    try:
        # Envoyer la requête à l'API de Telegram de manière asynchrone
        async with aiohttp.ClientSession() as session:
            async with session.post(TELEGRAM_API_URL, json=data) as response:
                if response.status == 200:
                    api_response = await response.json()

                    # Vérifier si la clé 'NumOfResults' est supérieure à zéro
                    if api_response.get('NumOfResults', 0) > 0:
                        # Décrémenter les crédits de l'utilisateur
                        user_credits[user_id] -= 1

                        # Créer un embed pour chaque résultat
                        results = []
                        for source, result in api_response['List'].items():
                            for data in result['Data']:
                                embed = discord.Embed(title=f"Résultat de {source} 📊", color=0x3498db)
                                for key, value in data.items():
                                    emoji = data_emojis.get(key, "➡️")
                                    embed.add_field(name=f"{emoji} {key}", value=value, inline=False)
                                results.append(embed)

                        # Pagination des résultats
                        current_page = 0
                        total_pages = len(results)

                        # Envoyer la confirmation dans le canal
                        confirmation_embed = discord.Embed(
                            title="・「💬」Recherche Envoyée",
                            description="Votre recherche a été envoyée en MP.",
                            color=0x2ecc71
                        )
                        confirmation_embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur
                        confirmation_embed.set_footer(text="Searcher # 2K24")
                        await ctx.send(embed=confirmation_embed)

                        # Envoyer les résultats en MP avec pagination
                        message = await ctx.author.send(embed=results[current_page].set_footer(text=f"Page {current_page + 1}/{total_pages}"))

                        if len(results) > 1:
                            view = View()

                            async def next_callback(interaction):
                                nonlocal current_page
                                current_page += 1
                                if current_page >= len(results):
                                    current_page = 0
                                await interaction.response.edit_message(embed=results[current_page].set_footer(text=f"Page {current_page + 1}/{total_pages}"))

                            async def previous_callback(interaction):
                                nonlocal current_page
                                current_page -= 1
                                if current_page < 0:
                                    current_page = len(results) - 1
                                await interaction.response.edit_message(embed=results[current_page].set_footer(text=f"Page {current_page + 1}/{total_pages}"))

                            next_button = Button(label="Suivant ➡️", style=discord.ButtonStyle.primary)
                            next_button.callback = next_callback
                            previous_button = Button(label="⬅️ Précédent", style=discord.ButtonStyle.primary)
                            previous_button.callback = previous_callback

                            view.add_item(previous_button)
                            view.add_item(next_button)

                            await message.edit(view=view)
                    else:
                        embed = discord.Embed(
                            title="Aucun Résultat",
                            description="Aucun résultat trouvé pour votre recherche.",
                            color=0xff0000
                        )
                        embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur
                        await ctx.send(embed=embed)
                else:
                    embed = discord.Embed(
                        title="Erreur de Recherche",
                        description="Une erreur s'est produite lors de la recherche. Veuillez réessayer plus tard.",
                        color=0xff0000
                    )
                    embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur
                    await ctx.send(embed=embed)
    except Exception as e:
        print(f"Erreur lors de la recherche : {e}")
        embed = discord.Embed(
            title="Aucun Résultat",
            description="Aucun résultat trouvé pour votre recherche.",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur
        await ctx.send(embed=embed)

# Fonction pour envoyer un message de log
async def send_log_message(ctx, command_name, details):
    timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    embed = discord.Embed(
        title="Commande exécutée",
        description=f"Utilisateur : {ctx.author.mention}\nCommande : {command_name}\nDétails : {details}",
        color=0x00ff00,
        timestamp=datetime.datetime.utcnow()
    )
    embed.set_thumbnail(url=ctx.author.avatar.url)

    logs_channel = bot.get_channel(int(LOGS_CHANNEL_ID))
    if logs_channel:
        await logs_channel.send(embed=embed)
    else:
        print(f"Erreur : Impossible de trouver le canal de logs avec l'ID {LOGS_CHANNEL_ID}")

# Commande pour afficher le solde de crédits d'un utilisateur
@bot.command(name='balance')
async def balance_command(ctx, user: discord.Member = None):
    await send_log_message(ctx, "balance", f"Vérification du solde des crédits")

    if user is None:
        # Utilisateur sans mention, afficher son propre solde
        user_id = str(ctx.author.id)
        user_name = f"{ctx.author.name}#{ctx.author.discriminator}"
        thumbnail_url = ctx.author.avatar.url
    else:
        # Utilisateur mentionné, afficher son solde
        user_id = str(user.id)
        user_name = f"{user.name}#{user.discriminator}"
        thumbnail_url = user.avatar.url

    # Vérifier si l'utilisateur a des crédits
    if user_id in user_credits:
        embed = discord.Embed(
            title=f"Solde des Crédits de {user_name}",
            description=f"Il vous reste {user_credits[user_id]} crédits.",
            color=0x00ff00
        )
    else:
        embed = discord.Embed(
            title=f"Solde des Crédits de {user_name}",
            description=f"Cet utilisateur n'a pas encore de crédits.",
            color=0xff0000
        )

    embed.set_thumbnail(url=thumbnail_url)  # Ajouter l'image de profil de l'utilisateur
    await ctx.send(embed=embed)

# Commande pour ajouter un utilisateur à la liste des administrateurs
@bot.command(name='wl')
async def whitelist_command(ctx, user: discord.Member):
    await send_log_message(ctx, "wl", f"Ajout de {user.name}#{user.discriminator} ({user.id}) à la liste des administrateurs")

    if str(ctx.author.id) != ADMIN_USER_ID:
        embed = discord.Embed(
            title="Autorisation Refusée",
            description="Vous n'êtes pas autorisé à utiliser cette commande.",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)
        await ctx.send(embed=embed)
        return

    if str(user.id) in admin_users:
        embed = discord.Embed(
            title="Déjà Administrateur",
            description=f"{user.name}#{user.discriminator} est déjà dans la liste des administrateurs.",
            color=0xff0000
        )
    else:
        admin_users.append(str(user.id))
        embed = discord.Embed(
            title="Utilisateur Ajouté aux Administrateurs",
            description=f"{user.name}#{user.discriminator} a été ajouté à la liste des administrateurs.",
            color=0x00ff00
        )

        # Enregistrer les administrateurs mis à jour dans le fichier
        with open(ADMIN_USERS_FILE, 'w') as f:
            json.dump(admin_users, f)

    embed.set_thumbnail(url=ctx.author.avatar.url)
    await ctx.send(embed=embed)


# Commande pour retirer un utilisateur de la liste des administrateurs
@bot.command(name='unwl')
async def unwhitelist_command(ctx, user: discord.Member):
    if str(ctx.author.id) != ADMIN_USER_ID:
        embed = discord.Embed(
            title="Autorisation Refusée",
            description="Vous n'êtes pas autorisé à utiliser cette commande.",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)
        await ctx.send(embed=embed)
        return

    if str(user.id) in admin_users:
        admin_users.remove(str(user.id))
        embed = discord.Embed(
            title="Utilisateur Retiré des Administrateurs",
            description=f"{user.name}#{user.discriminator} a été retiré de la liste des administrateurs.",
            color=0x00ff00
        )

        # Enregistrer les administrateurs mis à jour dans le fichier
        with open(ADMIN_USERS_FILE, 'w') as f:
            json.dump(admin_users, f)
    else:
        embed = discord.Embed(
            title="Utilisateur Non Trouvé dans la Liste des Administrateurs",
            description=f"{user.name}#{user.discriminator} n'était pas dans la liste des administrateurs.",
            color=0xff0000
        )

    embed.set_thumbnail(url=ctx.author.avatar.url)
    await ctx.send(embed=embed)

# Commande pour afficher le leaderboard des utilisateurs en fonction des crédits
@bot.command(name='leaderboard')
async def leaderboard_command(ctx):
    await send_log_message(ctx, "leaderboard", "Affichage du leaderboard")

    # Trier les utilisateurs par crédits (du plus élevé au plus bas)
    sorted_users = sorted(user_credits.items(), key=lambda x: x[1], reverse=True)

    # Créer un embed pour afficher le leaderboard
    embed = discord.Embed(
        title="Leaderboard - Top 10 👑",
        description="Classement des 10 premier utilisateurs en fonction de leurs crédits",
        color=0x3498db
    )

    # Ajouter chaque utilisateur au leaderboard
    for index, (user_id, credits) in enumerate(sorted_users[:10]):
        member = bot.get_user(int(user_id))
        if member is not None:
            embed.add_field(
                name=f"{index + 1}. {member.name}#{member.discriminator}",
                value=f"Crédits : {credits}",
                inline=False
            )

    embed.set_thumbnail(url=ctx.guild.icon.url)  # Ajouter l'icône du serveur
    await ctx.send(embed=embed)

# Commande pour réclamer des crédits une fois par jour
@bot.command(name='claim')
async def claim_command(ctx):
    await send_log_message(ctx, "claim", "Réclamation de 10 crédits")

    user_id = str(ctx.author.id)
    current_time = datetime.datetime.now().strftime("%Y-%m-%d")

    # Vérifier si l'utilisateur a déjà réclamé aujourd'hui
    if user_id in claimed_credits and claimed_credits[user_id] == current_time:
        embed = discord.Embed(
            title="Réclamation déjà effectuée",
            description="Vous avez déjà réclamé vos crédits aujourd'hui.",
            color=0xff0000
        )
    else:
        # Ajouter 10 crédits à l'utilisateur
        if user_id in user_credits:
            user_credits[user_id] += 10
        else:
            user_credits[user_id] = 10

        # Mettre à jour la date de la réclamation
        claimed_credits[user_id] = current_time

        # Enregistrer les crédits et les réclamations dans les fichiers
        with open(USER_CREDITS_FILE, 'w') as f:
            json.dump(user_credits, f)

        with open(CLAIMED_FILE, 'w') as f:
            json.dump(claimed_credits, f)

        embed = discord.Embed(
            title="Crédits Réclamés avec Succès",
            description="Vous avez réclamé 10 crédits avec succès.",
            color=0x00ff00
        )

    embed.set_thumbnail(url=ctx.author.avatar.url)  # Ajouter l'image de profil de l'utilisateur
    await ctx.send(embed=embed)

# Commande pour ajouter des crédits à un utilisateur (Admin uniquement)
@bot.command(name='addcredits', aliases=['add'])
async def add_credits_command(ctx, user: discord.Member, amount: int):
    await send_log_message(ctx, "addcredits", f"Ajout de {amount} crédits à {user.name}#{user.discriminator}")

    if str(ctx.author.id) != ADMIN_USER_ID:
        embed = discord.Embed(
            title="Autorisation Refusée",
            description="Vous n'êtes pas autorisé à utiliser cette commande.",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)
        await ctx.send(embed=embed)
        return

    # Ajouter des crédits à l'utilisateur spécifié
    user_id = str(user.id)
    if user_id in user_credits:
        user_credits[user_id] += amount
    else:
        user_credits[user_id] = amount

    # Enregistrer les crédits dans le fichier
    with open(USER_CREDITS_FILE, 'w') as f:
        json.dump(user_credits, f)

    embed = discord.Embed(
        title="Crédits Ajoutés avec Succès",
        description=f"{amount} crédits ont été ajoutés à {user.name}#{user.discriminator}.",
        color=0x00ff00
    )
    embed.set_thumbnail(url=ctx.author.avatar.url)
    await ctx.send(embed=embed)

# Commande pour supprimer un nombre spécifié de messages (Admin uniquement)
@bot.command(name='clear')
async def clear_command(ctx, amount: int):
    await send_log_message(ctx, "clear", f"Suppression de {amount} messages")

    if str(ctx.author.id) != ADMIN_USER_ID:
        embed = discord.Embed(
            title="Autorisation Refusée",
            description="Vous n'êtes pas autorisé à utiliser cette commande.",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)
        await ctx.send(embed=embed)
        return

    # Vérifier si le nombre de messages à supprimer est valide
    if amount <= 0:
        embed = discord.Embed(
            title="Nombre Invalide",
            description="Veuillez spécifier un nombre de messages à supprimer valide (supérieur à 0).",
            color=0xff0000
        )
        embed.set_thumbnail(url=ctx.author.avatar.url)
        await ctx.send(embed=embed)
        return

    # Supprimer les messages dans le canal actuel
    deleted_messages = await ctx.channel.purge(limit=amount + 1)

    embed = discord.Embed(
        title="Messages Supprimés avec Succès",
        description=f"{len(deleted_messages) - 1} messages ont été supprimés.",
        color=0x00ff00
    )
    embed.set_thumbnail(url=ctx.author.avatar.url)
    await ctx.send(embed=embed)

# Gérer les erreurs de commande
@bot.event
async def on_command_error(ctx, error):
    if isinstance(error, commands.CommandNotFound):
        embed = discord.Embed(
            title="Commande Introuvable",
            description="La commande que vous avez entrée est introuvable. Utilisez !help pour afficher la liste des commandes disponibles.",
            color=0xff0000
        )
        await ctx.send(embed=embed)
    elif isinstance(error, commands.MissingRequiredArgument):
        embed = discord.Embed(
            title="Argument Manquant",
            description="Vous n'avez pas fourni tous les arguments nécessaires pour cette commande.",
            color=0xff0000
        )
        await ctx.send(embed=embed)
    elif isinstance(error, commands.BadArgument):
        embed = discord.Embed(
            title="Mauvais Argument",
            description="Un ou plusieurs arguments fournis ne sont pas valides.",
            color=0xff0000
        )
        await ctx.send(embed=embed)
    elif isinstance(error, commands.MissingPermissions):
        embed = discord.Embed(
            title="Permissions Insuffisantes",
            description="Vous ne disposez pas des autorisations nécessaires pour exécuter cette commande.",
            color=0xff0000
        )
        await ctx.send(embed=embed)
    else:
        embed = discord.Embed(
            title="Erreur de Commande",
            description="Une erreur s'est produite lors de l'exécution de cette commande.",
            color=0xff0000
        )
        await ctx.send(embed=embed)

# Lancer le bot Discord
bot.run(DISCORD_TOKEN)
