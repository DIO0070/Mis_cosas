# CADE-SIMU a distancia

Para hacer el programa accesible de forma remota, podemos implementar una versión cliente-servidor usando una API REST. Aquí te muestro cómo modificar el código para permitir operaciones remotas:

```python
from flask import Flask, request, jsonify
from flask_httpauth import HTTPBasicAuth
import json
import threading

app = Flask(__name__)
auth = HTTPBasicAuth()

users = {
    "admin": "cade123"
}

@auth.verify_password
def verify_password(username, password):
    if username in users and users[username] == password:
        return username

class RemoteCadeManager:
    def __init__(self):
        self.projects = {}
        self.lock = threading.Lock()

    def create_project(self, project_id):
        with self.lock:
            self.projects[project_id] = {
                'components': [],
                'connections': [],
                'simulations': []
            }
        return {"status": "Project created", "project_id": project_id}

manager = RemoteCadeManager()

@app.route('/api/project/<string:project_id>', methods=['POST'])
@auth.login_required
def create_project(project_id):
    return jsonify(manager.create_project(project_id))

@app.route('/api/project/<string:project_id>/component', methods=['POST'])
@auth.login_required
def add_component(project_id):
    data = request.json
    required = ['type', 'position']
    if not all(k in data for k in required):
        return jsonify({"error": "Missing parameters"}), 400
    
    with manager.lock:
        if project_id not in manager.projects:
            return jsonify({"error": "Project not found"}), 404
            
        new_component = {
            'id': len(manager.projects[project_id]['components']) + 1,
            'type': data['type'],
            'position': data['position'],
            'label': data.get('label', "")
        }
        
        manager.projects[project_id]['components'].append(new_component)
        return jsonify(new_component)

@app.route('/api/project/<string:project_id>/connect', methods=['POST'])
@auth.login_required
def connect_components(project_id):
    data = request.json
    required = ['source_id', 'output_pin', 'dest_id', 'input_pin']
    if not all(k in data for k in required):
        return jsonify({"error": "Missing parameters"}), 400
    
    with manager.lock:
        if project_id not in manager.projects:
            return jsonify({"error": "Project not found"}), 404
            
        connection = {
            'from': {'id': data['source_id'], 'pin': data['output_pin']},
            'to': {'id': data['dest_id'], 'pin': data['input_pin']}
        }
        
        manager.projects[project_id]['connections'].append(connection)
        return jsonify({"status": "Connected"})

@app.route('/api/project/<string:project_id>/simulate', methods=['POST'])
@auth.login_required
def simulate(project_id):
    with manager.lock:
        if project_id not in manager.projects:
            return jsonify({"error": "Project not found"}), 404
            
        # Implementar lógica de simulación aquí
        return jsonify({"status": "Simulation completed", "results": {}})

@app.route('/api/project/<string:project_id>/export', methods=['GET'])
@auth.login_required
def export_project(project_id):
    with manager.lock:
        if project_id not in manager.projects:
            return jsonify({"error": "Project not found"}), 404
            
        return jsonify(manager.projects[project_id])

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, threaded=True)
```

**Para usar este sistema remoto:**

1. **Requisitos:**
```bash
pip install flask flask-httpauth
```

2. **Ejecutar el servidor:**
```bash
python cade_server.py
```

3. **Ejemplos de uso desde cliente (usando curl):**

- Crear proyecto:
```bash
curl -u admin:cade123 -X POST http://localhost:5000/api/project/mi_proyecto
```

- Añadir componente:
```bash
curl -u admin:cade123 -H "Content-Type: application/json" -X POST \
-d '{"type": "AND", "position": [100,200]}' \
http://localhost:5000/api/project/mi_proyecto/component
```

- Conectar componentes:
```bash
curl -u admin:cade123 -H "Content-Type: application/json" -X POST \
-d '{"source_id": 1, "output_pin": "out", "dest_id": 2, "input_pin": "in"}' \
http://localhost:5000/api/project/mi_proyecto/connect
```

**Características principales:**

1. **Acceso remoto mediante API REST**
2. Autenticación básica
3. Gestión de múltiples proyectos
4. Operaciones atómicas con bloqueos (thread-safe)
5. Formato JSON para todas las comunicaciones
6. Endpoints para:
   - Creación de proyectos
   - Gestión de componentes
   - Conexiones entre elementos
   - Simulación remota
   - Exportación de proyectos

**Para mejorar el sistema remoto podrías:**

1. Añadir una interfaz web con React/Flask
2. Implementar WebSockets para actualizaciones en tiempo real
3. Agregar más tipos de componentes lógicos
4. Incluir almacenamiento persistente (SQLite/MySQL)
5. Mejorar el sistema de autenticación con JWT
6. Añadir logging y monitorización
7. Implementar un sistema de versionado de proyectos

¿Te gustaría que desarrolle alguna de estas mejoras en particular o prefieres ver cómo implementar un cliente gráfico remoto?
