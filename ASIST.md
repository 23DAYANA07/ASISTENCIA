# ASISTENCIA
import os, io, csv, sqlite3, secrets
from datetime import datetime, date, timedelta
from zoneinfo import ZoneInfo
from functools import wraps
from flask import Flask, request, redirect, url_for, session, render_template_string, send_file, abort, flash
from werkzeug.middleware.proxy_fix import ProxyFix
import qrcode

APP_DIR = os.path.dirname(os.path.abspath(__file__))
DATA_DIR = os.getenv("DATA_DIR", APP_DIR)
os.makedirs(DATA_DIR, exist_ok=True)
DB_PATH = os.path.join(DATA_DIR, "asistencias.db")
TZ = ZoneInfo(os.getenv("APP_TIMEZONE", "America/Mexico_City"))
app = Flask(__name__)
app.secret_key = os.getenv("SECRET_KEY") or "CAMBIAR-SECRET-KEY-ANTES-DE-PUBLICAR"
app.wsgi_app = ProxyFix(app.wsgi_app, x_proto=1, x_host=1)
ADMIN_PASSWORD = os.getenv("ADMIN_PASSWORD", "CAMBIAR-ESTA-CONTRASENA")
BASE_URL = os.getenv("BASE_URL", "").rstrip("/")

def now():
    return datetime.now(TZ)

def db():
    con = sqlite3.connect(DB_PATH, timeout=30)
    con.row_factory = sqlite3.Row
    return con

def init_db():
    with db() as con:
        con.execute("""CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL,
            token TEXT NOT NULL UNIQUE, active INTEGER NOT NULL DEFAULT 1)""")
        con.execute("""CREATE TABLE IF NOT EXISTS attendance (
            id INTEGER PRIMARY KEY AUTOINCREMENT, employee_id INTEGER NOT NULL,
            event TEXT NOT NULL CHECK(event IN ('ENTRADA','SALIDA')),
            timestamp TEXT NOT NULL, FOREIGN KEY(employee_id) REFERENCES employees(id))""")
init_db()

def admin_required(fn):
    @wraps(fn)
    def wrapped(*args, **kwargs):
        if not session.get("admin"):
            return redirect(url_for("login"))
        return fn(*args, **kwargs)
    return wrapped

STYLE = """
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
*{box-sizing:border-box}body{font-family:Arial,sans-serif;background:#f3f5f7;color:#17212b;margin:0}
.wrap{max-width:1050px;margin:25px auto;padding:0 15px}.card{background:#fff;border-radius:14px;padding:21px;margin:14px 0;box-shadow:0 3px 14px #0000000c}
h1{font-size:25px;margin:0 0 8px}h2{font-size:19px;margin:0 0 13px}.muted{color:#667085}
input,select,button{font-size:15px;padding:11px;border-radius:8px;border:1px solid #ccd2d8}
button,.btn{background:#17212b;color:white;border:0;cursor:pointer;text-decoration:none;display:inline-block;padding:11px 15px;border-radius:8px}
.btn.secondary{background:#e8edf2;color:#17212b}.row{display:flex;gap:10px;flex-wrap:wrap;align-items:center}
.row input{flex:1;min-width:170px}table{width:100%;border-collapse:collapse}th,td{text-align:left;padding:10px;border-bottom:1px solid #e8ebee;font-size:14px}
.notice{padding:12px;border-radius:8px;background:#edf5ff;margin:10px 0}.ok{background:#e8f7ed}.warn{background:#fff4e5}
.big{font-size:22px;font-weight:bold}.center{text-align:center}.small{font-size:13px}
@media(max-width:650px){.wrap{margin:12px auto}.card{padding:15px}table{display:block;overflow-x:auto;white-space:nowrap}h1{font-size:22px}}
</style>
"""

def get_range():
    today=now().date()
    start=request.args.get("start") or (today-timedelta(days=today.weekday())).isoformat()
    end=request.args.get("end") or today.isoformat()
    try:
        date.fromisoformat(start); date.fromisoformat(end)
    except ValueError: abort(400,"Fechas inválidas")
    if start>end: abort(400,"La fecha inicial no puede ser posterior a la final")
    return start,end

def csv_response(rows, filename, headers):
    out=io.StringIO()
    w=csv.writer(out)
    w.writerow(headers)
    for row in rows: w.writerow(row)
    data=("\ufeff"+out.getvalue()).encode("utf-8-sig")
    return send_file(io.BytesIO(data),mimetype="text/csv; charset=utf-8",
                     as_attachment=True,download_name=filename)

