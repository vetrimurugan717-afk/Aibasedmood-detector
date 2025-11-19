# Aibasedmood-detector
from flask import Flask, render_template, request, jsonify
import librosa
import numpy as np
import os
import random
from werkzeug.utils import secure_filename

app = Flask(_name_)
UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER

MOODS = ["Happy", "Sad", "Energetic", "Calm"]

def filename_heuristic(name: str):
    """Detect mood based on filename keywords."""
    n = name.lower()
    if any(k in n for k in ["happy", "joy", "smile", "dance", "party"]):
        return "Happy"
    elif any(k in n for k in ["sad", "lonely", "blue", "cry", "tear"]):
        return "Sad"
    elif any(k in n for k in ["energetic", "fast", "rock", "run", "beat"]):
        return "Energetic"
    elif any(k in n for k in ["calm", "chill", "relax", "sleep", "soft", "lofi"]):
        return "Calm"
    return None


def bpm_to_mood(bpm: float):
    """Map BPM value to a mood category."""
    if bpm >= 120:
        return "Energetic"
    elif bpm >= 95:
        return "Happy"
    elif bpm >= 70:
        return "Calm"
    else:
        return "Sad"


def analyze_audio(file_path: str, filename: str):
    """Main mood analysis function combining heuristic + BPM analysis."""
    mood = filename_heuristic(filename)
    detected_by = "Filename Heuristic"
    confidence = "High"

    if mood is None:
        try:
            y, sr = librosa.load(file_path)
            tempo, _ = librosa.beat.beat_track(y=y, sr=sr)
            bpm = round(tempo, 2)
            mood = bpm_to_mood(bpm)
            detected_by = f"BPM {bpm}"
            confidence = "Medium" if bpm else "Low"
        except Exception as e:
            print("Error analyzing audio:", e)
            mood = random.choice(MOODS)
            detected_by = "Random Fallback"
            confidence = "Low"

    return {
        "filename": filename,
        "mood": mood,
        "detected_by": detected_by,
        "confidence": confidence,
    }


@app.route("/")
def index():
    """Serve the frontend page."""
    return render_template("index.html")  # put the HTML file in /templates folder


@app.route("/analyze", methods=["POST"])
def analyze():
    """Receive audio file and return mood result."""
    if "file" not in request.files:
        return jsonify({"error": "No file uploaded"}), 400

    file = request.files["file"]
    if file.filename == "":
        return jsonify({"error": "Empty filename"}), 400

    filename = secure_filename(file.filename)
    file_path = os.path.join(app.config["UPLOAD_FOLDER"], filename)
    file.save(file_path)

    result = analyze_audio(file_path, filename)

    # clean up file
    try:
        os.remove(file_path)
    except:
        pass

    return jsonify(result)


if _name_ == "_main_":
    app.run(debug=True)
