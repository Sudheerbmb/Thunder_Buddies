# *MCQ Generator with Flask and AWS Bedrock*

## *📌 Overview*
This project is a *Flask-based MCQ generator* that extracts text from uploaded *PDF, DOCX, and TXT files, generates **multiple-choice questions (MCQs)* using *AWS Bedrock, and allows users to **download the MCQs as a PDF*.

## *🛠 Tech Stack*
- *Flask* (Python Web Framework)
- *AWS Bedrock* (For text-based MCQ generation)
- *pdfplumber* (Extract text from PDFs)
- *python-docx* (Extract text from DOCX files)
- *FPDF* (Generate PDFs)
- *HTML/CSS/JavaScript* (Frontend)
- *Gunicorn* (Production Deployment)
- *EC2 Instance* (Hosting the Flask app)

---

## *🚀 Step 1: Set Up EC2 Instance & Install Dependencies*
### *🔹 1.1: Connect to Your EC2 Instance*
If you haven't set up your *AWS EC2 instance*, follow these steps:

1. *Launch an EC2 instance* (Amazon Linux 2 or Ubuntu recommended).
2. Connect to the instance via SSH:
   sh
   ssh -i your-key.pem ec2-user@your-ec2-ip
   

### *🔹 1.2: Install Python and Required Dependencies*
sh
sudo yum update -y
sudo yum install python3 -y
sudo yum install gcc python3-devel -y


#### *(Optional) Set Up a Virtual Environment*
sh
sudo pip3 install virtualenv
mkdir myflask-proj
cd myflask-proj
python3 -m venv venv
source venv/bin/activate


### *🔹 1.3: Install Flask and Dependencies*
sh
pip install Flask gunicorn
pip install -r requirements.txt


---

## *📝 Step 2: Project Structure*
Ensure your *Flask project* follows this structure:

myflask-proj/
│── app.py
│── requirements.txt
│── templates/
│   ├── index.html
│   ├── results.html
│── uploads/
│── results/


---

## *📌 Step 3: Create the Flask Application*
### **📝 3.1: Create app.py**
python
import os
import json
import boto3
import pdfplumber
import docx
from fpdf import FPDF
from flask import Flask, render_template, request, send_file
from werkzeug.utils import secure_filename

UPLOAD_FOLDER = "uploads"
RESULTS_FOLDER = "results"
ALLOWED_EXTENSIONS = {'pdf', 'txt', 'docx'}
AWS_REGION = "ap-south-1"
BEDROCK_MODEL_ID = "meta.llama3-8b-instruct-v1:0"

app = Flask(__name__)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER
app.config["RESULTS_FOLDER"] = RESULTS_FOLDER
app.config["ALLOWED_EXTENSIONS"] = ALLOWED_EXTENSIONS

bedrock = boto3.client("bedrock-runtime", region_name=AWS_REGION)

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in app.config["ALLOWED_EXTENSIONS"]

def extract_text_from_file(file_path):
    text = ""
    if file_path.endswith(".pdf"):
        with pdfplumber.open(file_path) as pdf:
            text = " ".join(page.extract_text() for page in pdf.pages if page.extract_text())
    elif file_path.endswith(".docx"):
        doc = docx.Document(file_path)
        text = "\n".join([para.text for para in doc.paragraphs])
    elif file_path.endswith(".txt"):
        with open(file_path, "r", encoding="utf-8") as file:
            text = file.read()
    return text

def generate_mcqs(text, num_mcqs, difficulty, num_options):
    prompt = f"Generate {num_mcqs} multiple-choice questions at {difficulty} difficulty level with {num_options} answer options:\n\n{text}\n\n"
    payload = {"prompt": prompt, "max_gen_len": 512, "temperature": 0.5, "top_p": 0.9}
    response = bedrock.invoke_model(
        modelId=BEDROCK_MODEL_ID,
        body=json.dumps(payload),
        contentType="application/json",
        accept="application/json"
    )
    response_body = json.loads(response["body"].read())
    return response_body.get("generation", "No MCQs generated.")

def create_pdf(mcqs, filename):
    pdf = FPDF()
    pdf.add_page()
    pdf.set_font("Arial", size=12)
    for line in mcqs.split("\n"):
        pdf.cell(0, 10, line, ln=True)
    pdf_path = os.path.join(app.config["RESULTS_FOLDER"], filename)
    pdf.output(pdf_path)
    return pdf_path

@app.route("/")
def home():
    return render_template("index.html")

@app.route("/upload", methods=["POST"])
def upload():
    if "file" not in request.files:
        return "No file uploaded"
    file = request.files["file"]
    if file and allowed_file(file.filename):
        filename = secure_filename(file.filename)
        file_path = os.path.join(app.config["UPLOAD_FOLDER"], filename)
        file.save(file_path)
        text = extract_text_from_file(file_path)
        num_mcqs = int(request.form.get("num_mcqs", 5))
        difficulty = request.form.get("difficulty", "Medium")
        num_options = int(request.form.get("num_options", 4))
        mcqs = generate_mcqs(text, num_mcqs, difficulty, num_options)
        pdf_filename = f"{filename}_mcqs.pdf"
        pdf_path = create_pdf(mcqs, pdf_filename)
        return render_template("results.html", mcqs=mcqs, pdf_filename=pdf_filename)
    return "Invalid file format"

@app.route("/download/<filename>")
def download(filename):
    file_path = os.path.join(app.config["RESULTS_FOLDER"], filename)
    return send_file(file_path, as_attachment=True)

if __name__ == "__main__":
    os.makedirs(UPLOAD_FOLDER, exist_ok=True)
    os.makedirs(RESULTS_FOLDER, exist_ok=True)
    app.run(debug=True, port=6001)


---

## *🚀 Step 4: Run the Flask App*
### *🔹 Start the Application*
sh
python app.py


### *🔹 Run with Gunicorn (Production)*
sh
gunicorn -w 2 -b 0.0.0.0:5000 app:app


---

## *🌍 Step 5: Access the Web App*
1. Open your browser.
2. Go to **[Link for Website](http://13.233.153.193:5000)** to upload a file and generate MCQs.

---

## *🚀 Final Thoughts*
- 🎯 *Fully functional Flask app*
- 🚀 *Deployed on EC2*
- 📜 *Uses AWS Bedrock for MCQ generation*
- 📄 *Generates downloadable PDFs*



