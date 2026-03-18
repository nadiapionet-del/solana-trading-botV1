import os
import logging
import anthropic
from telegram import Update, constants
from telegram.ext import ApplicationBuilder, CommandHandler, MessageHandler, filters, ContextTypes

# ─── CONFIG ───────────────────────────────────────────────────────────────────
TELEGRAM_TOKEN = os.environ.get("TELEGRAM_TOKEN", "TON_TOKEN_ICI")
ANTHROPIC_API_KEY = os.environ.get("ANTHROPIC_API_KEY", "TA_CLE_API_ICI")
SOLSCAN_API_KEY = os.environ.get("SOLSCAN_API_KEY", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJjcmVhdGVkQXQiOjE3NzM3ODUzMTUzMDIsImVtYWlsIjoiZGl0cmF0dkBnbWFpbC5jb20iLCJhY3Rpb24iOiJ0b2tlbi1hcGkiLCJhcGlWZXJzaW9uIjoidjIiLCJpYXQiOjE3NzM3ODUzMTV9.n1wAN-7MgOEHCBQqZrcwsmpCSIHUpD0K1yQsK0eQSuE")
AUTHORIZED_USER = os.environ.get("AUTHORIZED_USER", "")  # Ton username Telegram (ex: zMnyy_)

# ─── LOGGING ──────────────────────────────────────────────────────────────────
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# ─── CLIENT ANTHROPIC ─────────────────────────────────────────────────────────
client = anthropic.Anthropic(api_key=ANTHROPIC_API_KEY)

# ─── HISTORIQUE DES CONVERSATIONS ─────────────────────────────────────────────
conversation_history: dict[int, list] = {}

# ─── SYSTEM PROMPT TRADING SOLANA ─────────────────────────────────────────────
SYSTEM_PROMPT = """Tu es un assistant IA spécialisé dans le trading de meme coins sur Solana.

Tu as accès à Solscan API (clé : fournie) et DexScreener.
Tu maîtrises : bundles Jito, MEV, analyse technique (Fibonacci, EMA, RSI, Bollinger Bands), 
lifecycle des meme coins, détection de rug pulls, lecture de Bubblemaps, et tout l'écosystème Solana 2025-2026.

Ton utilisateur est un trader expérimenté. Sois direct, précis, et donne des analyses actionnables.

Quand on te donne une adresse de token Solana, tu analyses :
- La distribution des holders (risque de concentration)
- Le volume / market cap ratio
- Les signaux narratifs et sociaux
- Les niveaux techniques (Fibonacci, support/résistance)
- Le verdict et les points d'entrée

Réponds en français, de façon concise et professionnelle. 
Utilise des emojis avec parcimonie pour la clarté."""

# ─── VÉRIFICATION UTILISATEUR AUTORISÉ ────────────────────────────────────────
def is_authorized(update: Update) -> bool:
    if not AUTHORIZED_USER:
        return True  # Si pas de restriction, tout le monde peut accéder
    username = update.effective_user.username or ""
    return username.lower() == AUTHORIZED_USER.lower().lstrip("@")

# ─── COMMANDE /start ───────────────────────────────────────────────────────────
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update):
        await update.message.reply_text("❌ Accès non autorisé.")
        return

    user_id = update.effective_user.id
    conversation_history[user_id] = []

    await update.message.reply_text(
        "🚀 *Solana Trading Bot — Connecté*\n\n"
        "Je suis ton assistant IA spécialisé Solana + Meme Coins.\n\n"
        "*Ce que tu peux me demander :*\n"
        "• `Analyse ce coin : [adresse]`\n"
        "• `Check ce wallet : [adresse]`\n"
        "• `Donne moi 3 coins early stage trending`\n"
        "• `Explique-moi les bundles Jito`\n"
        "• `Fibonacci sur PUNCH`\n\n"
        "*Commandes :*\n"
        "/start — Redémarrer\n"
        "/reset — Effacer l'historique\n"
        "/help — Aide\n\n"
        "Envoie ton premier message 👇",
        parse_mode=constants.ParseMode.MARKDOWN
    )

# ─── COMMANDE /reset ───────────────────────────────────────────────────────────
async def reset(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update):
        return
    user_id = update.effective_user.id
    conversation_history[user_id] = []
    await update.message.reply_text("🔄 Conversation réinitialisée.")

# ─── COMMANDE /help ────────────────────────────────────────────────────────────
async def help_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update):
        return
    await update.message.reply_text(
        "📖 *Guide d'utilisation*\n\n"
        "*Analyse de token :*\n"
        "`Analyse ce token : [adresse Solana]`\n\n"
        "*Analyse de wallet :*\n"
        "`Check ce wallet : [adresse Solana]`\n\n"
        "*Alpha et watchlist :*\n"
        "`Donne moi des coins sous 50M MC avec narrative forte`\n\n"
        "*Technique :*\n"
        "`Fibonacci sur [coin]` / `EMA sur [coin]`\n\n"
        "*Éducation Solana :*\n"
        "`Explique les bundles` / `C'est quoi un rug pull`\n\n"
        "/reset — Effacer l'historique de conversation",
        parse_mode=constants.ParseMode.MARKDOWN
    )

# ─── GESTION DES MESSAGES ─────────────────────────────────────────────────────
async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not is_authorized(update):
        await update.message.reply_text("❌ Accès non autorisé.")
        return

    user_id = update.effective_user.id
    user_message = update.message.text

    # Initialiser l'historique si besoin
    if user_id not in conversation_history:
        conversation_history[user_id] = []

    # Indicateur "en train d'écrire..."
    await context.bot.send_chat_action(
        chat_id=update.effective_chat.id,
        action=constants.ChatAction.TYPING
    )

    # Ajouter le message à l'historique
    conversation_history[user_id].append({
        "role": "user",
        "content": user_message
    })

    # Garder seulement les 20 derniers messages (évite de dépasser le context window)
    if len(conversation_history[user_id]) > 20:
        conversation_history[user_id] = conversation_history[user_id][-20:]

    try:
        # Appel à Claude
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1500,
            system=SYSTEM_PROMPT,
            messages=conversation_history[user_id]
        )

        assistant_message = response.content[0].text

        # Ajouter la réponse à l'historique
        conversation_history[user_id].append({
            "role": "assistant",
            "content": assistant_message
        })

        # Envoyer la réponse
        # Telegram limite à 4096 caractères par message
        if len(assistant_message) > 4096:
            for i in range(0, len(assistant_message), 4096):
                await update.message.reply_text(
                    assistant_message[i:i+4096],
                    parse_mode=constants.ParseMode.MARKDOWN
                )
        else:
            try:
                await update.message.reply_text(
                    assistant_message,
                    parse_mode=constants.ParseMode.MARKDOWN
                )
            except Exception:
                # Si le markdown échoue, envoyer en texte brut
                await update.message.reply_text(assistant_message)

    except anthropic.APIError as e:
        logger.error(f"Erreur API Anthropic : {e}")
        await update.message.reply_text(
            f"❌ Erreur API Claude : {str(e)}\nRéessaie dans quelques secondes."
        )
    except Exception as e:
        logger.error(f"Erreur inattendue : {e}")
        await update.message.reply_text("❌ Une erreur est survenue. Réessaie.")

# ─── MAIN ──────────────────────────────────────────────────────────────────────
def main():
    logger.info("Démarrage du bot Telegram Solana Trading...")

    app = ApplicationBuilder().token(TELEGRAM_TOKEN).build()

    # Handlers
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("reset", reset))
    app.add_handler(CommandHandler("help", help_command))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    logger.info("Bot démarré ! En attente de messages...")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
