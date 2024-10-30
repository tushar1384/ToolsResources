# ToolsResources
Scheduling tools and resources for continous learning
# web.py
from flask import Flask, request, jsonify
import sqlite3
from datetime import datetime

app = Flask(__name__)

# --- Database setup ---
def init_db():
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS meetings (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            mentor TEXT,
            mentee TEXT,
            date TEXT,
            time TEXT,
            status TEXT DEFAULT 'Pending'
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user TEXT,
            goal TEXT,
            deadline TEXT,
            completed INTEGER DEFAULT 0
        )
    ''')
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS session_logs (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user TEXT,
            log TEXT,
            date TEXT
        )
    ''')
    conn.commit()
    conn.close()

# Helper function to get database connection
def get_db_connection():
    conn = sqlite3.connect('database.db')
    conn.row_factory = sqlite3.Row
    return conn

# --- 1. Calendar Integration - Schedule a Meeting ---
@app.route('/api/schedule_meeting', methods=['POST'])
def schedule_meeting():
    data = request.get_json()
    mentor = data['mentor']
    mentee = data['mentee']
    date = data['date']
    time = data['time']

    conn = get_db_connection()
    # Check for conflicts in date and time
    conflicts = conn.execute(
        'SELECT * FROM meetings WHERE date = ? AND time = ?', (date, time)
    ).fetchall()

    if conflicts:
        conn.close()
        return jsonify({'message': 'Conflict detected, please choose another time.'}), 409

    # Insert meeting into database if no conflicts
    conn.execute(
        'INSERT INTO meetings (mentor, mentee, date, time, status) VALUES (?, ?, ?, ?, ?)',
        (mentor, mentee, date, time, 'Scheduled')
    )
    conn.commit()
    conn.close()
    return jsonify({'message': 'Meeting scheduled successfully.'}), 201

# --- 2. Automated Meeting Scheduling (Simple Confirmation) ---
@app.route('/api/confirm_meeting', methods=['POST'])
def confirm_meeting():
    data = request.get_json()
    meeting_id = data['meeting_id']
    conn = get_db_connection()

    # Update meeting status to confirmed
    conn.execute('UPDATE meetings SET status = ? WHERE id = ?', ('Confirmed', meeting_id))
    conn.commit()
    conn.close()
    return jsonify({'message': 'Meeting confirmed successfully.'}), 200

# --- 3. Reminder Notifications ---
@app.route('/api/reminders', methods=['GET'])
def reminders():
    conn = get_db_connection()
    today = datetime.now().strftime("%Y-%m-%d")
    
    # Fetch meetings scheduled for today
    meetings = conn.execute(
        'SELECT * FROM meetings WHERE date = ?', (today,)
    ).fetchall()
    conn.close()

    # Prepare reminder messages for each meeting
    reminders = [
        f'Reminder: Meeting with {meeting["mentor"]} and {meeting["mentee"]} at {meeting["time"]}.'
        for meeting in meetings
    ]
    return jsonify({'reminders': reminders}), 200

# --- 4. Goal and Task Setting ---
@app.route('/api/add_task', methods=['POST'])
def add_task():
    data = request.get_json()
    user = data['user']
    goal = data['goal']
    deadline = data['deadline']

    conn = get_db_connection()
    conn.execute(
        'INSERT INTO tasks (user, goal, deadline) VALUES (?, ?, ?)',
        (user, goal, deadline)
    )
    conn.commit()
    conn.close()
    return jsonify({'message': 'Task added successfully.'}), 201

@app.route('/api/tasks', methods=['GET'])
def get_tasks():
    conn = get_db_connection()
    tasks = conn.execute('SELECT * FROM tasks').fetchall()
    conn.close()

    # Return all tasks as a JSON response
    task_list = [{'id': task['id'], 'user': task['user'], 'goal': task['goal'], 'deadline': task['deadline'], 'completed': bool(task['completed'])} for task in tasks]
    return jsonify({'tasks': task_list}), 200

# --- 5. Session Logs and Notes ---
@app.route('/api/add_session_log', methods=['POST'])
def add_session_log():
    data = request.get_json()
    user = data['user']
    log = data['log']
    date = datetime.now().strftime("%Y-%m-%d")

    conn = get_db_connection()
    conn.execute(
        'INSERT INTO session_logs (user, log, date) VALUES (?, ?, ?)',
        (user, log, date)
    )
    conn.commit()
    conn.close()
    return jsonify({'message': 'Session log saved successfully.'}), 201

@app.route('/api/session_logs', methods=['GET'])
def get_session_logs():
    conn = get_db_connection()
    logs = conn.execute('SELECT * FROM session_logs').fetchall()
    conn.close()

    # Return session logs as a JSON response
    log_list = [{'id': log['id'], 'user': log['user'], 'log': log['log'], 'date': log['date']} for log in logs]
    return jsonify({'session_logs': log_list}), 200

# Initialize database and run the application
if __name__ == '__main__':
    init_db()
    app.run(debug=True)