@app.route("/")
def home():
    return redirect(url_for("admin"))

@app.route("/login",methods=["GET","POST"])
def login():
    if request.method=="POST":
        if secrets.compare_digest(request.form.get("password",""),ADMIN_PASSWORD):
            session["admin"]=True
            session.permanent=True
            return redirect(url_for("admin"))
        flash("Contraseña incorrecta.")
    return render_template_string(STYLE+"""
    <div class="wrap"><div class="card" style="max-width:430px;margin:65px auto">
    <h1>Autoservicio JHONY</h1><h2>Control de asistencia QR</h2>
    {% with ms=get_flashed_messages() %}{% for m in ms %}<div class="notice warn">{{m}}</div>{% endfor %}{% endwith %}
    <form method="post"><input name="password" type="password" placeholder="Contraseña de administrador" required style="width:100%;margin:10px 0">
    <button style="width:100%">Iniciar sesión</button></form></div></div>""")

@app.route("/logout")
def logout():
    session.clear()
    return redirect(url_for("login"))

@app.route("/admin")
@admin_required
def admin():
    start,end=get_range()
    with db() as con:
        employees=con.execute("SELECT * FROM employees ORDER BY name").fetchall()
        today=now().date().isoformat()
        today_rows=con.execute("""SELECT a.*,e.name FROM attendance a JOIN employees e ON e.id=a.employee_id
            WHERE substr(a.timestamp,1,10)=? ORDER BY a.timestamp DESC""",(today,)).fetchall()
    return render_template_string(STYLE+"""
    <div class="wrap">
      <div class="row" style="justify-content:space-between"><div><h1>Autoservicio JHONY</h1><div class="muted">Panel de asistencia · {{today}}</div></div><a class="btn secondary" href="/logout">Cerrar sesión</a></div>
      {% with ms=get_flashed_messages() %}{% for m in ms %}<div class="notice">{{m}}</div>{% endfor %}{% endwith %}
      <div class="card"><h2>Registrar trabajador</h2><form class="row" method="post" action="/employee/add">
        <input name="name" placeholder="Nombre completo" maxlength="100" required><button>Agregar trabajador</button></form>
        <p class="muted small">Se generará un QR personal. Puedes abrirlo e imprimirlo desde la lista.</p></div>
      <div class="card"><h2>Trabajadores ({{employees|length}})</h2>
      {% if employees %}<table><tr><th>Nombre</th><th>Estado</th><th>QR</th><th>Acción</th></tr>
      {% for e in employees %}<tr><td><b>{{e.name}}</b></td><td>{{'Activo' if e.active else 'Inactivo'}}</td>
      <td><a href="/qr/{{e.id}}" target="_blank">Ver / imprimir QR</a></td><td><form method="post" action="/employee/toggle/{{e.id}}"><button class="btn secondary">{{'Desactivar' if e.active else 'Activar'}}</button></form></td></tr>{% endfor %}</table>
      {% else %}<p class="muted">Aún no hay trabajadores.</p>{% endif %}</div>
      <div class="card"><h2>Movimientos de hoy</h2>
      {% if today_rows %}<table><tr><th>Trabajador</th><th>Movimiento</th><th>Hora local</th></tr>
      {% for r in today_rows %}<tr><td>{{r.name}}</td><td>{{r.event}}</td><td>{{r.timestamp[11:19]}}</td></tr>{% endfor %}</table>
      {% else %}<p class="muted">Todavía no hay movimientos registrados hoy.</p>{% endif %}</div>
      <div class="card"><h2>Reportes de asistencia</h2>
        <p class="muted">Selecciona el periodo. Los archivos CSV se abren directamente con Excel.</p>
        <form class="row" method="get" action="/admin">
          <label>Desde <input type="date" name="start" value="{{start}}" required></label>
          <label>Hasta <input type="date" name="end" value="{{end}}" required></label>
          <button>Aplicar periodo</button></form>
        <div class="row" style="margin-top:14px">
          <a class="btn" href="/export/general?start={{start}}&end={{end}}">Descargar reporte general</a>
          <form class="row" method="get" action="/export/employee">
            <input type="hidden" name="start" value="{{start}}"><input type="hidden" name="end" value="{{end}}">
            <select name="employee_id" required><option value="">Selecciona trabajador</option>
            {% for e in employees if e.active %}<option value="{{e.id}}">{{e.name}}</option>{% endfor %}</select>
            <button>Descargar reporte individual</button></form>
        </div>
      </div>
      <div class="card"><h2>Uso del sistema</h2><ol><li>Agrega a cada trabajador y abre su QR.</li><li>El trabajador escanea su código con la cámara del celular.</li><li>Confirma el botón de registrar entrada o salida.</li><li>Descarga el reporte general o el individual al cierre de semana.</li></ol>
      <div class="notice warn"><b>Nota:</b> el QR identifica al trabajador, pero no demuestra por sí solo que esté físicamente en el negocio. Evita compartir o publicar los códigos. Los registros usan la hora del servidor en zona horaria de Ciudad de México.</div></div>
    </div>""",employees=employees,today_rows=today_rows,today=today,start=start,end=end)

