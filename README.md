# own-project
Had made this project to keep track on my own things of my personal and college
from flask import Flask, request, redirect, url_for, render_template_string, flash
import sqlite3
from pathlib import Path
from datetime import datetime

app = Flask(__name__)
app.secret_key = "change-this-secret-key"
DB_PATH = Path(__file__).with_name("student_assistant.db")

def db():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    with db() as conn:
        conn.execute("""
            CREATE TABLE IF NOT EXISTS tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                category TEXT NOT NULL DEFAULT 'Personal',
                due_date TEXT,
                priority TEXT NOT NULL DEFAULT 'Medium',
                done INTEGER NOT NULL DEFAULT 0,
                created_at TEXT NOT NULL
            )
        """)
        conn.execute("""
            CREATE TABLE IF NOT EXISTS jobs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                company TEXT NOT NULL,
                role TEXT NOT NULL,
                link TEXT,
                deadline TEXT,
                status TEXT NOT NULL DEFAULT 'Interested',
                notes TEXT
            )
        """)

PAGE = r"""
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Student Companion</title>
<style>
:root{--bg:#f4f6fb;--card:#fff;--ink:#1d2433;--muted:#697386;--accent:#6558e8;--line:#e7eaf2}
*{box-sizing:border-box}body{margin:0;font-family:Inter,Segoe UI,Arial,sans-serif;background:var(--bg);color:var(--ink)}
header{background:linear-gradient(120deg,#5145cd,#8877f4);color:white;padding:28px max(20px,calc((100% - 1150px)/2))}
header h1{margin:0 0 8px;font-size:clamp(25px,4vw,36px)}header p{margin:0;opacity:.9}
main{max-width:1150px;margin:24px auto;padding:0 18px 40px}
.topline{display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap;margin-bottom:18px}
.pill{background:#eceaff;color:#5145cd;padding:8px 12px;border-radius:999px;font-size:13px;font-weight:700}
.stats{display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:14px;margin-bottom:18px}
.card{background:var(--card);border:1px solid var(--line);border-radius:16px;padding:18px;box-shadow:0 5px 18px #242c4a08}
.stat-number{font-size:30px;font-weight:800;margin-top:6px}.muted{color:var(--muted);font-size:13px}
.layout{display:grid;grid-template-columns:1.05fr .95fr;gap:16px}.section-title{margin:0 0 14px;font-size:19px}
form{display:grid;gap:10px}input,select,textarea{width:100%;padding:11px 12px;border:1px solid #dce1ed;border-radius:10px;background:#fff;font:inherit;color:var(--ink)}
button,.button{border:0;border-radius:10px;padding:11px 14px;background:var(--accent);color:#fff;font-weight:700;cursor:pointer;text-decoration:none;display:inline-block}
button:hover,.button:hover{filter:brightness(.95)}button.secondary{background:#edf0f8;color:#323b50}
.form-grid{display:grid;grid-template-columns:1fr 1fr;gap:10px}.item{display:flex;align-items:flex-start;gap:10px;padding:13px 0;border-bottom:1px solid var(--line)}.item:last-child{border-bottom:0}
.item-main{flex:1;min-width:0}.item-title{font-weight:700;overflow-wrap:anywhere}.done{text-decoration:line-through;color:#8992a4}.tag{display:inline-block;background:#f0f2f8;border-radius:7px;padding:4px 7px;font-size:11px;margin:5px 4px 0 0;color:#4e5870}
.inline-form{display:inline}.inline-form button{padding:7px 9px;font-size:12px}.actions{display:flex;gap:6px;flex-wrap:wrap;margin-top:8px}.empty{padding:20px 0;color:var(--muted);font-size:14px}
.notice{background:#fff8df;border:1px solid #f3e2a1;color:#6c5714;padding:12px 14px;border-radius:12px;margin-bottom:16px;font-size:13px}
.reminder-row{display:flex;justify-content:space-between;align-items:center;gap:10px;margin:12px 0}
footer{text-align:center;color:var(--muted);font-size:12px;margin-top:24px}
@media(max-width:760px){.stats{grid-template-columns:1fr 1fr}.layout{grid-template-columns:1fr}.stats .card:last-child{grid-column:1/-1}}
@media(max-width:440px){.stats{grid-template-columns:1fr 1fr}.form-grid{grid-template-columns:1fr}}
</style>
</head>
<body>
<header>
  <h1>🎓 Student Companion</h1>
  <p>Your everyday space for tasks, study goals, applications, and healthy routines.</p>
</header>
<main>
  <div class="topline">
    <div><strong id="today"></strong><div class="muted">Small steps today. Better opportunities tomorrow.</div></div>
    <span class="pill">YOUR PERSONAL DASHBOARD</span>
  </div>
  {% with messages = get_flashed_messages() %}
    {% if messages %}{% for message in messages %}<div class="notice">{{ message }}</div>{% endfor %}{% endif %}
  {% endwith %}
  <div class="notice">Reminder note: browser notifications work while this dashboard is open. Allow notification permission when asked. This starter version does not send alerts when the browser is closed.</div>
  <section class="stats">
    <div class="card"><div class="muted">Tasks left</div><div class="stat-number">{{ pending }}</div><div class="muted">Keep your next step clear.</div></div>
    <div class="card"><div cla
    ss="muted">Completed tasks</div><div class="stat-number">{{ completed }}</div><div class="muted">Progress counts.</div></div>
    <div class="card"><div class="muted">Applications tracked</div><div class="stat-number">{{ job_count }}</div><div class="muted">Follow up on opportunities.</div></div>
  </section>

  <div class="layout">
    <section class="card">
      <h2 class="section-title">✅ Tasks & deadlines</h2>
      <form method="post" action="{{ url_for('add_task') }}">
        <input name="title" placeholder="What do you need to do?" maxlength="180" required>
        <div class="form-grid">
          <select name="category">
            <option>College</option><option>DSA / Placement</option><option>Project</option><option>Internship</option><option>Personal</option><option>Health</option>
          </select>
          <select name="priority"><option>High</option><option selected>Medium</option><option>Low</option></select>
          <input name="due_date" type="date" aria-label="Due date">
          <button type="submit">+ Add task</button>
        </div>
      </form>
      <div style="margin-top:16px">
      {% for task in tasks %}
        <div class="item">
          <div class="item-main">
            <div class="item-title {% if task['done'] %}done{% endif %}">{{ task['title'] }}</div>
            <span class="tag">{{ task['category'] }}</span><span class="tag">{{ task['priority'] }} priority</span>
            {% if task['due_date'] %}<span class="tag">Due {{ task['due_date'] }}</span>{% endif %}
            <div class="actions">
              <form class="inline-form" method="post" action="{{ url_for('toggle_task', task_id=task['id']) }}"><button class="secondary" type="submit">{% if task['done'] %}Reopen{% else %}Mark done{% endif %}</button></form>
              <form class="inline-form" method="post" action="{{ url_for('delete_task', task_id=task['id']) }}" onsubmit="return confirm('Delete this task?')"><button class="secondary" type="submit">Delete</button></form>
            </div>
          </div>
        </div>
      {% else %}<div class="empty">No tasks yet. Add your first task above.</div>{% endfor %}
      </div>
    </section>

    <section class="card">
      <h2 class="section-title">💼 Internship & job tracker</h2>
      <form method="post" action="{{ url_for('add_job') }}">
        <input name="company" placeholder="Company name" maxlength="120" required>
        <input name="role" placeholder="Role / internship title" maxlength="160" required>
        <input name="link" type="url" placeholder="Application URL (optional)">
        <div class="form-grid">
          <input name="deadline" type="date" aria-label="Application deadline">
          <select name="status"><option>Interested</option><option>Applied</option><option>Assessment</option><option>Interview</option><option>Selected</option><option>Rejected</option></select>
        </div>
        <textarea name="notes" rows="2" placeholder="Notes (optional)"></textarea>
        <button type="submit">+ Save opportunity</button>
      </form>
      <div style="margin-top:16px">
      {% for job in jobs %}
        <div class="item">
          <div class="item-main">
            <div class="item-title">{{ job['role'] }}</div>
            <div class="muted">{{ job['company'] }}{% if job['deadline'] %} · Deadline {{ job['deadline'] }}{% endif %}</div>
            <span class="tag">{{ job['status'] }}</span>
            {% if job['notes'] %}<div class="muted" style="margin-top:6px">{{ job['notes'] }}</div>{% endif %}
            {% if job['link'] %}<div style="margin-top:6px"><a href="{{ job['link'] }}" target="_blank" rel="noopener noreferrer">Open application ↗</a></div>{% endif %}
            <div class="actions">
              <form class="inline-form" method="post" action="{{ url_for('cycle_job', job_id=job['id']) }}"><button class="secondary" type="submit">Next status</button></form>
              <form class="inline-form" method="post" action="{{ url_for('delete_job', job_id=job['id']) }}" onsubmit="return confirm('Delete this opportunity?')"><button class="secondary" type="submit">Delete</button></form>
            </div>
          </div>
        </div>
      {% else %}<div class="empty">Save internships and jobs here so you can follow up.</div>{% endfor %}
      </div>
    </section>
  </div>

  <section class="card" style="margin-top:16px">
    <h2 class="section-title">⏰ Daily routine reminders</h2>
    <p class="muted">Choose the reminders you want. They run in this browser tab while the dashboard remains open.</p>
    <div class="reminder-row"><div><strong>Water break</strong><div class="muted">Reminder every 60 minutes</div></div><label><input id="waterToggle" type="checkbox" checked style="width:auto"> On</label></div>
    <div class="reminder-row"><div><strong>Stretch / eye break</strong><div class="muted">Reminder every 50 minutes</div></div><label><input id="breakToggle" type="checkbox" checked style="width:auto"> On</label></div>
    <div class="reminder-row"><div><strong>Plan tomorrow</strong><div class="muted">One reminder in 2 minutes for demo; adjust before real use</div></div><label><input id="planToggle" type="checkbox"> On</label></div>
    <button type="button" id="enableNotifications">Enable browser notifications</button>
    <button type="button" class="secondary" id="testReminder">Test reminder</button>
    <div id="reminderStatus" class="muted" style="margin-top:10px">Notifications have not been enabled yet.</div>
  </section>
  <footer>Student Companion · Starter MVP · Your data is stored locally in student_assistant.db on this computer.</footer>
</main>
<script>
const now = new Date();
document.getElementById('today').textContent = now.toLocaleDateString(undefined,{weekday:'long',year:'numeric',month:'long',day:'numeric'});
const status = document.getElementById('reminderStatus');
function notify(title, body) {
  if (!('Notification' in window)) { status.textContent = 'This browser does not support notifications.'; alert(title + '\\n' + body); return; }
  if (Notification.permission === 'granted') new Notification(title, {body});
  else { status.textContent = 'Allow notifications first, then test again.'; }
}
document.getElementById('enableNotifications').addEventListener('click', async () => {
  if (!('Notification' in window)) { status.textContent = 'This browser does not support notifications.'; return; }
  const permission = await Notification.requestPermission();
  status.textContent = permission === 'granted' ? 'Notifications enabled. Keep this tab open for reminders.' : 'Permission not granted. You can enable it in browser site settings.';
});
document.getElementById('testReminder').addEventListener('click', () => notify('Student Companion', 'This is a test reminder. You are all set to test your routine.'));
setInterval(() => {
  if (document.getElementById('waterToggle').checked) notify('Time for water 💧', 'Take a water break if you need one.');
}, 60 * 60 * 1000);
setInterval(() => {
  if (document.getElementById('breakToggle').checked) notify('Take a short break 🌿', 'Look away from the screen and stretch for a moment.');
}, 50 * 60 * 1000);
document.getElementById('planToggle').addEventListener('change', (e) => {
  if (e.target.checked) {
    status.textContent = 'Demo reminder scheduled for 2 minutes from now.';
    setTimeout(() => { if (document.getElementById('planToggle').checked) notify('Plan tomorrow 📝', 'Review your pending tasks and choose your top priorities.'); }, 2 * 60 * 1000);
  }
});
</script>
</body>
</html>
"""

