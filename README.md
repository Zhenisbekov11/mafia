import random
from telegram import Update
from telegram.ext import (
    ApplicationBuilder, CommandHandler, ContextTypes
)

TOKEN = "8099725275:AAGOPOk7GEKfQYMNkyX7zYA4xcpHlaAScdQ"

players = {}        # user_id: user_obj
roles = {}          # user_id: role
alive = set()       # тірі ойыншылар
votes = {}          # user_id: voted_user_id
game_started = False

night_actions = {
    'mafia': None,
    'doctor': None,
    'detective': None
}

ROLE_LIST = ['mafia', 'doctor', 'detective']

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Сәлем! Мафия ойынына қош келдіңіз. /join арқылы қосылыңыз.")

async def join(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global game_started
    if game_started:
        await update.message.reply_text("Ойын басталып кетті.")
        return
    user = update.effective_user
    if user.id not in players:
        players[user.id] = user
        await update.message.reply_text(f"{user.first_name} ойынға қосылды!")
    else:
        await update.message.reply_text("Сіз бұған дейін қосылғансыз.")

async def startgame(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global game_started, alive
    if len(players) < 4:
        await update.message.reply_text("Кемінде 4 ойыншы керек.")
        return

    game_started = True
    user_ids = list(players.keys())
    random.shuffle(user_ids)

    for i, role in enumerate(ROLE_LIST):
        roles[user_ids[i]] = role
    for uid in user_ids[len(ROLE_LIST):]:
        roles[uid] = 'villager'

    alive = set(players.keys())
    for uid, role in roles.items():
        try:
            await context.bot.send_message(uid, f"Сіздің рөліңіз: {role.upper()}")
        except:
            await update.message.reply_text(f"{players[uid].first_name} жеке хабар алуға рұқсат бермеген.")

    await update.message.reply_text("Ойын басталды. Түн уақыты кірді.")
    await night_phase(context)

async def night_phase(context: ContextTypes.DEFAULT_TYPE):
    for uid in alive:
        role = roles[uid]
        if role == 'mafia':
            await context.bot.send_message(uid, "Кімді өлтіргіңіз келеді? /kill @username")
        elif role == 'doctor':
            await context.bot.send_message(uid, "Кімді емдегіңіз келеді? /save @username")
        elif role == 'detective':
            await context.bot.send_message(uid, "Кімді тексергіңіз келеді? /check @username")

async def handle_kill(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if roles.get(update.effective_user.id) != 'mafia':
        return
    username = update.message.text.split()[-1].replace('@', '')
    target = find_user_by_username(username)
    if target and target.id in alive:
        night_actions['mafia'] = target.id
        await update.message.reply_text(f"{username} өлтіруге таңдалды.")

async def handle_save(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if roles.get(update.effective_user.id) != 'doctor':
        return
    username = update.message.text.split()[-1].replace('@', '')
    target = find_user_by_username(username)
    if target and target.id in alive:
        night_actions['doctor'] = target.id
        await update.message.reply_text(f"{username} емдеуге таңдалды.")

async def handle_check(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if roles.get(update.effective_user.id) != 'detective':
        return
    username = update.message.text.split()[-1].replace('@', '')
    target = find_user_by_username(username)
    if target:
        role = roles[target.id]
        result = "MAFIA" if role == 'mafia' else "ЖОҚ"
        await update.message.reply_text(f"{username}: {result}")

async def end_night(update: Update, context: ContextTypes.DEFAULT_TYPE):
    killed = night_actions['mafia']
    saved = night_actions['doctor']
    night_actions.update({'mafia': None, 'doctor': None, 'detective': None})
if killed == saved:
        await update.message.reply_text("Түн тыныш өтті, ешкім өлген жоқ.")
    elif killed in alive:
        alive.remove(killed)
        await update.message.reply_text(f"{players[killed].first_name} түнде өлтірілді.")

    await update.message.reply_text("Күн шықты! /vote @username арқылы күдіктіңізге дауыс беріңіз.")

async def vote(update: Update, context: ContextTypes.DEFAULT_TYPE):
    voter = update.effective_user.id
    if voter not in alive:
        return
    username = update.message.text.split()[-1].replace('@', '')
    target = find_user_by_username(username)
    if target and target.id in alive:
        votes[voter] = target.id
        await update.message.reply_text(f"{username} үшін дауыс берілді.")

async def end_day(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not votes:
        await update.message.reply_text("Дауыс берілмеді.")
        return

    count = {}
    for vote_id in votes.values():
        count[vote_id] = count.get(vote_id, 0) + 1

    max_votes = max(count.values())
    suspects = [uid for uid, c in count.items() if c == max_votes]
    eliminated = random.choice(suspects)
    alive.remove(eliminated)
    await update.message.reply_text(f"{players[eliminated].first_name} көпшілік дауысымен ойыннан шығарылды.")
    votes.clear()

    await check_winner(update)

    await update.message.reply_text("Түн қайта басталды.")
    await night_phase(context)

async def check_winner(update: Update):
    mafia_alive = [uid for uid in alive if roles[uid] == 'mafia']
    others_alive = [uid for uid in alive if roles[uid] != 'mafia']

    if not mafia_alive:
        await update.message.reply_text("Тұрғындар жеңді!")
        reset_game()
    elif len(mafia_alive) >= len(others_alive):
        await update.message.reply_text("Мафия жеңді!")
        reset_game()

def find_user_by_username(username):
    for u in players.values():
        if u.username and u.username.lower() == username.lower():
            return u
    return None

def reset_game():
    global players, roles, alive, votes, night_actions, game_started
    players = {}
    roles = {}
    alive = set()
    votes = {}
    night_actions = {'mafia': None, 'doctor': None, 'detective': None}
    game_started = False

if name == "main":
    app = ApplicationBuilder().token(TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("join", join))
    app.add_handler(CommandHandler("startgame", startgame))
    app.add_handler(CommandHandler("kill", handle_kill))
    app.add_handler(CommandHandler("save", handle_save))
    app.add_handler(CommandHandler("check", handle_check))
    app.add_handler(CommandHandler("endnight", end_night))
    app.add_handler(CommandHandler("vote", vote))
    app.add_handler(CommandHandler("endday", end_day))

    app.run_polling()
    
