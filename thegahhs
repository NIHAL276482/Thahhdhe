from telegram import Update
from telegram.ext import Application, CommandHandler, ContextTypes
from faker import Faker
import logging
import traceback
import pycountry
from difflib import get_close_matches
import random
import json
import os
import phonenumbers

# Configuration
OWNER_ID = 7187126565  # Replace with your Telegram user ID
DATA_FILE = "bot_data.json"
LOG_FILE = "bot.log"

# Initialize data storage
if not os.path.exists(DATA_FILE):
    with open(DATA_FILE, 'w') as f:
        json.dump({"users": [], "blocked": []}, f)

# Load initial data
with open(DATA_FILE) as f:
    bot_data = json.load(f)

# Configure logging
logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO,
    filename=LOG_FILE
)
logger = logging.getLogger(__name__)

# Load and enhance country data
COUNTRIES = {}
for country in pycountry.countries:
    names = {
        country.name.lower(),
        country.alpha_2.lower(),
        getattr(country, 'common_name', '').lower(),
        getattr(country, 'official_name', '').lower()
    }
    names = {n for n in names if n}
    # Fetch calling code using phonenumbers
    try:
        calling_code = phonenumbers.country_code_for_region(country.alpha_2)
    except Exception:
        calling_code = "N/A"
    COUNTRIES[country.alpha_2] = {
        'names': names,
        'official_name': getattr(country, 'official_name', country.name),
        'code': country.alpha_2,
        'calling_code': calling_code
    }

# Localization configuration
LOCALE_FALLBACKS = {
    'gh': 'en_GH', 'ca': 'en_CA', 'au': 'en_AU', 'nz': 'en_NZ',
    'za': 'en_ZA', 'ng': 'en_NG', 'ke': 'sw_KE', 'jp': 'ja_JP',
    'in': 'en_IN', 'eg': 'ar_EG', 'mx': 'es_MX', 'br': 'pt_BR',
    'de': 'de_DE', 'fr': 'fr_FR', 'es': 'es_ES', 'ru': 'ru_RU',
    'cn': 'zh_CN', 'kr': 'ko_KR', 'tr': 'tr_TR', 'it': 'it_IT',
    'gb': 'en_GB', 'us': 'en_US', 'uk': 'en_GB'  # Added UK and US
}

# Custom geographical data
CITY_OVERRIDES = {
    'GH': ["Accra", "Kumasi", "Tamale", "Sekondi-Takoradi", "Sunyani"],
    'CA': ["Toronto", "Montreal", "Vancouver", "Calgary", "Ottawa"],
    # Add more countries as needed
}

STREET_FORMATS = {
    'GH': lambda fake: f"{fake.random_int(1, 300)} {random.choice(['Ring Rd', 'Spintex Rd', 'Legon Rd', 'Osu St'])}",
    'CA': lambda fake: f"{fake.random_int(1, 5000)} {fake.street_name()} {random.choice(['Ave', 'St', 'Rd', 'Blvd'])}",
    'default': lambda fake: fake.street_address()
}

POSTAL_CODE_FORMATS = {
    'CA': lambda fake: (
        f"{random.choice('ABCEGHJKLMNPRSTVXY')}{random.randint(0,9)}"
        f"{random.choice('ABCEGHJKLMNPRSTVWXYZ')} {random.randint(0,9)}"
        f"{random.choice('ABCEGHJKLMNPRSTVWXYZ')}{random.randint(0,9)}"
    ),
    'GH': lambda fake: f"GH-{random.randint(100, 999)}-{random.randint(1000, 9999)}",
    'default': lambda fake: fake.postcode() if hasattr(fake, 'postcode') else "N/A"
}

# Custom regions for all countries
CUSTOM_REGIONS = {
    'GH': ["Greater Accra", "Ashanti", "Northern", "Western", "Eastern"],
    'CA': ["Ontario", "Quebec", "British Columbia", "Alberta", "Manitoba"],
    # Add more countries as needed
}

# Admin decorator
def restricted(func):
    async def wrapper(update: Update, context: ContextTypes.DEFAULT_TYPE):
        if update.effective_user.id == OWNER_ID:
            return await func(update, context)
        else:
            await update.message.reply_text("🚫 Access denied.")
    return wrapper

# Admin commands
@restricted
async def admin_panel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    text = (
        "🛠️ *Owner Control Panel*\n\n"
        "/stats - Bot statistics\n"
        "/broadcast <message> - Broadcast to all users\n"
        "/block <user_id> - Block user\n"
        "/unblock <user_id> - Unblock user\n"
        "/logs - Get latest logs"
    )
    await update.message.reply_text(text, parse_mode="Markdown")

@restricted
async def stats(update: Update, context: ContextTypes.DEFAULT_TYPE):
    stats_text = (
        f"📊 *Bot Statistics*\n"
        f"• Total users: {len(bot_data['users'])}\n"
        f"• Blocked users: {len(bot_data['blocked'])}\n"
        f"• Last error: {bot_data.get('last_error', 'None')}"
    )
    await update.message.reply_text(stats_text, parse_mode="Markdown")