@app.route("/")
def home():
    with db() as conn:
        tasks = conn.execute("SELECT * FROM tasks ORDER BY done ASC, CASE priority WHEN 'High' THEN 1 WHEN 'Medium' THEN 2 ELSE 3 END, COALESCE(due_date, '9999-12-31'), id DESC").fetchall()
        jobs = conn.execute("SELECT * FROM jobs ORDER BY CASE status WHEN 'Interested' THEN 1 WHEN 'Applied' THEN 2 WHEN 'Assessment' THEN 3 WHEN 'Interview' THEN 4 ELSE 5 END, COALESCE(deadline, '9999-12-31'), id DESC").fetchall()
        pending = conn.execute("SELECT COUNT(*) FROM tasks WHERE done=0").fetchone()[0]
        completed = conn.execute("SELECT COUNT(*) FROM tasks WHERE done=1").fetchone()[0]
        job_count = conn.execute("SELECT COUNT(*) FROM jobs").fetchone()[0]
    return render_template_string(PAGE, tasks=tasks, jobs=jobs, pending=pending, completed=completed, job_count=job_count)

@app.post("/tasks/add")
def add_task():
    title = request.form.get("title", "").strip()
    if title:
        with db() as conn:
            conn.execute("INSERT INTO tasks(title, category, due_date, priority, created_at) VALUES (?, ?, ?, ?, ?)",
                         (title, request.form.get("category", "Personal"), request.form.get("due_date") or None,
                          request.form.get("priority", "Medium"), datetime.now().isoformat(timespec="seconds")))
        flash("Task added.")
    return redirect(url_for("home"))

