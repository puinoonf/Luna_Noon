#  Noon Transaction — Fraud Detection System
ระบบตรวจสอบความผิดปกติของธุรกรรมทางการเงิน โดยใช้ **Neo4j Graph Database** บน **Docker** พัฒนาด้วย Python Flask และ HTML Dashboard
> โปรเจ็กต์วิชา NoSQL Database | คณะวิทยาศาสตร์ สาขาวิทยาการคอมพิวเตอร์ มหาวิทยาลัยศรีนครินทรวิโรฒ

##  โครงสร้างโปรเจ็กต์
noon-transaction/
├── docker-compose.yml      # กำหนดค่า Neo4j Container
├── noontrain.py            # สร้างข้อมูลจำลอง (CSV Generator)
├── accounts_v2.csv         # ข้อมูลบัญชี 900 บัญชี
├── transactions_v2.csv     # ข้อมูลธุรกรรม 2,218 รายการ
├── noon.py                 # นำเข้าข้อมูลและรัน CRUD Demo
├── api_server.py           # Flask REST API Backend
└── dashboard.html          # Frontend Dashboard (Dark Theme)
##  วิธีติดตั้งและรันระบบ

### ขั้นตอนที่ 1 — เริ่ม Neo4j บน Docker
```bash
docker-compose up -d
```
docker ps
docker logs Noon_transaction_neo4j
```
| Service | Port | URL |
|---|---|---|
| Neo4j Browser | 7475 | http://localhost:7475 |
| Bolt Protocol | 7688 | bolt://localhost:7688 |

### ขั้นตอนที่ 2 — สร้างข้อมูลจำลอง (ถ้ายังไม่มี CSV)
```bash
python noontrain.py
```
จะสร้างไฟล์ `accounts_v2.csv` และ `transactions_v2.csv`


### ขั้นตอนที่ 3 — นำเข้าข้อมูลสู่ Neo4j
```bash
pip install neo4j tabulate --break-system-packages
python noon.py
```
- นำเข้า **900 บัญชี** และ **2,217 ธุรกรรม** เข้า Neo4j
- รัน CRUD Demo ครบทุกคำสั่ง
- แสดงผล Fraud Pattern Analysis

### ขั้นตอนที่ 4 — เริ่ม API Server
```bash
pip install flask flask-cors neo4j --break-system-packages
python api_server.py
```
API พร้อมใช้งานที่ `http://localhost:5000`

### ขั้นตอนที่ 5 — เปิด Dashboard
เปิดไฟล์ `dashboard.html` ในเบราว์เซอร์:
```bash
# macOS
open dashboard.html
# Linux
xdg-open dashboard.html
# หรือลาก dashboard.html ใส่ในเบราว์เซอร์โดยตรง

##  โครงสร้าง Graph Database
```
(Person)-[:OWNS]->(Account)-[:SENT]->(Transaction)-[:RECEIVED_BY]->(Account)
```
| Node / Relationship | Properties |
|---|---|
| `Person` | name |
| `Account` | id, bank, province, balance, risk_score, status, tier |
| `Transaction` | id, amount, currency, type, timestamp, ground_truth, flagged, flag_rule |
| `OWNS` | — |
| `SENT` | amount, timestamp |
| `RECEIVED_BY` | amount, timestamp |
---

## 🔧 CRUD Operations (Cypher)
### CREATE — สร้างข้อมูล
```cypher
// สร้างบัญชีและเจ้าของ
MERGE (p:Person {name: "สมชาย ใจดี"})
MERGE (a:Account {id: "ACC0001"})
SET a.bank = "SCB", a.balance = 100000, a.risk_score = 0.1
MERGE (p)-[:OWNS]->(a)
// สร้างธุรกรรม
CREATE (t:Transaction {id: "TX-T12345", amount: 50000, timestamp: datetime()})
CREATE (from)-[:SENT {amount: 50000}]->(t)
CREATE (t)-[:RECEIVED_BY {amount: 50000}]->(to)


