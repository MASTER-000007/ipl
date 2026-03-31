# =============================
# ULTIMATE IPL PREDICTION APP (AUTO-LOCK + MOBILE)
# =============================
from flask import Flask, request, render_template_string, redirect, session, send_file, flash
import csv
import os
import tempfile
import shutil
from datetime import datetime, timedelta

app = Flask(__name__)
app.secret_key = "ipl_ultimate_secret_key_2024"

# ================= FILE CONFIG =================
DIR_DATA = "data"
FILE_USERS = os.path.join(DIR_DATA, "users.csv")
FILE_MATCHES = os.path.join(DIR_DATA, "matches.csv")
FILE_PREDS = os.path.join(DIR_DATA, "predictions.csv")
FILE_RESULTS = os.path.join(DIR_DATA, "results.csv")
FILE_LOGS = os.path.join(DIR_DATA, "activity_logs.csv")
FILE_ANNOUNCEMENTS = os.path.join(DIR_DATA, "announcements.csv")
FILE_POINT_ADJUSTMENTS = os.path.join(DIR_DATA, "point_adjustments.csv")

if not os.path.exists(DIR_DATA):
    os.makedirs(DIR_DATA)

# ================= HELPERS =================
def init_files():
    if not os.path.exists(FILE_USERS):
        with open(FILE_USERS, "w", newline="") as f:
            csv.writer(f).writerow(["username", "password", "role", "status", "created_at"])
            csv.writer(f).writerow(["admin", "admin123", "admin", "active", datetime.now().isoformat()])
            users = ["hari", "arun", "vijay", "rahul", "kiran", "ajay", "rohit"]
            for u in users:
                csv.writer(f).writerow([u, "pass123", "user", "active", datetime.now().isoformat()])

    if not os.path.exists(FILE_MATCHES):
        with open(FILE_MATCHES, "w", newline="") as f:
            csv.writer(f).writerow(["match_id", "team1", "team2", "date", "time", "type", "winner", "status", "created_at", "lock_time"])

    if not os.path.exists(FILE_PREDS):
        with open(FILE_PREDS, "w", newline="") as f:
            csv.writer(f).writerow(["user", "match_id", "prediction", "timestamp", "status"])

    if not os.path.exists(FILE_RESULTS):
        with open(FILE_RESULTS, "w", newline="") as f:
            csv.writer(f).writerow(["match_id", "winner", "updated_by", "updated_at"])

    if not os.path.exists(FILE_LOGS):
        with open(FILE_LOGS, "w", newline="") as f:
            csv.writer(f).writerow(["timestamp", "user", "action", "details"])

    if not os.path.exists(FILE_ANNOUNCEMENTS):
        with open(FILE_ANNOUNCEMENTS, "w", newline="") as f:
            csv.writer(f).writerow(["id", "title", "message", "created_by", "created_at", "active"])

    if not os.path.exists(FILE_POINT_ADJUSTMENTS):
        with open(FILE_POINT_ADJUSTMENTS, "w", newline="") as f:
            csv.writer(f).writerow(["user", "points", "reason", "admin", "timestamp"])

def read_csv(filename):
    if not os.path.exists(filename):
        return []
    with open(filename, "r", newline="") as f:
        return list(csv.DictReader(f))

def write_csv(filename, data, fieldnames):
    dir_name = os.path.dirname(filename)
    fd, temp_path = tempfile.mkstemp(dir=dir_name)
    try:
        with os.fdopen(fd, 'w', newline="") as f:
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(data)
        shutil.move(temp_path, filename)
    except Exception:
        if os.path.exists(temp_path):
            os.remove(temp_path)
        raise

def log_activity(user, action, details):
    logs = read_csv(FILE_LOGS)
    logs.append({
        "timestamp": datetime.now().isoformat(),
        "user": user,
        "action": action,
        "details": details
    })
    write_csv(FILE_LOGS, logs, ["timestamp", "user", "action", "details"])

def get_user(username):
    users = read_csv(FILE_USERS)
    for u in users:
        if u['username'] == username:
            return u
    return None

def update_user_status(username, status):
    users = read_csv(FILE_USERS)
    for u in users:
        if u['username'] == username:
            u['status'] = status
            break
    write_csv(FILE_USERS, users, ["username", "password", "role", "status", "created_at"])

def check_auto_lock():
    """Automatically lock matches that have passed their lock time"""
    matches = read_csv(FILE_MATCHES)
    updated = False
    now = datetime.now()
    
    for m in matches:
        if m.get('status') == 'UPCOMING':
            try:
                match_datetime = datetime.strptime(f"{m['date']} {m.get('time', '19:30')}", "%Y-%m-%d %H:%M")
                # Lock 30 minutes before match time
                lock_time = match_datetime - timedelta(minutes=30)
                
                if now >= lock_time:
                    m['status'] = 'LOCKED'
                    m['lock_time'] = lock_time.isoformat()
                    updated = True
                    log_activity("SYSTEM", "AUTO_LOCK", f"Auto-locked {m['match_id']}")
            except:
                pass
    
    if updated:
        write_csv(FILE_MATCHES, matches, ["match_id", "team1", "team2", "date", "time", "type", "winner", "status", "created_at", "lock_time"])
    
    return matches

def get_match_status(m):
    """Determine match status with auto-lock logic"""
    if m.get('status') == 'COMPLETED':
        return 'COMPLETED'
    
    try:
        match_datetime = datetime.strptime(f"{m['date']} {m.get('time', '19:30')}", "%Y-%m-%d %H:%M")
        lock_time = match_datetime - timedelta(minutes=30)
        now = datetime.now()
        
        if now >= lock_time:
            return 'LOCKED'
        elif now >= match_datetime:
            return 'COMPLETED'
        else:
            return 'UPCOMING'
    except:
        return m.get('status', 'UPCOMING')