@restricted
async def broadcast(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Please provide a message to broadcast")
        return
    
    message = ' '.join(context.args)
    success = 0
    failures = 0
    
    for user_id in bot_data['users']:
        try:
            await context.bot.send_message(chat_id=user_id, text=message)
            success += 1
        except Exception as e:
            failures += 1
            logger.error(f"Broadcast failed to {user_id}: {str(e)}")
    
    await update.message.reply_text(
        f"📢 Broadcast results:\n"
        f"• Success: {success}\n"
        f"• Failures: {failures}"
    )

@restricted
async def block_user(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Please provide a user ID")
        return
    
    user_id = context.args[0]
    bot_data['blocked'].append(user_id)
    save_data()
    
    await update.message.reply_text(f"User {user_id} blocked ✅")

@restricted
async def unblock_user(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not context.args:
        await update.message.reply_text("Please provide a user ID")
        return
    
    user_id = context.args[0]
    if user_id in bot_data['blocked']:
        bot_data['blocked'].remove(user_id)
        save_data()
        await update.message.reply_text(f"User {user_id} unblocked ✅")
    else:
        await update.message.reply_text("User not found in blocked list")

@restricted
async def get_logs(update: Update, context: ContextTypes.DEFAULT_TYPE):
    try:
        with open(LOG_FILE, 'rb') as f:
            await context.bot.send_document(
                chat_id=OWNER_ID,
                document=f,
                filename="bot_logs.log"
            )
    except Exception as e:
        await update.message.reply_text(f"Error getting logs: {str(e)}")

def save_data():
    with open(DATA_FILE, 'w') as f:
        json.dump(bot_data, f)

# Modified start command
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    if user_id not in bot_data['users']:
        bot_data['users'].append(user_id)
        save_data()
    
    text = (
        "🌍 Universal Address Generator Bot\n\n"
        "Usage: /address <country name/code>\n"
        "Examples:\n"
        "/address Ghana\n"
        "/address CA\n\n"
    )
    
    if int(user_id) == OWNER_ID:
        text += "👑 [Owner Panel](/admin)"
    
    await update.message.reply_text(text, parse_mode="Markdown")

# Modified address command with blocking check
async def address(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.effective_user.id)
    
    if user_id in bot_data['blocked']:
        await update.message.reply_text("🚫 Your access has been restricted")
        return
    
    try:
        if not context.args:
            raise ValueError("Please provide a country name/code")
        
        query = ' '.join(context.args)
        country = find_country(query)
        
        if not country:
            raise ValueError(f"Country '{query}' not found. Try official names")
        
        country_code = country['code'].upper()
        fake = get_localized_faker(country_code)

        # Generate address components
        address_data = {
            'street': STREET_FORMATS.get(country_code, STREET_FORMATS['default'])(fake),
            'city': random.choice(CITY_OVERRIDES.get(country_code, [fake.city()])),
            'postcode': POSTAL_CODE_FORMATS.get(country_code, POSTAL_CODE_FORMATS['default'])(fake),
            'region': safe_region(fake, country_code),
            'country': country['official_name'],
            'calling_code': country['calling_code'],
            'name': fake.name(),
            'gender': fake.prefix(),
            'phone_number': fake.phone_number()
        }

        # Log generated address for debugging
        logger.info(f"Generated address for {country_code}: {address_data}")

        response = (
            f"📍 **{address_data['country']}** ({country_code})\n\n"
            f"• 👤 Name: {address_data['name']} ({address_data['gender']})\n"
            f"• 📞 Phone: +{address_data['calling_code']} {address_data['phone_number']}\n"
            f"• 🏠 Street: {address_data['street']}\n"
            f"• � City: {address_data['city']}\n"
            f"• 📮 Postal Code: {address_data['postcode']}\n"
            f"• 🏛️ Region: {address_data['region']}\n\n"
            "Bot created by @ilovethatbreaths"
        )

        await update.message.reply_text(response, parse_mode="Markdown")

    except Exception as e:
        logger.error(f"Error: {traceback.format_exc()}")
        bot_data['last_error'] = str(e)
        save_data()
        await update.message.reply_text(f"❌ Error: {str(e)}")

def find_country(query: str):
    query = query.lower().strip()
    for code, data in COUNTRIES.items():
        if query == code.lower() or query in data['names']:
            return data
    all_names = [name for data in COUNTRIES.values() for name in data['names']]
    matches = get_close_matches(query, all_names, n=1, cutoff=0.6)
    return next((data for data in COUNTRIES.values() if matches and matches[0] in data['names']), None)

def get_localized_faker(country_code: str):
    locale = LOCALE_FALLBACKS.get(country_code.lower(), 'en_US')
    try:
        return Faker(locale)
    except Exception:
        logger.warning(f"Locale '{locale}' not found, falling back to 'en_US'")
        return Faker('en_US')

def safe_region(fake, country_code):
    try:
        if country_code in CUSTOM_REGIONS:
            return random.choice(CUSTOM_REGIONS[country_code])
        return fake.administrative_unit()
    except Exception:
        try:
            return fake.state()
        except Exception:
            return fake.current_country()

def main():
    application = Application.builder().token("7200695595:AAHcMHHJHgw0jgULpD-FlMnFo_RdMaC1N9Y").build()
    
    # Admin commands
    application.add_handler(CommandHandler("admin", admin_panel))
    application.add_handler(CommandHandler("stats", stats))
    application.add_handler(CommandHandler("broadcast", broadcast))
    application.add_handler(CommandHandler("block", block_user))
    application.add_handler(CommandHandler("unblock", unblock_user))
    application.add_handler(CommandHandler("logs", get_logs))
    
    # User commands
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("address", address))
    
    application.run_polling()

if __name__ == "__main__":
    main()
