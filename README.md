app.py -
------------------------------------------------------------
import os
import gtts
from flask import Flask, render_template, request, send_from_directory, jsonify
from moviepy.editor import AudioFileClip, ImageClip, CompositeVideoClip, vfx, concatenate_videoclips, VideoFileClip
from datetime import datetime
import threading

app = Flask(__name__)

# Variables to track progress and time
progress = 0
start_time = None
end_time = None

# Lock for controlling access to progress variable
progress_lock = threading.Lock()

@app.route("/")
def index():
    return render_template("imagetovideo.html")

@app.route("/process", methods=["POST"])
def process():
    global progress, start_time, end_time

    def update_progress():
        global progress, start_time, end_time
        while progress < 100:
            with progress_lock:
                if start_time and progress > 0:
                    end_time = datetime.now()
                    elapsed_time = (end_time - start_time).total_seconds()
                    estimated_total_time = elapsed_time / (progress / 100)
                    time_left = round(estimated_total_time - elapsed_time, 1)
                else:
                    time_left = "Calculating..."
            time.sleep(1)  # Update every 1 second

    num_images = int(request.form["num_images"])

    # Create a directory to store temporary files
    temp_dir = "temp"
    os.makedirs(temp_dir, exist_ok=True)

    # List to store generated video file paths
    video_paths = []

    # Set the frames per second (fps) for the video
    fps = 24

    # Start the progress update thread
    progress_thread = threading.Thread(target=update_progress)
    progress_thread.daemon = True
    progress_thread.start()

    start_time = datetime.now()  # Start timing the process

    for i in range(1, num_images + 1):
        text = request.form[f"text{i}"]
        audio_text = request.form[f"audio{i}"]
        image = request.files[f"image{i}"]

        # Skip empty image uploads
        if not image:
            continue

        # Save the uploaded image to the temporary directory
        image_path = os.path.join(temp_dir, image.filename)
        image.save(image_path)

        # Create a GTTS speech
        speech = gtts.gTTS(audio_text, lang="en", slow=False, tld="us")
        timestamp = datetime.now().strftime("%Y%m%d%H%M%S")  # Add timestamp
        mp3_file_name = f"new{i}_{timestamp}.mp3"  # Add timestamp to the file name
        speech.save(os.path.join(temp_dir, mp3_file_name))

        # Load the audio and image
        speech1 = AudioFileClip(os.path.join(temp_dir, mp3_file_name))
        image1 = ImageClip(image_path).set_duration(10)

        # Generate video
        video = CompositeVideoClip([image1.set_duration(speech1.duration).set_audio(speech1)]).fx(vfx.speedx, 1.1)
        timestamp = datetime.now().strftime("%Y%m%d%H%M%S")  # Add timestamp
        video_file_name = f"video{i}_{timestamp}.mp4"  # Add timestamp to the file name
        video_path = os.path.join(temp_dir, video_file_name)
        video.write_videofile(video_path, codec='libx264', fps=fps)

        # Store the video path
        video_paths.append(video_path)

        # Clean up temporary files
        os.remove(image_path)
        os.remove(os.path.join(temp_dir, mp3_file_name))

        with progress_lock:
            progress = (i / num_images) * 100

    # Stop timing the process
    end_time = datetime.now()

    # Concatenate all generated videos into one
    combined_video = concatenate_videoclips([VideoFileClip(path) for path in video_paths], method="compose")

    # Provide a link to download the combined video
    output_dir = "output"
    os.makedirs(output_dir, exist_ok=True)
    timestamp = datetime.now().strftime("%Y%m%d%H%M%S")  # Add timestamp
    combined_video_file_name = f"combined_video_{timestamp}.mp4"  # Add timestamp to the file name
    combined_video_path = os.path.join(output_dir, combined_video_file_name)
    combined_video.write_videofile(combined_video_path, codec='libx264', fps=fps)

    return f"Combined video generated. <a href='/download/{combined_video_file_name}' download>Download Combined Video</a>"

@app.route("/download/<filename>")
def download(filename):
    return send_from_directory("output", filename)

@app.route("/progress")
def get_progress():
    global progress
    time_left = update_progress(progress)
    return jsonify({"progress": progress, "time_left": time_left})

if __name__ == "__main__":
    app.run(debug=True)