@app.route("/employee/add",methods=["POST"])
@admin_required
def add_employee():
    name=request.form.get("name","").strip()
    if not name: abort(400)
    with db() as con:
        count=con.execute("SELECT COUNT(*) FROM employees").fetchone()[0]
        if count>=100: 
            flash("Se alcanzó el límite de 100 registros de trabajadores.")
            return redirect(url_for("admin"))
        con.execute("INSERT INTO employees(name,token) VALUES(?,?)",(name,secrets.token_urlsafe(24)))
    flash("Trabajador agregado. Abre 'Ver / imprimir QR' para obtener su código.")
    return redirect(url_for("admin"))

@app.route("/employee/toggle/<int:employee_id>",methods=["POST"])
@admin_required
def toggle_employee(employee_id):
    with db() as con: con.execute("UPDATE employees SET active=1-active WHERE id=?",(employee_id,))
    return redirect(url_for("admin"))

@app.route("/qr/<int:employee_id>")
@admin_required
def qr_page(employee_id):
    with db() as con: e=con.execute("SELECT * FROM employees WHERE id=?",(employee_id,)).fetchone()
    if not e: abort(404)
    base=BASE_URL or request.url_root.rstrip("/")
    target=f"{base}/marcar/{e['token']}"
    img=qrcode.make(target)
    buf=io.BytesIO(); img.save(buf,format="PNG"); buf.seek(0)
    return send_file(buf,mimetype="image/png",download_name=f"QR_{e['name']}.png")

@app.route("/marcar/<token>",methods=["GET","POST"])
def mark(token):
    with db() as con:
        e=con.execute("SELECT * FROM employees WHERE token=? AND active=1",(token,)).fetchone()
        if not e: abort(404)
        last=con.execute("SELECT event,timestamp FROM attendance WHERE employee_id=? ORDER BY id DESC LIMIT 1",(e["id"],)).fetchone()
    def next_move(lastrow):
        if lastrow and lastrow["event"]=="ENTRADA" and lastrow["timestamp"][:10]==now().date().isoformat():
            return "SALIDA"
        return "ENTRADA"
    event=next_move(last)
    if request.method=="POST":
        stamp=now().isoformat(timespec="seconds")
        with db() as con:
            con.execute("BEGIN IMMEDIATE")
            last=con.execute("SELECT event,timestamp FROM attendance WHERE employee_id=? ORDER BY id DESC LIMIT 1",(e["id"],)).fetchone()
            event=next_move(last)
            # Prevent accidental repeated scans of the same action within 45 seconds.
            if last and last["event"]==event and last["timestamp"][:10]==now().date().isoformat():
                previous=datetime.fromisoformat(last["timestamp"])
                if (now()-previous).total_seconds()<45:
                    return render_template_string(STYLE+"""<div class="wrap"><div class="card center"><h1>Registro reciente</h1><p>Ya se registró un movimiento hace unos segundos. Espera antes de volver a escanear.</p></div></div>""")
            con.execute("INSERT INTO attendance(employee_id,event,timestamp) VALUES(?,?,?)",(e["id"],event,stamp))
        return render_template_string(STYLE+"""
        <div class="wrap"><div class="card center" style="max-width:520px;margin:45px auto"><h1>¡Registro realizado!</h1>
        <div class="notice ok big">{{event}} registrada</div><p><b>{{name}}</b></p><p class="big">{{stamp}}</p><p class="muted">Puedes cerrar esta página.</p></div></div>""",
        event=event,name=e["name"],stamp=stamp.replace("T"," "))
    return render_template_string(STYLE+"""
    <div class="wrap"><div class="card center" style="max-width:520px;margin:45px auto"><h1>Autoservicio JHONY</h1><h2>Registro de asistencia</h2>
    <p class="muted">Trabajador</p><div class="big">{{name}}</div><div class="notice">Siguiente movimiento: <b>{{event}}</b></div>
    {% if last %}<p class="muted small">Último movimiento: {{last.event}} · {{last.timestamp.replace('T',' ')}}</p>{% endif %}
    <form method="post"><button style="font-size:20px;width:100%;padding:18px">Confirmar {{event}}</button></form>
    <p class="muted small">Confirma solo al llegar o al terminar tu jornada.</p></div></div>""",name=e["name"],event=event,last=last)

