# SKILL Queue – Message Queue Middleware

Universitäres Forschungsprojekt im Rahmen von SKILL/VL  
Frankfurt University of Applied Sciences, WS 2023/24

**Zeitraum**: Oktober 2023 – April 2024  
**Technischer Beitrag**: Konzeption, Implementierung und Integration einer priorisierten Message Queue als Middleware in Python (Celery + RabbitMQ) in eine bestehende Proxmox VE-Infrastruktur

### Projekt-Overview

Entwicklung einer skalierbaren Message Queue zur asynchronen, entkoppelten Kommunikation zwischen virtualisierten Komponenten (Lernumgebungen auf Proxmox-Basis).  

**Kernprobleme ohne Queue**:
- Synchrone Kopplung → Ausfälle bei Modulfehlern
- Begrenzte Skalierbarkeit bei steigender Anzahl virtualisierter Räume
- Schwierige Fehlerisolierung und Nachverfolgbarkeit

**Lösung**: Asynchrone Task-Queue mit Priorisierung, Retry-Mechanismen und Fehlerbehandlung.

### Technische Umgebung

**Software Stack**
- Python 3.x
- Celery (Task Queue / Distributed Task Processing)
- RabbitMQ (Message Broker)
- Proxmox VE (Hypervisor für KVM/QEMU-VMs)
- Ceph (verteilter Speicher)
- FastAPI (REST-Endpunkte für Task-Trigger)
- Pydantic (Datenvalidierung & Serialisierung)
- SQLAlchemy / asynchrone DB-Session (Datenbank-Integration)
- asyncio (asynchrone I/O)

**Hardware/Infrastruktur**
- Proxmox-Cluster (mehrere Nodes)
- Hyperkonvergentes Setup (Compute + Storage kombiniert)

**Nicht-funktionale Anforderungen** (Kernpunkte)
- Skalierbarkeit: ≥ 10 Proxmox-Nodes
- Hohe Verfügbarkeit & niedrige Latenz bei Prioritätsnachrichten
- Robuste Fehlerbehandlung (Timeout, Retry, Dead Letter Queue)

### Code & Implementierung – Wichtige Auszüge

**1. Celery Task (create_template_task)**  
Asynchrone Aufgabe zur Erstellung virtueller Raumvorlagen:

```python
@celery_app.task
def create_template_task(virtualroomtemplate, user):
    db = refreshDB()
    try:
        if isinstance(virtualroomtemplate, dict):
            virtualroomtemplate = VirtualRoomCreate.parse_obj(virtualroomtemplate)
        if isinstance(user, dict):
            user = UserCreate.parse_obj(user)
        
        result = loop.run_until_complete(
            database_middleware.create_virtualroom_template(
                template=virtualroomtemplate,
                user=user,
                db=db
            )
        )
        template = VirtualRoom.parse_obj(result)
        return template
    finally:
        db.close()
```
- Validierung mit Pydantic
- Asynchrone DB-Operation via asyncio
- Automatisches Schließen der DB-Connection

2. **FastAPI-ähnlicher Endpunkt (create_template)**
Trigger der asynchronen Celery-Task über HTTP:
```Python
@router.post('/template', ...)
async def create_template(
    virtualroomtemplate: VirtualRoomCreate,
    user: Annotated[UserCreate, Depends(uma_permission_virtualroomtemplate_create)],
    db: DBSessionDependency
) -> VirtualRoom:
    try:
        virtualroomtemplate = jsonable_encoder(virtualroomtemplate, exclude_none=True)
        user = jsonable_encoder(user, exclude_none=True)
        
        result = tasks.create_template_task.apply_async(
            args=[virtualroomtemplate, user],
            queue='high'
        )
        result = result.get()
        return result
    except (keycloak.ResourceCreationException, 
            ProtectionAPITokenException, 
            database_middleware.ProxmoxCreationException) as e:
        raise HTTPException(status_code=400, detail=f"Resource creation failed: {e}")
```
- Asynchrone Ausführung via Celery + RabbitMQ
- Prioritäts-Queue ('high')
- Fehlerbehandlung mit HTTP-Exceptions

3. **Celery-Konfiguration (app.py)**
Zentrale Konfiguration der Celery-Anwendung:
```Python
from celery import Celery
from kombu import Queue

celery_app = Celery('skill_queue')

celery_app.config_from_object({
    'broker_url': 'pyamqp://guest:guest@localhost//',
    'result_backend': 'db+sqlite:///results.sqlite3',
})

CELERY_DEFAULT_QUEUE = 'default'

CELERY_QUEUES = (
    Queue('default'),
    Queue('medium'),
    Queue('high'),
)

from tasks import *
celery_app.autodiscover_tasks(['tasks'])
```
- RabbitMQ als Broker (```pyamqp```)
- SQLite als Ergebnis-Backend
- Mehrere Queues für Priorisierung (```default```, ```medium```, ```high```)
- Automatische Task-Entdeckung aus tasks-Modul
