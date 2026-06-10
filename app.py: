from flask import Flask, render_template, request, jsonify
from flask_cors import CORS
from flask_socketio import SocketIO, emit
import sqlite3
import jwt
from datetime import datetime, timedelta
from functools import wraps
import os

app = Flask(__name__)
app.config['SECRET_KEY'] = 'tu_clave_secreta_2024'
CORS(app)
socketio = SocketIO(app, cors_allowed_origins="*")

# Base de datos
DB_NAME = 'bot_trading.db'

def init_db():
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users
                 (id INTEGER PRIMARY KEY, username TEXT UNIQUE, password TEXT, balance REAL DEFAULT 10000)''')
    c.execute('''CREATE TABLE IF NOT EXISTS trades
                 (id INTEGER PRIMARY KEY, user_id INTEGER, symbol TEXT, type TEXT, price REAL, amount REAL, timestamp TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS strategies
                 (id INTEGER PRIMARY KEY, user_id INTEGER, name TEXT, type TEXT, active INTEGER DEFAULT 0)''')
    
    # Usuario demo
    try:
        c.execute("INSERT INTO users (username, password, balance) VALUES (?, ?, ?)", 
                 ('admin', 'admin123', 10000))
    except:
        pass
    
    conn.commit()
    conn.close()

init_db()

# JWT Token
def generate_token(username):
    payload = {
        'username': username,
        'exp': datetime.utcnow() + timedelta(days=30)
    }
    return jwt.encode(payload, app.config['SECRET_KEY'], algorithm='HS256')

def verify_token(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.headers.get('Authorization', '').replace('Bearer ', '')
        try:
            jwt.decode(token, app.config['SECRET_KEY'], algorithms=['HS256'])
        except:
            return jsonify({'error': 'Token inválido'}), 401
        return f(*args, **kwargs)
    return decorated

# Rutas API
@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/login', methods=['POST'])
def login():
    data = request.json
    username = data.get('username')
    password = data.get('password')
    
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("SELECT id, balance FROM users WHERE username=? AND password=?", (username, password))
    user = c.fetchone()
    conn.close()
    
    if user:
        token = generate_token(username)
        return jsonify({'success': True, 'token': token, 'balance': user[1]})
    
    return jsonify({'success': False, 'message': 'Credenciales inválidas'}), 401

@app.route('/api/balance', methods=['GET'])
@verify_token
def get_balance():
    username = request.headers.get('X-Username', 'admin')
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("SELECT balance FROM users WHERE username=?", (username,))
    balance = c.fetchone()[0]
    conn.close()
    return jsonify({'balance': balance})

@app.route('/api/trade', methods=['POST'])
@verify_token
def create_trade():
    data = request.json
    username = request.headers.get('X-Username', 'admin')
    
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("SELECT id FROM users WHERE username=?", (username,))
    user_id = c.fetchone()[0]
    
    c.execute("INSERT INTO trades (user_id, symbol, type, price, amount, timestamp) VALUES (?, ?, ?, ?, ?, ?)",
             (user_id, data['symbol'], data['type'], data['price'], data['amount'], datetime.now().isoformat()))
    conn.commit()
    conn.close()
    
    return jsonify({'success': True, 'message': 'Operación registrada'})

@app.route('/api/trades', methods=['GET'])
@verify_token
def get_trades():
    username = request.headers.get('X-Username', 'admin')
    conn = sqlite3.connect(DB_NAME)
    c = conn.cursor()
    c.execute("SELECT id, symbol, type, price, amount, timestamp FROM trades WHERE user_id IN (SELECT id FROM users WHERE username=?)", (username,))
    trades = [{'id': t[0], 'symbol': t[1], 'type': t[2], 'price': t[3], 'amount': t[4], 'timestamp': t[5]} for t in c.fetchall()]
    conn.close()
    return jsonify({'trades': trades})

@socketio.on('market_data')
def handle_market_data():
    import random
    data = {
        'BTC': round(40000 + random.uniform(-500, 500), 2),
        'ETH': round(2500 + random.uniform(-50, 50), 2),
        'XRP': round(0.5 + random.uniform(-0.05, 0.05), 4)
    }
    emit('market_update', data, broadcast=True)

if __name__ == '__main__':
    socketio.run(app, debug=True, host='0.0.0.0', port=int(os.environ.get('PORT', 5000)))