@app.post("/tasks/<int:task_id>/toggle")
def toggle_task(task_id):
    with db() as conn:
        conn.execute("UPDATE tasks SET done = CASE done WHEN 0 THEN 1 ELSE 0 END WHERE id=?", (task_id,))
    return redirect(url_for("home"))

@app.post("/tasks/<int:task_id>/delete")
def delete_task(task_id):
    with db() as conn:
        conn.execute("DELETE FROM tasks WHERE id=?", (task_id,))
    return redirect(url_for("home"))

@app.post("/jobs/add")
def add_job():
    company = request.form.get("company", "").strip()
    role = request.form.get("role", "").strip()
    if company and role:
        with db() as conn:
            conn.execute("INSERT INTO jobs(company, role, link, deadline, status, notes) VALUES (?, ?, ?, ?, ?, ?)",
                         (company, role, request.form.get("link", "").strip() or None,
                          request.form.get("deadline") or None, request.form.get("status", "Interested"),
                          request.form.get("notes", "").strip() or None))
        flash("Opportunity saved.")
    return redirect(url_for("home"))

@app.post("/jobs/<int:job_id>/cycle")
def cycle_job(job_id):
    statuses = ["Interested", "Applied", "Assessment", "Interview", "Selected", "Rejected"]
    with db() as conn:
        row = conn.execute("SELECT status FROM jobs WHERE id=?", (job_id,)).fetchone()
        if row:
            next_status = statuses[(statuses.index(row["status"]) + 1) % len(statuses)] if row["status"] in statuses else statuses[0]
            conn.execute("UPDATE jobs SET status=? WHERE id=?", (next_status, job_id))
    return redirect(url_for("home"))

@app.post("/jobs/<int:job_id>/delete")
def delete_job(job_id):
    with db() as conn:
        conn.execute("DELETE FROM jobs WHERE id=?", (job_id,))
    return redirect(url_for("home"))

if __name__ == "__main__":
    init_db()
    print("Student Companion is starting. Open http://127.0.0.1:5000 in your browser.")
    app.run(debug=True)
