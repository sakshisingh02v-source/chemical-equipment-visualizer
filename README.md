# Chemical Equipment Visualizer

This project is a web and desktop application for visualizing chemical equipment data from CSV files. It consists of a Django backend with REST API, a React web frontend, and a PyQt desktop frontend.

## Architecture

```
chemical-equipment-visualizer/
│
├── backend/                 # Django backend and analytics API
│   ├── manage.py
│   ├── requirements.txt
│   ├── backend/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   └── analytics/
│       ├── __init__.py
│       ├── models.py
│       ├── views.py
│       ├── serializers.py
│       ├── urls.py
│       └── utils.py
│
├── web-frontend/            # React web app
│   ├── package.json
│   └── src/
│       ├── App.js
│       ├── api.js
│       └── components/
│           ├── Upload.js
│           ├── Summary.js
│           └── Charts.js
│
├── desktop-frontend/        # PyQt desktop app
│   ├── main.py
│   └── requirements.txt
│
├── sample_equipment_data.csv
└── README.md
```

---

## Backend (Django/DRF)

- REST API for file uploads and analytics history.
- On CSV upload: parses file, computes summary statistics, stores up to 5 recent analyses.

Install requirements:
```
cd backend
pip install -r requirements.txt
```

**Key dependencies:**  
- django  
- djangorestframework  
- pandas  
- reportlab

### `analytics/models.py`

```python
from django.db import models

class Dataset(models.Model):
    filename = models.CharField(max_length=200)
    summary = models.JSONField()
    uploaded_at = models.DateTimeField(auto_now_add=True)
```

### `analytics/utils.py`

```python
import pandas as pd

def analyze_csv(file):
    df = pd.read_csv(file)
    return {
        "total_equipment": len(df),
        "avg_flowrate": df["Flowrate"].mean(),
        "avg_pressure": df["Pressure"].mean(),
        "avg_temperature": df["Temperature"].mean(),
        "type_distribution": df["Type"].value_counts().to_dict()
    }
```

### `analytics/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from .models import Dataset
from .utils import analyze_csv

class UploadCSV(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        file = request.FILES['file']
        summary = analyze_csv(file)

        Dataset.objects.create(filename=file.name, summary=summary)

        # Keep only 5 most recent
        if Dataset.objects.count() > 5:
            Dataset.objects.order_by('uploaded_at').first().delete()

        return Response(summary)

class History(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        data = Dataset.objects.order_by('-uploaded_at')[:5]
        return Response([d.summary for d in data])
```

### `analytics/urls.py`

```python
from django.urls import path
from .views import UploadCSV, History

urlpatterns = [
    path('upload/', UploadCSV.as_view()),
    path('history/', History.as_view()),
]
```

### `backend/urls.py`

```python
from django.urls import path, include

urlpatterns = [
    path('api/', include('analytics.urls')),
]
```

---

## Web Frontend (React)

Installs:  
```
cd web-frontend
npm install
```

**Key dependencies:**  
- react
- chart.js

### `api.js`

```js
export const uploadCSV = async (file, token) => {
  const formData = new FormData();
  formData.append("file", file);

  const res = await fetch("http://localhost:8000/api/upload/", {
    method: "POST",
    headers: { Authorization: `Token ${token}` },
    body: formData
  });

  return res.json();
};
```

### `App.js`

```js
import Upload from "./components/Upload";

function App() {
  return (
    <div>
      <h2>Chemical Equipment Visualizer</h2>
      <Upload />
    </div>
  );
}

export default App;
```

### `components/Upload.js`

```js
import { uploadCSV } from "../api";

function Upload() {
  const handleUpload = async (e) => {
    const file = e.target.files[0];
    const data = await uploadCSV(file, "YOUR_TOKEN");
    console.log(data);
  };

  return <input type="file" onChange={handleUpload} />;
}

export default Upload;
```

---

## Desktop Frontend (PyQt5)

Install requirements:
```
cd desktop-frontend
pip install -r requirements.txt
```

**Key dependencies:**  
- pyqt5
- requests
- matplotlib

### `main.py`

```python
import sys, requests
from PyQt5.QtWidgets import QApplication, QWidget, QPushButton, QFileDialog
import matplotlib.pyplot as plt

class App(QWidget):
    def __init__(self):
        super().__init__()
        self.btn = QPushButton("Upload CSV", self)
        self.btn.clicked.connect(self.upload)
        self.setWindowTitle("Chemical Equipment Visualizer")
        self.show()

    def upload(self):
        path, _ = QFileDialog.getOpenFileName(self, "Open CSV")
        files = {'file': open(path, 'rb')}
        headers = {'Authorization': 'Token YOUR_TOKEN'}

        res = requests.post("http://localhost:8000/api/upload/", files=files, headers=headers)
        data = res.json()

        plt.bar(data["type_distribution"].keys(), data["type_distribution"].values())
        plt.show()

app = QApplication(sys.argv)
window = App()
sys.exit(app.exec_())
```

---

## Sample Data

A sample CSV (`sample_equipment_data.csv`) should be available in the root for testing/demo.

## Usage

1. **Start backend:**  
   ```
   python manage.py runserver
   ```
2. **Run web frontend:**  
   ```
   npm start
   ```
3. **Run desktop frontend:**  
   ```
   python main.py
   ```
4. **Upload CSV files:**  
   Use the web or desktop UI, authenticate via Token, view summary and charts.

## Notes

- Default API assumes API token auth (update `"YOUR_TOKEN"` in examples).
- Only the 5 most recent dataset summaries are stored.
- Update `CORS`/auth settings for production usage as needed.

---