### READ — อ่านข้อมูล
```cypher
// ดูธุรกรรมต้องสงสัย
MATCH (from:Account)-[:SENT]->(t:Transaction)-[:RECEIVED_BY]->(to:Account)
WHERE t.amount >= 400000 OR t.ground_truth = true
MATCH (fp:Person)-[:OWNS]->(from)
RETURN fp.name, from.id, t.amount, t.ground_truth
ORDER BY t.amount DESC LIMIT 10

### UPDATE — แก้ไขข้อมูล
```cypher
// อายัดบัญชี
MATCH (a:Account {id: "ACC0001"})
SET a.status = "frozen", a.risk_score = 1.0
// Flag ธุรกรรมผิดปกติ
MATCH (t:Transaction)
WHERE t.ground_truth = true
SET t.flagged = true, t.flag_rule = "known_fraud"

### DELETE — ลบข้อมูล
```cypher
// ลบธุรกรรม (พร้อม Relationship)
MATCH (t:Transaction {id: "TX-T12345"})
DETACH DELETE t
// Reset flags ทั้งหมด
MATCH (t:Transaction)
SET t.flagged = false, t.flag_rule = ""

##  API Endpoints
```
| Method | Endpoint | คำอธิบาย |
|---|---|---|
| GET | `/api/stats` | สถิติภาพรวมระบบ |
| GET | `/api/accounts` | รายชื่อบัญชีเรียงตาม Risk Score |
| GET | `/api/suspicious` | ธุรกรรมต้องสงสัย |
| GET | `/api/fraud-patterns` | 4 รูปแบบ Fraud Pattern |
| GET | `/api/risk-distribution` | การกระจาย Risk Score |
| GET | `/api/precision-recall` | ประสิทธิภาพการตรวจจับ |
| GET | `/api/velocity` | บัญชีที่มีธุรกรรมถี่ผิดปกติ |
| GET | `/api/cross-province` | ธุรกรรมข้ามจังหวัดสูงผิดปกติ |
| GET | `/api/thresholds` | ค่า Threshold แบบ Dynamic |
| POST | `/api/apply-rules` | รันกฎตรวจจับ Fraud |
| POST | `/api/transfer` | สร้างธุรกรรมใหม่ |
| POST | `/api/freeze` | อายัดบัญชี |

##  Fraud Patterns ที่ตรวจจับได้
```
| รูปแบบ | คำอธิบาย | Cypher Pattern |
|---|---|---|
| **Circular Flow** | โอนเงินเป็นวงจร A→B→C→A | 3-hop cycle detection |
| **Money Mule** | บัญชีรับเงินจากหลายแหล่งแล้วส่งต่อ | in≥2, out≥2 |
| **Smurfing** | แตกย่อยธุรกรรมหลายครั้งใกล้เคียงกัน | tx≥4, std_dev ต่ำ |
| **Shell Company** | รับเงินเยอะแต่โอนออกเกือบทั้งหมด | passthrough ≥ 80% |

##  docker-compose.yml
```yaml
version: '3.8'
services:
  neo4j:
    image: neo4j:5.15-community
    container_name: Noon_transaction_neo4j
    ports:
      - "7475:7474"   # HTTP Browser
      - "7688:7687"   # Bolt Protocol
    environment:
      NEO4J_AUTH: neo4j/noon2024!
      NEO4J_PLUGINS: '["apoc"]'
      NEO4J_dbms_memory_heap_max__size: "2G"
    volumes:
      - neo4j_data:/data
      - neo4j_plugins:/plugins
    networks:
      - fraud_network

##  ข้อมูลจำลอง
```
| รายการ | จำนวน |
|---|---|
| บัญชี (Accounts) | 900 บัญชี |
| ธุรกรรม (Transactions) | 2,217 รายการ |
| ธนาคาร | 10 แห่ง (SCB, KBANK, BBL, KTB, TTB, BAY, GSB, BAAC, CIMBT, UOB) |
| จังหวัด | ครอบคลุมทุกจังหวัดในไทย |
| Fraud Rate | ~0.5% (Circular, Smurfing, Mule, Shell) |

##  ผู้จัดทำ
```
| ชื่อ | รหัสนักศึกษา |
|---|---|
| นางสาวฑิฆัมพร ท้วมแก้ว | 67102010161 |
| นายอามีน สาแม | 67102010532 |

**อาจารย์ที่ปรึกษา:** ผศ.ดร.วราภรณ์ วิยานนท์  
**คณะวิทยาศาสตร์ สาขาวิทยาการคอมพิวเตอร์ มหาวิทยาลัยศรีนครินทรวิโรฒ | ปีการศึกษา 2569**
