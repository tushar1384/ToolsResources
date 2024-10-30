# ToolsResources
Scheduling tools and resources for continous learning
# web.py
from flask import Flask, request, jsonify
import sqlite3
from datetime import datetime

app = Flask(__name__)

# --- Database setup ---
def initialise_dbs():
    link= sqlite3.connect('database.db')
    cursor = jodo.cursor()
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
    link.commit()
    link.close()

# Helper function to get database connection
def get_dbs_connection():
    link = sqlite3.connect('database.db')
    link.row_factory = sqlite3.Row
    return link

# --- 1. Calendar Integration - Schedule a Meeting ---
@app.route('/api/schedule_meeting', methods=['POST'])
def schedule_meeting():
    data = request.get_json()
    mentor = data['mentor']
    mentee = data['mentee']
    date = data['date']
    time = data['time']

    link = get_dbs_connection()
    # Check for conflicts in date and time
    conflicts = link.execute(
        'SELECT * FROM meetings WHERE date = ? AND time = ?', (date, time)
    ).fetchall()

    if conflicts:
        link.close()
        return jsonify({'message': 'Conflict detected, please choose another time.'}), 409

    # Insert meeting into database if no conflicts
    link.execute(
        'INSERT INTO meetings (mentor, mentee, date, time, status) VALUES (?, ?, ?, ?, ?)',
        (mentor, mentee, date, time, 'Scheduled')
    )
    link.commit()
    link.close()
    return jsonify({'message': 'Meeting scheduled successfully.'}), 201

# --- 2. Automated Meeting Scheduling (Simple Confirmation) ---
@app.route('/api/confirm_meeting', methods=['POST'])
def confirm_meet():
    data = request.get_json()
    meet_id = data['meeting_id']
    link = get_dbs_connection()

    # Update meeting status to confirmed
    link.execute('UPDATE meetings SET status = ? WHERE id = ?', ('Confirmed', meet_id))
    link.commit()
    link.close()
    return jsonify({'message': 'Meeting confirmed successfully.'}), 200

# --- 3. Reminder Notifications ---
@app.route('/api/reminders', methods=['GET'])
def reminders():
    link = get_dbs_connection()
    today = datetime.now().strftime("%Y-%m-%d")
    
    # Fetch meetings scheduled for today
    meetings = link.execute(
        'SELECT * FROM meetings WHERE date = ?', (today,)
    ).fetchall()
    link.close()

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
    target = data['goal']
    deadline = data['deadline']

    link = get_db_connection()
    link.execute(
        'INSERT INTO tasks (user, goal, deadline) VALUES (?, ?, ?)',
        (user, target, deadline)
    )
    link.commit()
    link.close()
    return jsonify({'message': 'Task added successfully.'}), 201

@app.route('/api/tasks', methods=['GET'])
def get_tasks():
    link = get_dbs_connection()
    tasks = link.execute('SELECT * FROM tasks').fetchall()
    link.close()

    # Return all tasks as a JSON response
    tasks_array = [{'id': task['id'], 'user': task['user'], 'goal': task['goal'], 'deadline': task['deadline'], 'completed': bool(task['completed'])} for task in tasks]
    return jsonify({'tasks': tasks_array}), 200

# --- 5. Session Logs and Notes ---
@app.route('/api/add_session_log', methods=['POST'])
def add_session_log():
    data = request.get_json()
    user = data['user']
    log = data['log']
    date = datetime.now().strftime("%Y-%m-%d")

    link = get_dbs_connection()
    link.execute(
        'INSERT INTO session_logs (user, log, date) VALUES (?, ?, ?)',
        (user, log, date)
    )
    link.commit()
    link.close()
    return jsonify({'message': 'Session log saved successfully...'}), 201

@app.route('/api/session_logs', methods=['GET'])
def get_session_logs():
    link = get_dbs_connection()
    logs = link.execute('SELECT * FROM session_logs').fetchall()
    link.close()

    # Return session logs as a JSON response
    log_list = [{'id': log['id'], 'user': log['user'], 'log': log['log'], 'date': log['date']} for log in logs]
    return jsonify({'session_logs': log_list}), 200

# Initialize database and run the application
if __name__ == '__main__':
    initialise_dbs()
    app.run(debug=True)