@app.route("/export/general")
@admin_required
def export_general():
    start,end=get_range()
    with db() as con:
        rows=con.execute("""SELECT e.name,a.event,a.timestamp FROM attendance a JOIN employees e ON e.id=a.employee_id
            WHERE substr(a.timestamp,1,10) BETWEEN ? AND ? ORDER BY e.name,a.timestamp""",(start,end)).fetchall()
        summary=con.execute("""SELECT e.name,
            SUM(CASE WHEN a.event='ENTRADA' THEN 1 ELSE 0 END) entradas,
            SUM(CASE WHEN a.event='SALIDA' THEN 1 ELSE 0 END) salidas
            FROM employees e LEFT JOIN attendance a ON a.employee_id=e.id
            AND substr(a.timestamp,1,10) BETWEEN ? AND ? GROUP BY e.id ORDER BY e.name""",(start,end)).fetchall()
    out=io.StringIO(); w=csv.writer(out)
    w.writerow(["REPORTE GENERAL DE ASISTENCIA",start+" a "+end]); w.writerow([])
    w.writerow(["RESUMEN POR TRABAJADOR","Entradas","Salidas"])
    for r in summary: w.writerow([r["name"],r["entradas"] or 0,r["salidas"] or 0])
    w.writerow([]); w.writerow(["DETALLE DE MOVIMIENTOS"]); w.writerow(["Trabajador","Movimiento","Fecha y hora"])
    for r in rows: w.writerow([r["name"],r["event"],r["timestamp"].replace("T"," ")])
    return csv_response([],f"asistencia_general_{start}_a_{end}.csv",[]) if False else send_file(
        io.BytesIO(("\ufeff"+out.getvalue()).encode("utf-8-sig")),mimetype="text/csv; charset=utf-8",
        as_attachment=True,download_name=f"asistencia_general_{start}_a_{end}.csv")

@app.route("/export/employee")
@admin_required
def export_employee():
    start,end=get_range()
    employee_id=request.args.get("employee_id","")
    try: employee_id=int(employee_id)
    except ValueError: abort(400,"Selecciona un trabajador")
    with db() as con:
        emp=con.execute("SELECT * FROM employees WHERE id=?",(employee_id,)).fetchone()
        if not emp: abort(404)
        rows=con.execute("""SELECT event,timestamp FROM attendance WHERE employee_id=?
            AND substr(timestamp,1,10) BETWEEN ? AND ? ORDER BY timestamp""",(employee_id,start,end)).fetchall()
    entries=sum(1 for r in rows if r["event"]=="ENTRADA")
    exits=sum(1 for r in rows if r["event"]=="SALIDA")
    out=io.StringIO(); w=csv.writer(out)
    w.writerow(["REPORTE INDIVIDUAL DE ASISTENCIA"]); w.writerow(["Trabajador",emp["name"]])
    w.writerow(["Periodo",start+" a "+end]); w.writerow(["Total de entradas",entries]); w.writerow(["Total de salidas",exits])
    w.writerow([]); w.writerow(["Movimiento","Fecha y hora"])
    for r in rows: w.writerow([r["event"],r["timestamp"].replace("T"," ")])
    return send_file(io.BytesIO(("\ufeff"+out.getvalue()).encode("utf-8-sig")),
        mimetype="text/csv; charset=utf-8",as_attachment=True,
        download_name=f"asistencia_{employee_id}_{start}_a_{end}.csv")

@app.get("/health")
def health():
    return {"status":"ok"}

if __name__=="__main__":
    app.run(host="0.0.0.0",port=int(os.getenv("PORT","5000")),debug=False)

        value: America/Mexico_City
      - key: ADMIN_PASSWORD
        sync: false
      - key: SECRET_KEY
        generateValue: true

 Flask>=3.0,<4.0
qrcode[pil]>=7.4,<9.0
gunicorn>=22.0,<24.0

services:
  - type: web
    name: asistencia-qr-autoservicio-jhony
    runtime: python
    plan: starter
    buildCommand: pip install -r requirements.txt
    startCommand: gunicorn app:app
    healthCheckPath: /health
    disk:
      name: asistencia-data
      mountPath: /var/data
      sizeGB: 1
    envVars:
      - key: DATA_DIR
        value: /var/data
      - key: APP_TIMEZONE       