def calculate_leaderboard():
    matches = check_auto_lock()  # Check auto-lock before calculating
    predictions = read_csv(FILE_PREDS)
    adjustments = read_csv(FILE_POINT_ADJUSTMENTS)
    match_map = {m['match_id']: m for m in matches}
    user_preds = {}
    
    for p in predictions:
        if p['user'] not in user_preds:
            user_preds[p['user']] = []
        user_preds[p['user']].append(p)
    
    scores = {}
    user_stats = {}
    
    for user, preds in user_preds.items():
        score = 0
        streak = 0
        correct = 0
        total = 0
        preds_with_date = []
        
        for p in preds:
            mid = p['match_id']
            if mid in match_map:
                m = match_map[mid]
                if m.get('status') == 'COMPLETED' and m.get('winner'):
                    preds_with_date.append({
                        'pred': p['prediction'],
                        'winner': m['winner'],
                        'type': m.get('type', 'LEAGUE'),
                        'date': m.get('date', '2024-01-01')
                    })
        
        preds_with_date.sort(key=lambda x: x['date'])
        
        for item in preds_with_date:
            total += 1
            if item['pred'] == item['winner']:
                correct += 1
                points = 3 if item['type'] == 'PLAYOFF' else 1
                score += points
                streak += 1
                if streak % 3 == 0:
                    score += 1
            else:
                streak = 0
        
        for adj in adjustments:
            if adj['user'] == user:
                try:
                    score += int(adj['points'])
                except:
                    pass
        
        scores[user] = score
        user_stats[user] = {
            'correct': correct,
            'total': total,
            'accuracy': round((correct/total*100), 2) if total > 0 else 0
        }

    sorted_scores = sorted(scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_scores, user_stats

def get_active_announcements():
    announcements = read_csv(FILE_ANNOUNCEMENTS)
    return [a for a in announcements if a.get('active') == 'True']

def get_time_until_lock(m):
    """Calculate time remaining until match locks"""
    try:
        match_datetime = datetime.strptime(f"{m['date']} {m.get('time', '19:30')}", "%Y-%m-%d %H:%M")
        lock_time = match_datetime - timedelta(minutes=30)
        now = datetime.now()
        
        if now >= lock_time:
            return "Locked"
        
        remaining = lock_time - now
        hours = int(remaining.total_seconds() // 3600)
        minutes = int((remaining.total_seconds() % 3600) // 60)
        
        if hours > 0:
            return f"{hours}h {minutes}m"
        else:
            return f"{minutes}m"
    except:
        return "Unknown"

# ================= TEMPLATES (MOBILE RESPONSIVE) =================
STYLE = """
<style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, sans-serif; background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%); color: #e0e0e0; margin: 0; padding: 10px; min-height: 100vh; }
    .container { max-width: 100%; margin: 0 auto; background: rgba(22, 33, 62, 0.95); padding: 15px; border-radius: 12px; box-shadow: 0 8px 32px rgba(0,0,0,0.3); }
    h1 { color: #e94560; text-align: center; font-size: 1.8em; margin-bottom: 10px; text-shadow: 2px 2px 4px rgba(0,0,0,0.5); }
    h2 { color: #e94560; font-size: 1.4em; margin: 15px 0 10px 0; }
    h3 { color: #4cc9f0; font-size: 1.1em; margin: 10px 0; }
    input, select, textarea, button { width: 100%; padding: 12px; margin: 6px 0; border-radius: 8px; border: 1px solid #0f3460; font-size: 14px; -webkit-appearance: none; }
    input, select, textarea { background: #0f3460; color: white; }
    input:focus, select:focus, textarea:focus { outline: 2px solid #e94560; }
    button { background: linear-gradient(135deg, #e94560, #c0354e); color: white; font-weight: bold; cursor: pointer; transition: all 0.3s; border: none; }
    button:hover { transform: translateY(-2px); box-shadow: 0 4px 15px rgba(233, 69, 96, 0.4); }
    button:active { transform: translateY(0); }
    button.danger { background: linear-gradient(135deg, #dc3545, #c82333); }
    button.success { background: linear-gradient(135deg, #28a745, #218838); }
    button.warning { background: linear-gradient(135deg, #ffc107, #e0a800); color: #000; }
    button.secondary { background: linear-gradient(135deg, #6c757d, #5a6268); }
    button.small { padding: 8px 12px; width: auto; font-size: 12px; }
    table { width: 100%; border-collapse: collapse; margin-top: 15px; background: #0f3460; border-radius: 8px; overflow: hidden; font-size: 13px; }
    th, td { padding: 10px 8px; text-align: left; border-bottom: 1px solid #1a1a2e; }
    th { background: linear-gradient(135deg, #e94560, #c0354e); color: white; }
    tr:hover { background: #16213e; }
    .badge { padding: 4px 10px; border-radius: 15px; font-size: 0.7em; font-weight: bold; display: inline-block; }
    .badge-LEAGUE { background: #4cc9f0; color: #000; }
    .badge-PLAYOFF { background: #f72585; color: #fff; }
    .badge-active { background: #28a745; color: #fff; }
    .badge-inactive { background: #dc3545; color: #fff; }
    .badge-banned { background: #6c757d; color: #fff; }
    .badge-warning { background: #ffc107; color: #000; }
    .nav { display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; flex-wrap: wrap; gap: 8px; }
    .nav a { color: #4cc9f0; text-decoration: none; padding: 8px 12px; background: #0f3460; border-radius: 6px; transition: all 0.3s; font-size: 13px; white-space: nowrap; }
    .nav a:hover { background: #e94560; color: white; }
    .flash { background: linear-gradient(135deg, #ff9f43, #f39c12); color: #000; padding: 12px; border-radius: 8px; margin-bottom: 10px; text-align: center; font-weight: bold; font-size: 13px; }
    .flash-error { background: linear-gradient(135deg, #dc3545, #c82333); color: white; }
    .flash-success { background: linear-gradient(135deg, #28a745, #218838); color: white; }
    .card { background: #0f3460; padding: 15px; margin: 10px 0; border-radius: 10px; border-left: 5px solid #e94560; }
    .card-playoff { border-left-color: #f72585; }
    .stats-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(100px, 1fr)); gap: 10px; margin: 15px 0; }
    .stat-box { background: linear-gradient(135deg, #0f3460, #16213e); padding: 15px 10px; border-radius: 10px; text-align: center; }
    .stat-box h3 { margin: 0; font-size: 1.5em; color: #4cc9f0; }
    .stat-box p { margin: 5px 0 0 0; color: #aaa; font-size: 11px; }
    .announcement { background: linear-gradient(135deg, #2d3436, #1e272e); padding: 15px; margin: 10px 0; border-radius: 10px; border-left: 5px solid #f39c12; }
    .locked { opacity: 0.5; pointer-events: none; filter: grayscale(1); }
    .tabs { display: flex; gap: 5px; margin-bottom: 15px; flex-wrap: wrap; overflow-x: auto; }
    .tab { padding: 8px 15px; background: #0f3460; border-radius: 8px; cursor: pointer; transition: all 0.3s; font-size: 13px; white-space: nowrap; }
    .tab.active { background: #e94560; }
    .form-row { display: flex; gap: 8px; }
    .form-row > * { flex: 1; }
    .countdown { background: #e94560; color: white; padding: 5px 10px; border-radius: 5px; font-size: 12px; font-weight: bold; }
    .match-info { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 5px; }
    .match-teams { font-size: 1.2em; font-weight: bold; margin: 10px 0; }
    .mobile-menu { display: none; }
    
    /* Mobile Optimizations */
    @media (max-width: 768px) {
        body { padding: 5px; }
        .container { padding: 10px; border-radius: 8px; }
        h1 { font-size: 1.5em; }
        h2 { font-size: 1.2em; }
        .nav { flex-direction: column; align-items: stretch; }
        .nav a { text-align: center; margin: 3px 0; }
        .form-row { flex-direction: column; }
        table { font-size: 11px; }
        th, td { padding: 8px 5px; }
        .stats-grid { grid-template-columns: repeat(3, 1fr); }
        .stat-box { padding: 10px 5px; }
        .stat-box h3 { font-size: 1.2em; }
        .stat-box p { font-size: 10px; }
        .tabs { overflow-x: scroll; -webkit-overflow-scrolling: touch; }
        button { padding: 14px; font-size: 15px; }
        .card { padding: 12px; }
        .match-teams { font-size: 1.1em; }
        .mobile-hamburger { display: block; }
    }
    
    @media (max-width: 480px) {
        .stats-grid { grid-template-columns: repeat(2, 1fr); }
        .nav a { padding: 10px; }
        h1 { font-size: 1.3em; }
        .badge { font-size: 0.65em; padding: 3px 8px; }
    }
    
    /* Touch-friendly */
    @media (hover: none) {
        button:hover { transform: none; }
        .nav a:hover { background: #0f3460; }
    }
    
    /* Scrollable tables for mobile */
    .table-container { overflow-x: auto; -webkit-overflow-scrolling: touch; }
    
    /* Loading animation */
    .loading { display: inline-block; width: 20px; height: 20px; border: 3px solid #f3f3f3; border-top: 3px solid #e94560; border-radius: 50%; animation: spin 1s linear infinite; }
    @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }
</style>
"""

HTML_LOGIN = """
<!DOCTYPE html>
<html>
<head>
    <title>IPL Predictor - Login</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container" style="max-width: 400px; margin: 50px auto;">
    <h1>🏆 IPL PREDICTOR</h1>
    <p style="text-align:center; color:#aaa; margin-bottom:20px; font-size:14px;">Predict Matches • Earn Points • Win Prizes</p>
    
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, message in messages %}
          <div class="flash flash-{{ category }}">{{ message }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}
    
    <form method="post">
        <input name="username" placeholder="👤 Username" required autocomplete="username" style="padding:15px;">
        <input name="password" type="password" placeholder="🔒 Password" required autocomplete="current-password" style="padding:15px;">
        <button type="submit" style="padding:15px; margin-top:15px;">🔐 LOGIN</button>
    </form>
    
    <div style="margin-top:20px; text-align:center; color:#aaa; font-size:12px; line-height:1.8;">
        <p><strong>Default Users:</strong> hari, arun, vijay, rahul, kiran, ajay, rohit</p>
        <p><strong>Password:</strong> pass123</p>
        <p><strong>Admin:</strong> admin / admin123</p>
    </div>
</div>
</body>
</html>
"""

HTML_HOME = """
<!DOCTYPE html>
<html>
<head>
    <title>IPL Predictor - Home</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container">
    <div class="nav">
        <div style="font-size:1.1em;">👤 {{ user }} {% if role == 'admin' %}<span class="badge badge-active">ADMIN</span>{% endif %}</div>
        <div style="display:flex; gap:5px; flex-wrap:wrap; justify-content:center;">
            <a href="/home">🏠</a>
            <a href="/leaderboard">🏆</a>
            <a href="/my-stats">📊</a>
            {% if role == 'admin' %}
                <a href="/admin">⚙️</a>
                <a href="/admin-users">👥</a>
            {% endif %}
            <a href="/logout">🚪</a>
        </div>
    </div>
    
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, message in messages %}
          <div class="flash flash-{{ category }}">{{ message }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}
    
    {% for ann in announcements %}
    <div class="announcement">
        <h3 style="margin:0 0 8px 0; color:#f39c12; font-size:1em;">📢 {{ ann.title }}</h3>
        <p style="margin:0; font-size:13px; line-height:1.5;">{{ ann.message }}</p>
        <small style="color:#aaa; font-size:11px;">Posted by {{ ann.created_by }} on {{ ann.created_at[:10] }}</small>
    </div>
    {% endfor %}
    
    <div class="stats-grid">
        <div class="stat-box">
            <h3>{{ total_matches }}</h3>
            <p>Upcoming</p>
        </div>
        <div class="stat-box">
            <h3>{{ my_predictions }}</h3>
            <p>My Bets</p>
        </div>
        <div class="stat-box">
            <h3>{{ my_streak }}</h3>
            <p>Streak 🔥</p>
        </div>
    </div>
    
    <h2>🏏 Upcoming Matches</h2>
    
    {% if not matches %}
        <div class="card" style="text-align:center;">
            <p style="color:#aaa;">No upcoming matches. Check back later!</p>
        </div>
    {% endif %}
    
    {% for m in matches %}
    <div class="card {% if m.type == 'PLAYOFF' %}card-playoff{% endif %}">
        <div class="match-info">
            <div>
                <span class="badge badge-{{ m.type }}">{{ m.type }}</span>
                <span style="margin-left:8px; color:#aaa; font-size:12px;">📅 {{ m.date }}</span>
            </div>
            <div>
                {% if m.status == 'UPCOMING' %}
                    <span class="countdown">⏰ Locks in: {{ m.time_until_lock }}</span>
                {% elif m.status == 'LOCKED' %}
                    <span class="badge badge-warning">🔒 LOCKED</span>
                {% endif %}
            </div>
        </div>
        <div class="match-teams">{{ m.team1 }} 🆚 {{ m.team2 }}</div>
        <div style="color:#aaa; font-size:12px; margin-bottom:10px;">🕐 {{ m.time }} | {{ m.match_id }}</div>
        
        {% if m.betted %}
            <button class="locked" disabled style="padding:12px;">🔒 Your Prediction: {{ m.my_prediction }}</button>
        {% elif m.status == 'LOCKED' %}
            <button class="locked" disabled style="padding:12px; background:#6c757d;">🔒 Betting Closed</button>
        {% else %}
            <form method="post" action="/place_bet">
                <input type="hidden" name="match_id" value="{{ m.match_id }}">
                <div class="form-row">
                    <select name="prediction" required style="padding:12px;">
                        <option value="" disabled selected>Select Winner</option>
                        <option value="{{ m.team1 }}">{{ m.team1 }}</option>
                        <option value="{{ m.team2 }}">{{ m.team2 }}</option>
                    </select>
                    <button type="submit" style="width:100px; padding:12px;">⚡</button>
                </div>
            </form>
        {% endif %}
    </div>
    {% endfor %}
</div>

<script>
// Auto-refresh every 60 seconds to check lock status
setTimeout(function(){ location.reload(); }, 60000);
</script>
</body>
</html>
"""

HTML_LEADERBOARD = """
<!DOCTYPE html>
<html>
<head>
    <title>Leaderboard</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container">
    <div class="nav">
        <h2 style="margin:0;">🏆 Leaderboard</h2>
        <div style="display:flex; gap:5px;">
            <a href="/home">🏠</a>
            <a href="/my-stats">📊</a>
            <a href="/logout">🚪</a>
        </div>
    </div>
    
    <div class="stats-grid">
        <div class="stat-box">
            <h3>{{ total_users }}</h3>
            <p>Players</p>
        </div>
        <div class="stat-box">
            <h3>{{ total_matches_completed }}</h3>
            <p>Completed</p>
        </div>
    </div>
    
    <div class="table-container">
        <table>
            <thead>
                <tr>
                    <th>#</th>
                    <th>Player</th>
                    <th>Pts</th>
                    <th>Acc</th>
                </tr>
            </thead>
            <tbody>
                {% for user, score in scores %}
                <tr style="{% if user == current_user %}background:#1a1a2e; border-left:3px solid #e94560;{% endif %}">
                    <td>
                        {% if loop.index == 1 %}🥇{% elif loop.index == 2 %}🥈{% elif loop.index == 3 %}🥉{% else %}{{ loop.index }}{% endif %}
                    </td>
                    <td>
                        {{ user }}
                        {% if user == current_user %}<span class="badge badge-active">YOU</span>{% endif %}
                    </td>
                    <td style="font-weight:bold; color:#e94560;">{{ score }}</td>
                    <td>{{ stats.get(user, {}).get('accuracy', 0) }}%</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
    
    <div class="card" style="margin-top:15px;">
        <h3 style="font-size:1em;">📋 Scoring Rules</h3>
        <ul style="color:#aaa; line-height:1.8; font-size:13px; padding-left:20px;">
            <li>✅ League Win: <strong>+1 Pt</strong></li>
            <li>✅ Playoff Win: <strong>+3 Pts</strong></li>
            <li>🔥 Streak: <strong>+1 Pt</strong> per 3 consecutive</li>
            <li>⚡ Auto-lock: 30 min before match</li>
        </ul>
    </div>
</div>
</body>
</html>
"""

HTML_MY_STATS = """
<!DOCTYPE html>
<html>
<head>
    <title>My Stats</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container">
    <div class="nav">
        <h2 style="margin:0;">📊 My Stats</h2>
        <div style="display:flex; gap:5px;">
            <a href="/home">🏠</a>
            <a href="/leaderboard">🏆</a>
            <a href="/logout">🚪</a>
        </div>
    </div>
    
    <div class="stats-grid">
        <div class="stat-box">
            <h3>{{ total_predictions }}</h3>
            <p>Total</p>
        </div>
        <div class="stat-box">
            <h3>{{ correct_predictions }}</h3>
            <p>Correct</p>
        </div>
        <div class="stat-box">
            <h3>{{ accuracy }}%</h3>
            <p>Accuracy</p>
        </div>
        <div class="stat-box">
            <h3>{{ current_streak }}</h3>
            <p>Streak</p>
        </div>
        <div class="stat-box">
            <h3>{{ total_points }}</h3>
            <p>Points</p>
        </div>
        <div class="stat-box">
            <h3>{{ rank }}</h3>
            <p>Rank</p>
        </div>
    </div>
    
    <h3>📜 History</h3>
    <div class="table-container">
        <table>
            <thead>
                <tr>
                    <th>Match</th>
                    <th>Pred</th>
                    <th>Result</th>
                    <th>Pts</th>
                </tr>
            </thead>
            <tbody>
                {% for p in predictions %}
                <tr>
                    <td style="font-size:11px;">{{ p.match_id }}<br>{{ p.match[:20] }}</td>
                    <td>{{ p.prediction }}</td>
                    <td>
                        {% if p.winner %}
                            {% if p.prediction == p.winner %}
                                <span class="badge badge-active">✅</span>
                            {% else %}
                                <span class="badge badge-inactive">❌</span>
                            {% endif %}
                        {% else %}
                            <span class="badge" style="background:#6c757d;">⏳</span>
                        {% endif %}
                    </td>
                    <td>{{ p.points or '-' }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>
</body>
</html>
"""

# Replace the HTML_ADMIN variable with this updated version:

HTML_ADMIN = """
<!DOCTYPE html>
<html>
<head>
    <title>Admin Panel</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container">
    <div class="nav">
        <h2 style="margin:0;">⚙️ Admin Panel</h2>
        <div style="display:flex; gap:5px;">
            <a href="/home">🏠</a>
            <a href="/admin-users">👥</a>
            <a href="/admin-logs">📋</a>
            <a href="/logout">🚪</a>
        </div>
    </div>
    
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, message in messages %}
          <div class="flash flash-{{ category }}">{{ message }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}
    
    <div class="tabs">
        <div class="tab active" onclick="showTab('matches')">Matches</div>
        <div class="tab" onclick="showTab('points')">Points</div>
        <div class="tab" onclick="showTab('announce')">Posts</div>
        <div class="tab" onclick="showTab('export')">Export</div>
    </div>
    
    <div id="matches-tab">
        <div class="card">
            <h3 style="font-size:1em;">➕ Create Match</h3>
            <form method="post">
                <input type="hidden" name="action" value="create_match">
                <div class="form-row">
                    <input name="match_id" placeholder="ID (M01)" required>
                    <input name="date" type="date" required>
                </div>
                <div class="form-row">
                    <input name="time" type="time" value="19:30" required>
                    <select name="type">
                        <option value="LEAGUE">League</option>
                        <option value="PLAYOFF">Playoff</option>
                    </select>
                </div>
                <div class="form-row">
                    <select name="team1" required>{% for t in teams %}<option>{{t}}</option>{% endfor %}</select>
                    <select name="team2" required>{% for t in teams %}<option>{{t}}</option>{% endfor %}</select>
                </div>
                <button type="submit" class="success" style="padding:14px;">Create Match</button>
            </form>
            <p style="color:#aaa; font-size:12px; margin-top:10px;">⚠️ Auto-locks 30 min before match time</p>
        </div>
        
        <h3 style="font-size:1.1em;">📅 All Matches</h3>
        <div class="table-container">
            <table>
                <thead>
                    <tr>
                        <th>ID</th>
                        <th>Match</th>
                        <th>Date/Time</th>
                        <th>Type</th>
                        <th>Status</th>
                        <th>Winner</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {% for m in matches %}
                    <tr>
                        <td><strong>{{ m.match_id }}</strong></td>
                        <td style="font-size:11px;">{{ m.team1 }}<br>vs {{ m.team2 }}</td>
                        <td style="font-size:11px;">{{ m.date }}<br>{{ m.time }}</td>
                        <td><span class="badge badge-{{ m.type }}">{{ m.type }}</span></td>
                        <td><span class="badge badge-{% if m.status == 'UPCOMING' %}active{% elif m.status == 'LOCKED' %}warning{% else %}inactive{% endif %}">{{ m.status }}</span></td>
                        <td>{{ m.winner or '-' }}</td>
                        <td>
                            <div style="display:flex; gap:5px; flex-wrap:wrap;">
                                {% if m.status == 'UPCOMING' %}
                                    <form method="post" style="display:inline;">
                                        <input type="hidden" name="action" value="lock_match">
                                        <input type="hidden" name="match_id" value="{{ m.match_id }}">
                                        <button type="submit" class="warning small" title="Lock Betting">🔒</button>
                                    </form>
                                {% endif %}
                                
                                {% if m.status != 'COMPLETED' %}
                                    <form method="post" style="display:inline;">
                                        <input type="hidden" name="action" value="update_result">
                                        <input type="hidden" name="match_id" value="{{ m.match_id }}">
                                        <select name="winner" style="padding:6px; width:60px; font-size:11px;">
                                            <option value="{{ m.team1 }}">{{ m.team1[:3] }}</option>
                                            <option value="{{ m.team2 }}">{{ m.team2[:3] }}</option>
                                        </select>
                                        <button type="submit" class="success small" title="Set Winner">✅</button>
                                    </form>
                                {% endif %}
                                
                                <!-- DELETE BUTTON - ALWAYS VISIBLE -->
                                <form method="post" style="display:inline;" onsubmit="return confirm('⚠️ Delete match {{ m.match_id }}?\\n\\nThis will also delete all predictions for this match.\\n\\nThis action CANNOT be undone!');">
                                    <input type="hidden" name="action" value="delete_match">
                                    <input type="hidden" name="match_id" value="{{ m.match_id }}">
                                    <button type="submit" class="danger small" title="Delete Match">🗑️</button>
                                </form>
                            </div>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
    
    <div id="points-tab" style="display:none;">
        <div class="card">
            <h3 style="font-size:1em;">💎 Adjust Points</h3>
            <form method="post">
                <input type="hidden" name="action" value="adjust_points">
                <select name="user" required>
                    <option value="">Select User</option>
                    {% for u in all_users %}
                    <option value="{{ u.username }}">{{ u.username }} ({{ u.role }})</option>
                    {% endfor %}
                </select>
                <input name="points" type="number" placeholder="Points (+/-)" required>
                <textarea name="reason" placeholder="Reason" required style="height:60px;"></textarea>
                <button type="submit" class="warning" style="padding:14px;">Adjust Points</button>
            </form>
        </div>
        
        <h3 style="font-size:1em;">📝 Adjustment History</h3>
        <div class="table-container">
            <table>
                <thead>
                    <tr><th>User</th><th>Pts</th><th>Reason</th><th>Admin</th><th>Date</th></tr>
                </thead>
                <tbody>
                    {% for adj in adjustments %}
                    <tr>
                        <td>{{ adj.user }}</td>
                        <td style="color:{% if adj.points|int > 0 %}#28a745{% else %}#dc3545{% endif %}; font-weight:bold;">
                            {% if adj.points|int > 0 %}+{% endif %}{{ adj.points }}
                        </td>
                        <td style="font-size:11px;">{{ adj.reason }}</td>
                        <td style="font-size:11px;">{{ adj.admin }}</td>
                        <td style="font-size:11px;">{{ adj.timestamp[:10] }}</td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
    
    <div id="announce-tab" style="display:none;">
        <div class="card">
            <h3 style="font-size:1em;">📢 New Announcement</h3>
            <form method="post">
                <input type="hidden" name="action" value="create_announcement">
                <input name="title" placeholder="Title" required>
                <textarea name="message" placeholder="Message" required style="height:80px;"></textarea>
                <button type="submit" class="success" style="padding:14px;">Post Announcement</button>
            </form>
        </div>
        
        <h3 style="font-size:1em;">Active Posts</h3>
        <div class="table-container">
            <table>
                <thead>
                    <tr><th>Title</th><th>Message</th><th>By</th><th>Date</th><th>Action</th></tr>
                </thead>
                <tbody>
                    {% for ann in announcements %}
                    <tr>
                        <td style="font-size:11px;"><strong>{{ ann.title }}</strong></td>
                        <td style="font-size:11px;">{{ ann.message[:40] }}...</td>
                        <td style="font-size:11px;">{{ ann.created_by }}</td>
                        <td style="font-size:11px;">{{ ann.created_at[:10] }}</td>
                        <td>
                            <form method="post" style="display:inline;">
                                <input type="hidden" name="action" value="toggle_announcement">
                                <input type="hidden" name="id" value="{{ ann.id }}">
                                <button type="submit" class="{% if ann.active == 'True' %}warning{% else %}success{% endif %} small">
                                    {% if ann.active == 'True' %}👁️ Hide{% else %}📢 Show{% endif %}
                                </button>
                            </form>
                        </td>
                    </tr>
                    {% endfor %}
                </tbody>
            </table>
        </div>
    </div>
    
    <div id="export-tab" style="display:none;">
        <div class="card">
            <h3 style="font-size:1em;">📥 Download Data</h3>
            <div class="form-row" style="flex-direction:column; gap:10px;">
                <a href="/download/predictions"><button class="success">📊 Predictions CSV</button></a>
                <a href="/download/matches"><button class="success">🏏 Matches CSV</button></a>
                <a href="/download/users"><button class="success">👥 Users CSV</button></a>
                <a href="/download/logs"><button class="warning">📋 Activity Logs CSV</button></a>
            </div>
        </div>
        
        <div class="card">
            <h3 style="font-size:1em;">⚠️ Danger Zone</h3>
            <p style="color:#aaa; font-size:12px; margin-bottom:10px;">These actions cannot be undone!</p>
            <div class="form-row" style="flex-direction:column; gap:10px;">
                <form method="post" onsubmit="return confirm('⚠️ Reset ALL predictions?\\n\\nThis will delete all user predictions.\\n\\nThis action CANNOT be undone!');">
                    <input type="hidden" name="action" value="reset_predictions">
                    <button type="submit" class="danger" style="padding:14px;">🗑️ Reset All Predictions</button>
                </form>
                <form method="post" onsubmit="return confirm('⚠️ Reset ALL score adjustments?\\n\\nThis will remove all manual point changes.\\n\\nThis action CANNOT be undone!');">
                    <input type="hidden" name="action" value="reset_scores">
                    <button type="submit" class="danger" style="padding:14px;">🔄 Reset Score Adjustments</button>
                </form>
            </div>
        </div>
    </div>
</div>

<script>
function showTab(tabName) {
    document.querySelectorAll('.tabs .tab').forEach(t => t.classList.remove('active'));
    document.querySelectorAll('[id$="-tab"]').forEach(t => t.style.display = 'none');
    event.target.classList.add('active');
    document.getElementById(tabName + '-tab').style.display = 'block';
}
</script>
</body>
</html>
"""

HTML_ADMIN_USERS = """
<!DOCTYPE html>
<html>
<head>
    <title>User Management</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container">
    <div class="nav">
        <h2 style="margin:0;">👥 Users</h2>
        <div style="display:flex; gap:5px;">
            <a href="/admin">⚙️</a>
            <a href="/home">🏠</a>
            <a href="/logout">🚪</a>
        </div>
    </div>
    
    {% with messages = get_flashed_messages(with_categories=true) %}
      {% if messages %}
        {% for category, message in messages %}
          <div class="flash flash-{{ category }}">{{ message }}</div>
        {% endfor %}
      {% endif %}
    {% endwith %}
    
    <div class="card">
        <h3 style="font-size:1em;">➕ Add User</h3>
        <form method="post">
            <input type="hidden" name="action" value="add_user">
            <div class="form-row">
                <input name="username" placeholder="Username" required>
                <input name="password" placeholder="Password" required>
            </div>
            <select name="role">
                <option value="user">User</option>
                <option value="admin">Admin</option>
            </select>
            <button type="submit" class="success" style="padding:14px;">Create</button>
        </form>
    </div>
    
    <h3 style="font-size:1.1em;">All Users</h3>
    <div class="table-container">
        <table>
            <thead>
                <tr><th>User</th><th>Role</th><th>Status</th><th>Action</th></tr>
            </thead>
            <tbody>
                {% for u in users %}
                <tr>
                    <td>{{ u.username }}</td>
                    <td><span class="badge badge-{% if u.role == 'admin' %}active{% else %}inactive{% endif %}">{{ u.role }}</span></td>
                    <td><span class="badge badge-{% if u.status == 'active' %}active{% elif u.status == 'banned' %}banned{% else %}inactive{% endif %}">{{ u.status }}</span></td>
                    <td>
                        {% if u.status == 'active' %}
                            <form method="post" style="display:inline;">
                                <input type="hidden" name="action" value="ban_user">
                                <input type="hidden" name="username" value="{{ u.username }}">
                                <button type="submit" class="danger small">Ban</button>
                            </form>
                        {% else %}
                            <form method="post" style="display:inline;">
                                <input type="hidden" name="action" value="unban_user">
                                <input type="hidden" name="username" value="{{ u.username }}">
                                <button type="submit" class="success small">Unban</button>
                            </form>
                        {% endif %}
                        {% if u.role != 'admin' %}
                            <form method="post" style="display:inline;" onsubmit="return confirm('Delete?');">
                                <input type="hidden" name="action" value="delete_user">
                                <input type="hidden" name="username" value="{{ u.username }}">
                                <button type="submit" class="danger small">Del</button>
                            </form>
                        {% endif %}
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>
</body>
</html>
"""

HTML_ADMIN_LOGS = """
<!DOCTYPE html>
<html>
<head>
    <title>Activity Logs</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
""" + STYLE + """</head>
<body>
<div class="container">
    <div class="nav">
        <h2 style="margin:0;">📋 Logs</h2>
        <div style="display:flex; gap:5px;">
            <a href="/admin">⚙️</a>
            <a href="/home">🏠</a>
            <a href="/logout">🚪</a>
        </div>
    </div>
    
    <div class="table-container">
        <table>
            <thead>
                <tr><th>Time</th><th>User</th><th>Action</th><th>Details</th></tr>
            </thead>
            <tbody>
                {% for log in logs %}
                <tr>
                    <td style="font-size:11px;">{{ log.timestamp[11:19] }}<br>{{ log.timestamp[:10] }}</td>
                    <td>{{ log.user }}</td>
                    <td>{{ log.action }}</td>
                    <td style="font-size:11px;">{{ log.details[:40] }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>
</div>
</body>
</html>
"""

TEAMS = ["CSK", "MI", "RCB", "KKR", "GT", "SRH", "RR", "DC", "PBKS", "LSG"]

# ================= ROUTES =================
@app.route("/", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        u = request.form.get("username")
        p = request.form.get("password")
        user = get_user(u)
        
        if user is None:
            flash("Invalid Credentials", "error")
            return render_template_string(HTML_LOGIN)
        
        if user.get('status') == 'banned':
            flash("Your account has been banned. Contact admin.", "error")
            return render_template_string(HTML_LOGIN)
        
        if user.get('password') == p:
            session['user'] = u
            session['role'] = user.get('role', 'user')
            log_activity(u, "LOGIN", "User logged in successfully")
            return redirect("/home")
        
        flash("Invalid Credentials", "error")
    return render_template_string(HTML_LOGIN)

@app.route("/home")
def home():
    if "user" not in session:
        return redirect("/")
    
    # Check auto-lock before showing matches
    matches = check_auto_lock()
    user = session['user']
    preds = read_csv(FILE_PREDS)
    my_bet_ids = [p['match_id'] for p in preds if p['user'] == user]
    today = str(datetime.now().date())
    upcoming = []
    
    for m in matches:
        # Update status based on auto-lock
        m['status'] = get_match_status(m)
        m['time_until_lock'] = get_time_until_lock(m)
        
        if m.get('status') in ['UPCOMING', 'LOCKED'] and m.get('date', '') >= today:
            m['betted'] = m['match_id'] in my_bet_ids
            if m['betted']:
                my_pred = next((p for p in preds if p['user'] == user and p['match_id'] == m['match_id']), None)
                m['my_prediction'] = my_pred['prediction'] if my_pred else '-'
            upcoming.append(m)
    
    streak = 0
    for m in matches:
        if m.get('status') == 'COMPLETED' and m.get('winner'):
            my_pred = next((p for p in preds if p['user'] == user and p['match_id'] == m['match_id']), None)
            if my_pred and my_pred['prediction'] == m['winner']:
                streak += 1
            else:
                streak = 0
    
    announcements = get_active_announcements()
    
    return render_template_string(HTML_HOME, 
        user=user, 
        role=session['role'], 
        matches=upcoming, 
        teams=TEAMS,
        total_matches=len([m for m in upcoming if m.get('status') == 'UPCOMING']),
        my_predictions=len(my_bet_ids),
        my_streak=streak,
        announcements=announcements)

@app.route("/place_bet", methods=["POST"])
def place_bet():
    if "user" not in session:
        return redirect("/")
    
    user = session['user']
    match_id = request.form.get("match_id")
    prediction = request.form.get("prediction")
    
    matches = check_auto_lock()
    match = next((m for m in matches if m['match_id'] == match_id), None)
    
    if not match:
        flash("Match not found", "error")
        return redirect("/home")
    
    # Check if match is auto-locked
    actual_status = get_match_status(match)
    if actual_status != 'UPCOMING':
        flash("Cannot bet - Match is locked or completed", "error")
        return redirect("/home")
        
    preds = read_csv(FILE_PREDS)
    for p in preds:
        if p['user'] == user and p['match_id'] == match_id:
            flash("Prediction already submitted", "error")
            return redirect("/home")
    
    new_pred = {
        "user": user,
        "match_id": match_id,
        "prediction": prediction,
        "timestamp": datetime.now().isoformat(),
        "status": "active"
    }
    
    fieldnames = ["user", "match_id", "prediction", "timestamp", "status"]
    data = preds + [new_pred]
    write_csv(FILE_PREDS, data, fieldnames)
    
    log_activity(user, "PLACE_BET", f"Predicted {prediction} for {match_id}")
    flash("Prediction Submitted! ✅", "success")
    return redirect("/home")

@app.route("/leaderboard")
def leaderboard():
    if "user" not in session:
        return redirect("/")
    
    scores, stats = calculate_leaderboard()
    users = read_csv(FILE_USERS)
    active_users = [u for u in users if u.get('status') == 'active' and u.get('role') == 'user']
    matches = read_csv(FILE_MATCHES)
    completed = [m for m in matches if m.get('status') == 'COMPLETED']
    
    return render_template_string(HTML_LEADERBOARD, 
        scores=scores, 
        stats=stats,
        current_user=session['user'],
        role=session['role'],
        total_users=len(active_users),
        total_matches_completed=len(completed))

@app.route("/my-stats")
def my_stats():
    if "user" not in session:
        return redirect("/")
    
    user = session['user']
    preds = read_csv(FILE_PREDS)
    matches = read_csv(FILE_MATCHES)
    match_map = {m['match_id']: m for m in matches}
    
    user_preds = [p for p in preds if p['user'] == user]
    total_predictions = len(user_preds)
    correct = 0
    current_streak = 0
    points = 0
    
    prediction_history = []
    
    for p in user_preds:
        m = match_map.get(p['match_id'], {})
        winner = m.get('winner', '')
        is_correct = p['prediction'] == winner if winner else None
        
        pts = 0
        if is_correct == True:
            correct += 1
            current_streak += 1
            pts = 3 if m.get('type') == 'PLAYOFF' else 1
            points += pts
            if current_streak % 3 == 0:
                points += 1
        elif is_correct == False:
            current_streak = 0
        
        prediction_history.append({
            'match_id': p['match_id'],
            'match': f"{m.get('team1', '-')} vs {m.get('team2', '-')}",
            'prediction': p['prediction'],
            'winner': winner,
            'points': pts if is_correct == True else 0 if winner else None
        })
    
    accuracy = round((correct/total_predictions*100), 2) if total_predictions > 0 else 0
    
    scores, _ = calculate_leaderboard()
    rank = next((i+1 for i, (u, _) in enumerate(scores) if u == user), len(scores)+1)
    
    return render_template_string(HTML_MY_STATS,
        total_predictions=total_predictions,
        correct_predictions=correct,
        accuracy=accuracy,
        current_streak=current_streak,
        total_points=points,
        rank=rank,
        predictions=prediction_history)

@app.route("/admin", methods=["GET", "POST"])
def admin():
    if "user" not in session or session.get("role") != "admin":
        flash("Access Denied", "error")
        return redirect("/")
    
    if request.method == "POST":
        action = request.form.get("action")
        admin_user = session['user']
        
        if action == "create_match":
            mid = request.form.get("match_id")
            t1 = request.form.get("team1")
            t2 = request.form.get("team2")
            date = request.form.get("date")
            time = request.form.get("time")
            mtype = request.form.get("type")
            
            matches = read_csv(FILE_MATCHES)
            if any(m['match_id'] == mid for m in matches):
                flash("Match ID already exists", "error")
            else:
                new_m = {
                    "match_id": mid, "team1": t1, "team2": t2,
                    "date": date, "time": time, "type": mtype,
                    "winner": "", "status": "UPCOMING",
                    "created_at": datetime.now().isoformat(),
                    "lock_time": ""
                }
                matches.append(new_m)
                write_csv(FILE_MATCHES, matches, ["match_id", "team1", "team2", "date", "time", "type", "winner", "status", "created_at", "lock_time"])
                log_activity(admin_user, "CREATE_MATCH", f"Created {mid}: {t1} vs {t2} at {time}")
                flash("Match Created! Auto-locks 30min before", "success")
                
        elif action == "update_result":
            mid = request.form.get("match_id")
            winner = request.form.get("winner")
            
            matches = read_csv(FILE_MATCHES)
            for m in matches:
                if m['match_id'] == mid:
                    m['winner'] = winner
                    m['status'] = 'COMPLETED'
                    break
            
            write_csv(FILE_MATCHES, matches, ["match_id", "team1", "team2", "date", "time", "type", "winner", "status", "created_at", "lock_time"])
            
            results = read_csv(FILE_RESULTS)
            results.append({
                "match_id": mid, "winner": winner,
                "updated_by": admin_user, "updated_at": datetime.now().isoformat()
            })
            write_csv(FILE_RESULTS, results, ["match_id", "winner", "updated_by", "updated_at"])
            
            log_activity(admin_user, "UPDATE_RESULT", f"Set winner {winner} for {mid}")
            flash("Result Updated! Leaderboard refreshed", "success")
            
        elif action == "lock_match":
            mid = request.form.get("match_id")
            matches = read_csv(FILE_MATCHES)
            for m in matches:
                if m['match_id'] == mid:
                    m['status'] = 'LOCKED'
                    break
            write_csv(FILE_MATCHES, matches, ["match_id", "team1", "team2", "date", "time", "type", "winner", "status", "created_at", "lock_time"])
            log_activity(admin_user, "LOCK_MATCH", f"Manually locked {mid}")
            flash("Match Locked", "success")
            
        elif action == "delete_match":
            mid = request.form.get("match_id")
            matches = read_csv(FILE_MATCHES)
            matches = [m for m in matches if m['match_id'] != mid]
            write_csv(FILE_MATCHES, matches, ["match_id", "team1", "team2", "date", "time", "type", "winner", "status", "created_at", "lock_time"])
            log_activity(admin_user, "DELETE_MATCH", f"Deleted {mid}")
            flash("Match Deleted", "success")
            
        elif action == "adjust_points":
            target_user = request.form.get("user")
            points = int(request.form.get("points"))
            reason = request.form.get("reason")
            
            adjustments = read_csv(FILE_POINT_ADJUSTMENTS)
            adjustments.append({
                "user": target_user, "points": points,
                "reason": reason, "admin": admin_user,
                "timestamp": datetime.now().isoformat()
            })
            write_csv(FILE_POINT_ADJUSTMENTS, adjustments, ["user", "points", "reason", "admin", "timestamp"])
            log_activity(admin_user, "ADJUST_POINTS", f"Adjusted {points} for {target_user}: {reason}")
            flash(f"Points adjusted for {target_user}", "success")
            
        elif action == "create_announcement":
            title = request.form.get("title")
            message = request.form.get("message")
            
            announcements = read_csv(FILE_ANNOUNCEMENTS)
            new_id = len(announcements) + 1
            announcements.append({
                "id": str(new_id), "title": title, "message": message,
                "created_by": admin_user, "created_at": datetime.now().isoformat(),
                "active": "True"
            })
            write_csv(FILE_ANNOUNCEMENTS, announcements, ["id", "title", "message", "created_by", "created_at", "active"])
            log_activity(admin_user, "CREATE_ANNOUNCEMENT", f"Posted: {title}")
            flash("Announcement Posted", "success")
            
        elif action == "toggle_announcement":
            aid = request.form.get("id")
            announcements = read_csv(FILE_ANNOUNCEMENTS)
            for a in announcements:
                if a['id'] == aid:
                    a['active'] = "False" if a['active'] == "True" else "True"
                    break
            write_csv(FILE_ANNOUNCEMENTS, announcements, ["id", "title", "message", "created_by", "created_at", "active"])
            flash("Announcement Updated", "success")
            
        elif action == "reset_predictions":
            write_csv(FILE_PREDS, [], ["user", "match_id", "prediction", "timestamp", "status"])
            log_activity(admin_user, "RESET", "Reset all predictions")
            flash("All predictions reset", "warning")
            
        elif action == "reset_scores":
            write_csv(FILE_POINT_ADJUSTMENTS, [], ["user", "points", "reason", "admin", "timestamp"])
            log_activity(admin_user, "RESET", "Reset all score adjustments")
            flash("Score adjustments reset", "warning")
    
    matches = read_csv(FILE_MATCHES)
    users = read_csv(FILE_USERS)
    adjustments = read_csv(FILE_POINT_ADJUSTMENTS)
    announcements = read_csv(FILE_ANNOUNCEMENTS)
    
    return render_template_string(HTML_ADMIN, 
        matches=matches, 
        teams=TEAMS, 
        all_users=users,
        adjustments=adjustments,
        announcements=announcements)

@app.route("/admin-users", methods=["GET", "POST"])
def admin_users():
    if "user" not in session or session.get("role") != "admin":
        return redirect("/")
    
    if request.method == "POST":
        action = request.form.get("action")
        admin_user = session['user']
        
        if action == "add_user":
            username = request.form.get("username")
            password = request.form.get("password")
            role = request.form.get("role")
            
            users = read_csv(FILE_USERS)
            if any(u['username'] == username for u in users):
                flash("Username already exists", "error")
            else:
                users.append({
                    "username": username, "password": password,
                    "role": role, "status": "active",
                    "created_at": datetime.now().isoformat()
                })
                write_csv(FILE_USERS, users, ["username", "password", "role", "status", "created_at"])
                log_activity(admin_user, "ADD_USER", f"Created user {username}")
                flash("User Created", "success")
                
        elif action == "ban_user":
            username = request.form.get("username")
            update_user_status(username, "banned")
            log_activity(admin_user, "BAN_USER", f"Banned {username}")
            flash(f"User {username} banned", "warning")
            
        elif action == "unban_user":
            username = request.form.get("username")
            update_user_status(username, "active")
            log_activity(admin_user, "UNBAN_USER", f"Unbanned {username}")
            flash(f"User {username} unbanned", "success")
            
        elif action == "delete_user":
            username = request.form.get("username")
            users = read_csv(FILE_USERS)
            users = [u for u in users if u['username'] != username]
            write_csv(FILE_USERS, users, ["username", "password", "role", "status", "created_at"])
            log_activity(admin_user, "DELETE_USER", f"Deleted {username}")
            flash(f"User {username} deleted", "warning")
    
    users = read_csv(FILE_USERS)
    return render_template_string(HTML_ADMIN_USERS, users=users)

@app.route("/admin-logs")
def admin_logs():
    if "user" not in session or session.get("role") != "admin":
        return redirect("/")
    
    logs = read_csv(FILE_LOGS)
    logs.reverse()
    return render_template_string(HTML_ADMIN_LOGS, logs=logs)

@app.route("/download/<data_type>")
def download(data_type):
    if "user" not in session or session.get("role") != "admin":
        return "Access Denied"
    
    files = {
        "predictions": FILE_PREDS,
        "matches": FILE_MATCHES,
        "users": FILE_USERS,
        "logs": FILE_LOGS
    }
    
    if data_type in files:
        return send_file(files[data_type], as_attachment=True)
    return "File not found"

@app.route("/logout")
def logout():
    if "user" in session:
        log_activity(session['user'], "LOGOUT", "User logged out")
    session.clear()
    return redirect("/")

# ================= RUN =================
if __name__ == "__main__":
    init_files()
    app.run(host="0.0.0.0", port=5000, debug=True)