_________________________________________________
html file -
<!DOCTYPE html>
<html>
<head>
    <title>Image to Video Converter</title>
    <style>
        .image-input {
            display: none;
        }

        .remove-image {
            background-color: #f44336;
            color: white;
            padding: 5px 10px;
            border: none;
            cursor: pointer;
            margin-left: 10px;
        }

        .clear-text {
            background-color: #2196F3;
            color: white;
            padding: 5px 10px;
            border: none;
            cursor: pointer;
            margin-left: 10px;
        }

        .progress-container {
            display: none;
        }

        .progress-bar {
            width: 0;
            background-color: #4CAF50;
            height: 30px;
        }
        
        .text-input-container {
            display: flex;
            justify-content: space-between;
            align-items: center;
        }

        .text-input {
            width: 45%;
            padding: 5px;
        }
    </style>
</head>
<body>
    <h1>Image to Video Converter</h1>
    <form action="/process" method="post" enctype="multipart/form-data">
        <label for="num_images">Number of Images (1-35):</label>
        <input type="number" name="num_images" id="num_images" min="1" max="35" required><br>
        {% for i in range(1, 36) %}
        <label for="image{{ i }}">Select an image (JPG or PNG) for Image {{ i }}:</label>
        <label id="image-label{{ i }}"></label>
        <input type="file" name="image{{ i }}" id="image{{ i }}" class="image-input" accept=".jpg, .png">
        <button type="button" class="choose-file" onclick="document.getElementById('image{{ i }}').click()">Choose File</button><br>
        <button type="button" class="remove-image" onclick="removeImage({{ i }})">Remove Image</button>
        <div class="text-input-container">
            <label for="text{{ i }}">Enter text for Image {{ i }}:</label>
            <textarea name="text{{ i }}" id="text{{ i }}" rows="4" cols="50" class="text-input"></textarea>
            <label for="audio{{ i }}">Text to Speech for Image {{ i }}:</label>
            <textarea name="audio{{ i }}" id="audio{{ i }}" rows="4" cols="50" class="text-input"></textarea>
        </div>
        <br>
        <button type="button" class="clear-text" onclick="clearText({{ i }})">Clear Text</button><br>
        <img id="output-image{{ i }}" style="display: none;">
        {% endfor %}
        <input type="submit" value="Convert to Video">
    </form>
    
    <div class="progress-container">
        <div class="progress-bar" id="progress-bar"></div>
        <p>Processing...</p>
        <p id="time-left">Time left: Calculating...</p>
    </div>
    
    <script>
        function removeImage(imageNumber) {
            const imageInput = document.getElementById("image" + imageNumber);
            imageInput.value = null;
            document.getElementById("image-label" + imageNumber).textContent = "";
            document.getElementById("output-image" + imageNumber).style.display = "none";
        }

        function clearText(textNumber) {
            const textArea = document.getElementById("text" + textNumber);
            textArea.value = "";
            document.getElementById("output-image" + textNumber).style.display = "none";
        }

        // Display the uploaded image's name and the text on the image
        document.querySelectorAll('.image-input').forEach((input, index) => {
            input.addEventListener('change', (event) => {
                const label = document.getElementById('image-label' + (index + 1));
                if (event.target.files.length > 0) {
                    label.textContent = 'Uploaded: ' + event.target.files[0].name;
                    const text = document.getElementById("text" + (index + 1)).value;
                    if (text) {
                        // Generate an updated image with text
                        const outputImage = document.getElementById("output-image" + (index + 1));
                        outputImage.src = URL.createObjectURL(event.target.files[0]);
                        outputImage.alt = text;
                        outputImage.style.display = "block";
                    }
                } else {
                    label.textContent = '';
                    document.getElementById("output-image" + (index + 1)).style.display = "none";
                }
            });
        });
        
        // Function to update progress
        function updateProgress() {
            fetch("/progress")
                .then(response => response.json())
                .then(data => {
                    document.querySelector(".progress-bar").style.width = data.progress + "%";
                    document.getElementById("time-left").textContent = "Time left: " + data.time_left;
                    if (data.progress < 100) {
                        setTimeout(updateProgress, 1000);  // Update every 1 second
                    }
                });
        }

        // Start updating progress when the form is submitted
        document.querySelector("form").addEventListener("submit", function () {
            document.querySelector(".progress-container").style.display = "block";
            setTimeout(updateProgress, 1000);  // Start updating progress after 1 second
        });
    </script>
</body>
</html>
-----------------------------------------------------------------------------------
