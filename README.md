# การสร้าง Model as a Service ด้วย Flask และ Docker

## 1. วัตถุประสงค์

1. ทำความเข้าใจแนวคิดของการให้บริการโมเดล Machine Learning ในรูปแบบ Model as a Service
2. ฝึกสร้างเว็บแอปพลิเคชันโดยใช้ Flask เพื่อรับข้อมูลและแสดงผลการทำนาย
3. ฝึกใช้ Docker เพื่อสร้างสภาพแวดล้อมที่รันได้เหมือนกันทุกเครื่อง
4. เผยแพร่ผลงานบน GitHub พร้อมเอกสารอธิบายขั้นตอนอย่างครบถ้วน

---

## 2. โครงสร้างโฟลเดอร์ของโปรเจ็ค

```text
ml_service/
├─ app.py
├─ train_model.py
├─ requirements.txt
├─ Dockerfile
├─ .gitignore
├─ README.md
├─ templates/
│  └─ main.html
├─ screenshots/
│  └─ prediction.png
└─ model/
   └─ model.pkl
```

โฟลเดอร์ `model` และไฟล์ `model/model.pkl` จะถูกสร้างอัตโนมัติเมื่อฝึกและบันทึกโมเดลด้วย `train_model.py`

---

## 3. ขั้นตอนการดำเนินงาน และวิธีการ Build & Run

### 3.1 สร้างโฟลเดอร์โครงการ

สร้างโฟลเดอร์สำหรับงานนี้ (เช่น `ml_service`) และเปิดโฟลเดอร์นี้ด้วยโปรแกรม Visual Studio Code

---

### 3.2 สร้างและฝึกโมเดล

ไฟล์: `train_model.py`

```python
from sklearn.datasets import fetch_california_housing
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestRegressor
import joblib, os

data = fetch_california_housing(as_frame=False)
X, y = data.data, data.target

Xtr, Xte, ytr, yte = train_test_split(X, y, test_size=0.2, random_state=42)

model = RandomForestRegressor(n_estimators=100, random_state=42)
model.fit(Xtr, ytr)

os.makedirs("model", exist_ok=True)
joblib.dump(model, "model/model.pkl")
print("บันทึกโมเดลเรียบร้อยที่ model/model.pkl")
```

---

### 3.3 สร้างแอปพลิเคชัน Flask

ไฟล์: `app.py`

```python
from flask import Flask, request, jsonify, render_template
import joblib, numpy as np, os

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
MODEL_PATH = os.path.join(BASE_DIR, "model", "model.pkl")
model = joblib.load(MODEL_PATH)

FEATURES = ["MedInc","HouseAge","AveRooms","AveBedrms","Population","AveOccup","Latitude","Longitude"]

app = Flask(__name__, template_folder="templates")

@app.get("/health")
def health():
    return {"ok": True, "features": FEATURES}

@app.post("/predict")
def predict():
    data = request.get_json(silent=True) or {}
    x = data.get("features")
    if not x or len(x) != len(FEATURES):
        return jsonify(error=f"Expected 8 features: {FEATURES}"), 400
    X = np.array([x], dtype=float)
    yhat = model.predict(X).tolist()[0]
    return jsonify(prediction=yhat)

@app.route("/", methods=["GET", "POST"])
def main():
    if request.method == "GET":
        return render_template("main.html")
    vals = [float(request.form.get(f)) for f in FEATURES]
    pred = model.predict(np.array([vals])).tolist()[0]
    return render_template("main.html", prediction=pred)

if __name__ == "__main__":
    app.run(debug=True)
```

---

### 3.4 สร้างหน้าเว็บสำหรับทดสอบ

ไฟล์: `templates/main.html` — หน้าเว็บแบบฟอร์มรับค่าฟีเจอร์ทั้ง 8 ตัว (`MedInc`, `HouseAge`, `AveRooms`, `AveBedrms`, `Population`, `AveOccup`, `Latitude`, `Longitude`) พร้อมปุ่ม `Use sample values`, `Predict` และ `Reset` สำหรับส่งข้อมูลไปยัง `/predict`

---

### 3.5 ไฟล์ `requirements.txt`

```text
flask==3.1.3
gunicorn==26.2.0
numpy==2.4.6
scikit-learn==1.9.1
joblib==1.6.0
pandas==3.1.1
```

---

### 3.6 ไฟล์ `Dockerfile`

```dockerfile
# Base image ที่ใช้ Python 3.11 แบบขนาดเล็ก
FROM python:3.11-slim

# สร้างโฟลเดอร์ทำงานภายใน container
WORKDIR /app

# คัดลอก requirements.txt และติดตั้ง dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# คัดลอกไฟล์โปรเจกต์ทั้งหมดเข้า container
COPY . .

# สร้างโมเดล (train model) ตอน build image
RUN python train_model.py

# เปิด port 5000 สำหรับ Flask
EXPOSE 5000

# สั่งให้รัน Flask app ผ่าน Gunicorn (production server)
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:5000", "app:app"]
```

---

### 3.7 การสร้างและรัน Docker Container (Build & Run)

เปิด Terminal / PowerShell ในโฟลเดอร์ `ml_service` แล้วรันคำสั่ง:

```bash
docker build -t ds-ml-service .
docker run --rm -p 5000:5000 ds-ml-service
```

> เมื่อรันสำเร็จ จะแสดงข้อความ `Listening at: http://0.0.0.0:5000 (1)`

เปิดเบราว์เซอร์และเข้าตรวจสอบ:

- **หน้าเว็บสำหรับทำนายผล:** `http://127.0.0.1:5000/`
- **ตรวจสอบสถานะ:** `http://127.0.0.1:5000/health`

*(หากต้องการทดสอบแบบไม่ใช้ Docker สามารถรันใน Virtual Environment ได้โดยสร้าง `.venv` ด้วย Python 3.11 ติดตั้ง `pip install -r requirements.txt` รัน `python train_model.py` แล้วรัน `gunicorn -w 2 -b 0.0.0.0:5000 app:app`)*

---

## 4. การตรวจสอบความถูกต้อง

- สร้างและรัน Docker image `ds-ml-service` ได้สำเร็จ (`Listening at: http://0.0.0.0:5000`)
- หน้าเว็บแสดงแบบฟอร์มรับค่าฟีเจอร์และผลการทำนายได้ (`Prediction: 4.029`)
- เรียกใช้งาน API `/predict` ด้วย JSON ได้ เช่น:
  `{"features":[8.3,25.0,6.0,1.0,1200,3.0,34.2,-118.3]}`

ผลลัพธ์ที่ได้อยู่ในรูปแบบ:
`{"prediction": 4.029472199999998}` (ประมาณ `4.03` หรือราว `$403,000` — ทศนิยมตำแหน่งท้าย ๆ อาจต่างกันเล็กน้อยตามเครื่อง)

### ทดสอบด้วย PowerShell (Windows)

```powershell
Invoke-RestMethod -Uri http://127.0.0.1:5000/predict -Method Post -ContentType "application/json" -Body '{"features":[8.3,25.0,6.0,1.0,1200,3.0,34.2,-118.3]}'
```

หรือแสดง HTTP Status และ Header ด้วย `Invoke-WebRequest`:

```powershell
Invoke-WebRequest -Uri "http://127.0.0.1:5000/predict" -Method POST -Body '{"features":[8.3,25.0,6.0,1.0,1200,3.0,34.2,-118.3]}' -ContentType "application/json"
```

### ทดสอบด้วย `curl` (macOS / Linux / Git Bash)

```bash
curl -i -X POST "http://127.0.0.1:5000/predict" \
  -H "Content-Type: application/json" \
  -d '{"features":[8.3,25.0,6.0,1.0,1200,3.0,34.2,-118.3]}'
```

### ภาพหน้าจอของหน้าเว็บที่ทำนายผลสำเร็จ

![ผลการทำนายบนหน้าเว็บ](screenshots/prediction.png)

---

## 5. วิธีส่งงาน

- Push โค้ดทั้งหมด (รวมถึง `Dockerfile` และ `README.md`) ไปยัง GitHub
- ในไฟล์ `README.md` ระบุวิธีการ build และ run
- แนบภาพหน้าจอของหน้าเว็บที่ทำนายผลสำเร็จ
- ส่งลิงก์ GitHub repository ผ่านระบบ [Google Form](https://forms.gle/D8UCryawXbEn4FSe7)

---

## 6. ปัญหาที่พบบ่อยและแนวทางแก้ไข

- ข้อความ `gunicorn: not found` → เพิ่ม `gunicorn` ใน `requirements.txt` แล้ว build ใหม่
- ข้อความ `TemplateNotFound: main.html` → ตรวจสอบว่าไฟล์อยู่ในโฟลเดอร์ `templates/`
- ข้อความ `FileNotFoundError: model.pkl` → ตรวจสอบว่ารัน `train_model.py` แล้ว
- ข้อความ `read-only file system` → ตรวจสอบให้ Drive C มีพื้นที่ว่างอย่างน้อย 5 GB แล้วรีสตาร์ท Docker
- หากพอร์ต `5000` ถูกใช้งาน → ใช้พอร์ตอื่น เช่น `-p 8080:5000`

---

## 7. หมายเหตุเพิ่มเติม

- ใช้โครงสร้างโฟลเดอร์เดียวกันทั้งหมดเพื่อให้ Docker รันได้ถูกต้อง
- หากต้องการทดสอบก่อน `build Docker` ให้สร้าง `virtual environment (python -m venv .venv)` แล้วติดตั้ง dependencies ตาม `requirements.txt`
- เพิ่มไฟล์ `.gitignore` เพื่อไม่ให้ไฟล์ขนาดใหญ่เช่น `model.pkl` และ `.venv` ถูก push ไป GitHub โดยไม่จำเป็น
