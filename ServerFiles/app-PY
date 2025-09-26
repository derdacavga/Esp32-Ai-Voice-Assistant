from flask import Flask, request, jsonify, Response, send_file
import torch
from transformers import pipeline, AutoTokenizer, AutoModelForSeq2SeqLM
import os
import logging
import io
import numpy as np
import scipy.io.wavfile as wavfile
import soundfile as sf
from pydub import AudioSegment
import time
from functools import lru_cache
import gc
import psutil
import threading
import time
from queue import Queue
import uuid
import subprocess
import tempfile
import atexit

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

IS_HF_SPACE = os.environ.get('SPACE_ID') is not None
HF_TOKEN = os.environ.get('HF_TOKEN')

if IS_HF_SPACE: 
    device = "cpu"
    torch.set_num_threads(2)
    os.environ['TOKENIZERS_PARALLELISM'] = 'false'
    logger.info("Running on Hugging Face Spaces - CPU optimized mode")
else:
    device = "cuda" if torch.cuda.is_available() else "cpu"
    torch.set_num_threads(4)

logger.info(f"Using device: {device}")

app = Flask(__name__)
app.config['TEMP_AUDIO_DIR'] = '/tmp/audio_responses'
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024

stt_pipeline = None
llm_model = None
llm_tokenizer = None
tts_pipeline = None
tts_type = None

active_files = {}
file_cleanup_lock = threading.Lock()
cleanup_thread = None

def cleanup_old_files():
    while True:
        try:
            with file_cleanup_lock:
                current_time = time.time()
                files_to_remove = []
                
                for file_id, file_info in list(active_files.items()):
                    if current_time - file_info['created_time'] > 300:
                        files_to_remove.append(file_id)
                
                for file_id in files_to_remove:
                    try:
                        if os.path.exists(active_files[file_id]['filepath']):
                            os.remove(active_files[file_id]['filepath'])
                        del active_files[file_id]
                        logger.info(f"Cleaned up file: {file_id}")
                    except Exception as e:
                        logger.warning(f"Cleanup error for {file_id}: {e}")
        except Exception as e:
            logger.error(f"Cleanup thread error: {e}")
        
        time.sleep(60)

def start_cleanup_thread():
    global cleanup_thread
    if cleanup_thread is None or not cleanup_thread.is_alive():
        cleanup_thread = threading.Thread(target=cleanup_old_files, daemon=True)
        cleanup_thread.start()
        logger.info("Cleanup thread started")

def cleanup_all_files():
    try:
        with file_cleanup_lock:
            for file_id, file_info in active_files.items():
                try:
                    if os.path.exists(file_info['filepath']):
                        os.remove(file_info['filepath'])
                except:
                    pass
            active_files.clear()
        
        if os.path.exists(app.config['TEMP_AUDIO_DIR']):
            import shutil
            shutil.rmtree(app.config['TEMP_AUDIO_DIR'], ignore_errors=True)
        
        logger.info("All temporary files cleaned up")
    except Exception as e:
        logger.warning(f"Final cleanup error: {e}")

atexit.register(cleanup_all_files)

def get_memory_usage(): 
    try:
        process = psutil.Process(os.getpid())
        memory_info = process.memory_info()
        return {
            "rss_mb": memory_info.rss / 1024 / 1024,
            "vms_mb": memory_info.vms / 1024 / 1024,
            "available_mb": psutil.virtual_memory().available / 1024 / 1024,
            "percent": psutil.virtual_memory().percent
        }
    except Exception as e:
        logger.warning(f"Memory info error: {e}")
        return {"rss_mb": 0, "vms_mb": 0, "available_mb": 0, "percent": 0}

def initialize_models():  
    global stt_pipeline, llm_model, llm_tokenizer, tts_pipeline, tts_type
    
    try:
        logger.info(f"Initial memory usage: {get_memory_usage()}")
         
        if stt_pipeline is None:
            logger.info("Loading Whisper-tiny STT model...")
            try:
                stt_pipeline = pipeline(
                    "automatic-speech-recognition", 
                    model="openai/whisper-tiny",
                    device=device,
                    torch_dtype=torch.float16 if device == "cuda" else torch.float32,
                    token=HF_TOKEN,
                    return_timestamps=False
                )
                logger.info("✅ STT model loaded successfully")
            except Exception as e:
                logger.error(f"STT loading failed: {e}")
                raise
            
            gc.collect()
            logger.info(f"STT loaded. Memory: {get_memory_usage()}")
 
        if llm_model is None:
            logger.info("Loading DialoGPT-small LLM...")
            try:
                model_name = "google/flan-t5-base"
                
                llm_tokenizer = AutoTokenizer.from_pretrained(
                    model_name,
                    token=HF_TOKEN,
                    trust_remote_code=True
                )
                
                llm_model = AutoModelForSeq2SeqLM.from_pretrained(
                    model_name,
                    torch_dtype=torch.float16 if device == "cuda" else torch.float32,
                    token=HF_TOKEN,
                    trust_remote_code=True
                ).to(device)
                
                if llm_tokenizer.pad_token is None:
                    llm_tokenizer.pad_token = llm_tokenizer.eos_token
                
                logger.info("✅ LLM model loaded successfully")
            except Exception as e:
                logger.error(f"LLM loading failed: {e}")
                raise
                
            gc.collect()
            logger.info(f"LLM loaded. Memory: {get_memory_usage()}")
             
        if tts_pipeline is None:
            logger.info("Loading TTS model...")
            tts_loaded = False
            
            try:
                from gtts import gTTS
                tts_pipeline = "gtts"
                tts_type = "gtts"
                tts_loaded = True
                logger.info("✅ Using gTTS (Google Text-to-Speech)")
            except ImportError:
                logger.warning("gTTS not available")
            
            if not tts_loaded:
                tts_pipeline = "silent"
                tts_type = "silent"
                logger.warning("Using silent fallback for TTS")
                
            gc.collect()
            logger.info(f"TTS loaded. Memory: {get_memory_usage()}")
            
        logger.info("🎉 All models loaded successfully!")
        start_cleanup_thread()
        
    except Exception as e:
        logger.error(f"❌ Model loading error: {e}")
        logger.error(f"Memory usage at error: {get_memory_usage()}")
        raise e

@lru_cache(maxsize=32)
def cached_generate_response(text_hash, text): 
    return generate_llm_response(text)

def generate_llm_response(text):
    try: 
        if len(text) > 200:
            text = text[:200]

        if not text.strip():
            return "I'm listening. How can I help you?"
 
        inputs = llm_tokenizer(
            text,
            return_tensors="pt",
            truncation=True,
            padding=True,
            max_length=512
        )
        input_ids = inputs["input_ids"].to(device)
        attention_mask = inputs.get("attention_mask")
        if attention_mask is not None:
            attention_mask = attention_mask.to(device)
 
        with torch.no_grad():
            is_seq2seq = getattr(getattr(llm_model, "config", {}), "is_encoder_decoder", False)

            gen_kwargs = dict(
                max_new_tokens=50,
                do_sample=True,
                temperature=0.7,
                top_k=50,
                top_p=0.9,
                no_repeat_ngram_size=2,
                early_stopping=True,
                pad_token_id=llm_tokenizer.eos_token_id if llm_tokenizer.pad_token_id is None else llm_tokenizer.pad_token_id,
                use_cache=True
            )

            if is_seq2seq:
                outputs_ids = llm_model.generate(
                    input_ids=input_ids,
                    attention_mask=attention_mask,
                    **gen_kwargs
                )
            else: 
                outputs_ids = llm_model.generate(
                    input_ids=input_ids,
                    **gen_kwargs
                )
 
        response = llm_tokenizer.decode(outputs_ids[0], skip_special_tokens=True)
 
        del inputs, input_ids, attention_mask, outputs_ids
        gc.collect()
        if device == "cuda":
            torch.cuda.empty_cache()

        response = response.strip()
        if not response or len(response) < 3:
            return "I understand. What else would you like to know?"

        return response

    except Exception as e:
        logger.error(f"LLM generation error: {e}", exc_info=True)
        return "I'm having trouble processing that. Could you try again?"


def preprocess_audio_optimized(audio_bytes):  
    try:
        logger.info(f"Processing audio: {len(audio_bytes)} bytes")
        
        if len(audio_bytes) > 44 and audio_bytes[:4] == b'RIFF':
            audio_bytes = audio_bytes[44:]  # WAV header'ı atla
            logger.info("WAV header removed")
        
        audio_data = np.frombuffer(audio_bytes, dtype=np.int16).astype(np.float32) / 32768.0
        
        max_samples = 30 * 16000
        if len(audio_data) > max_samples:
            audio_data = audio_data[:max_samples]
            logger.info("Audio trimmed to 30 seconds")
        
        min_samples = int(0.5 * 16000)
        if len(audio_data) < min_samples:
            logger.warning(f"Audio too short: {len(audio_data)/16000:.2f} seconds")
            return None, None
        
        logger.info(f"Audio processed: {len(audio_data)/16000:.2f} seconds")
        return 16000, audio_data
        
    except Exception as e:
        logger.error(f"Audio preprocessing error: {e}")
        raise e

def generate_tts_audio(text): 
    try:
        text = text.replace('\n', ' ').strip()
        
        if len(text) > 200:
            text = text[:200] + "..."
            
        if not text:
            text = "I understand."
        
        logger.info(f"TTS generating: '{text[:50]}...'")
        
        if tts_type == "gtts":
            from gtts import gTTS
            
            with tempfile.NamedTemporaryFile(suffix='.mp3', delete=False) as tmp_file:
                try:
                    tts = gTTS(text=text, lang='en', slow=False)
                    tts.save(tmp_file.name)
                    
                    from pydub import AudioSegment
                    audio_segment = AudioSegment.from_file(tmp_file.name, format="mp3")
                    audio_segment = audio_segment.set_frame_rate(16000).set_channels(1)  # Mono 16kHz
                    wav_buffer = io.BytesIO()
                    audio_segment.export(wav_buffer, format="wav")
                    wav_data = wav_buffer.getvalue()
                
                    os.unlink(tmp_file.name)
                
                    return wav_data
                    
                    if len(mp3_data) > 1000:
                        logger.info(f"TTS generated: {len(mp3_data)} bytes")
                        return mp3_data
                    else:
                        raise Exception("Generated audio too small")
                        
                except Exception as e:
                    if os.path.exists(tmp_file.name):
                        os.unlink(tmp_file.name)
                    raise e
        
        logger.warning("Using silent fallback")
        audio_segment = AudioSegment.from_file(tmp_file.name, format="mp3") 
        wav_buffer = io.BytesIO()
        audio_segment.export(wav_buffer, format="wav")
        return wav_buffer.getvalue()
        
    except Exception as e:
        logger.error(f"TTS error: {e}")  
        try:
            audio_segment = AudioSegment.from_file(tmp_file.name, format="mp3") 
            wav_buffer = io.BytesIO()
            audio_segment.export(wav_buffer, format="wav")
            return wav_buffer.getvalue()
        except:
            return b''  

@app.route('/process_audio', methods=['POST'])
def process_audio():  
    start_time = time.time()
    
    if not all([stt_pipeline, llm_model, llm_tokenizer, tts_pipeline]):
        logger.error("Models not ready")
        return jsonify({"error": "Models are still loading, please wait..."}), 503
    
    if not request.data:
        return jsonify({"error": "No audio data received"}), 400
    
    if len(request.data) < 1000:
        return jsonify({"error": "Audio data too small"}), 400
     
    initial_memory = get_memory_usage()
    logger.info(f"🎯 Processing started. Memory: {initial_memory['rss_mb']:.1f}MB")

    try: 
        logger.info("🎤 Converting speech to text...")
        stt_start = time.time()
        
        rate, audio_data = preprocess_audio_optimized(request.data)
        
        if audio_data is None:
            return jsonify({"error": "Invalid or too short audio"}), 400
        
        stt_result = stt_pipeline(
            {"sampling_rate": rate, "raw": audio_data},
            generate_kwargs={"language": "en"}
        )
        transcribed_text = stt_result.get('text', '').strip()
         
        del audio_data
        gc.collect()
        
        stt_time = time.time() - stt_start
        logger.info(f"✅ STT completed: '{transcribed_text}' ({stt_time:.2f}s)")
        
        if not transcribed_text or len(transcribed_text) < 2:
            transcribed_text = "Could you repeat that please?"

        logger.info("🤖 Generating AI response...")
        llm_start = time.time()
        
        text_hash = hash(transcribed_text.lower())
        assistant_response = cached_generate_response(text_hash, transcribed_text)
        
        llm_time = time.time() - llm_start
        logger.info(f"✅ LLM completed: '{assistant_response}' ({llm_time:.2f}s)")

        logger.info("🔊 Converting to speech...")
        tts_start = time.time()
        
        audio_response = generate_tts_audio(assistant_response)
        
        if not audio_response:
            return jsonify({"error": "TTS generation failed"}), 500
        
        tts_time = time.time() - tts_start
        total_time = time.time() - start_time
         
        gc.collect()
        torch.cuda.empty_cache() if device == "cuda" else None
        
        final_memory = get_memory_usage()
        logger.info(f"✅ Processing complete! Total: {total_time:.2f}s (STT:{stt_time:.1f}s, LLM:{llm_time:.1f}s, TTS:{tts_time:.1f}s)")
        logger.info(f"Memory: {initial_memory['rss_mb']:.1f}MB → {final_memory['rss_mb']:.1f}MB")

        if not os.path.exists(app.config['TEMP_AUDIO_DIR']):
            os.makedirs(app.config['TEMP_AUDIO_DIR'])
        
        file_id = str(uuid.uuid4())
        temp_filename = os.path.join(app.config['TEMP_AUDIO_DIR'], f"{file_id}.mp3")
        
        temp_filename = os.path.join(app.config['TEMP_AUDIO_DIR'], f"{file_id}.wav")
        with open(temp_filename, 'wb') as f:
            f.write(audio_response)
         
        with file_cleanup_lock:
            active_files[file_id] = {
                'filepath': temp_filename,
                'created_time': time.time(),
                'accessed': False
            }
         
        response_data = {
            'status': 'success',
            'file_id': file_id,
            'stream_url': f'/stream_audio/{file_id}',
            'message': assistant_response,
            'transcribed': transcribed_text,
            'processing_time': round(total_time, 2)
        }
        
        return jsonify(response_data)

    except Exception as e:
        logger.error(f"❌ Processing error: {e}", exc_info=True)
        gc.collect()
        torch.cuda.empty_cache() if device == "cuda" else None
        
        return jsonify({
            "error": "Processing failed",
            "details": str(e) if not IS_HF_SPACE else "Internal server error"
        }), 500

@app.route('/stream_audio/<file_id>')
def stream_audio(file_id):
    try:
        with file_cleanup_lock:
            if file_id in active_files:
                active_files[file_id]['accessed'] = True
                filepath = active_files[file_id]['filepath']
                
                if os.path.exists(filepath):
                    logger.info(f"Streaming audio: {file_id}")
                return send_file(
                    filepath,
                    mimetype='audio/wav',
                    as_attachment=False,
                    download_name='response.wav'
                )
        
        logger.warning(f"Audio file not found: {file_id}")
        return jsonify({'error': 'File not found'}), 404
        
    except Exception as e:
        logger.error(f"Stream error: {e}")
        return jsonify({'error': 'Stream failed'}), 500

@app.route('/health', methods=['GET'])
def health_check(): 
    memory = get_memory_usage()
    
    status = {
        "status": "ready" if all([stt_pipeline, llm_model, llm_tokenizer, tts_pipeline]) else "loading",
        "models": {
            "stt": stt_pipeline is not None,
            "llm": llm_model is not None and llm_tokenizer is not None,
            "tts": tts_pipeline is not None,
            "tts_type": tts_type
        },
        "system": {
            "device": device,
            "is_hf_space": IS_HF_SPACE,
            "memory_mb": round(memory['rss_mb'], 1),
            "available_mb": round(memory['available_mb'], 1),
            "memory_percent": round(memory['percent'], 1)
        },
        "files": {
            "active_count": len(active_files),
            "cleanup_running": cleanup_thread is not None and cleanup_thread.is_alive()
        }
    }
    
    return jsonify(status)

@app.route('/status', methods=['GET'])
def simple_status():
    models_ready = all([stt_pipeline, llm_model, llm_tokenizer, tts_pipeline])
    return jsonify({"ready": models_ready})

@app.route('/', methods=['GET'])
def home(): 
    return """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Voice AI Assistant</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; }
            .status { font-size: 18px; margin: 20px 0; }
            .ready { color: green; }
            .loading { color: orange; }
            .error { color: red; }
            code { background: #f4f4f4; padding: 2px 5px; }
        </style>
    </head>
    <body>
        <h1>🎙️ Voice AI Assistant Server</h1>
        <div class="status">Status: <span id="status">Checking...</span></div>
        
        <h2>API Endpoints:</h2>
        <ul>
            <li><code>POST /process_audio</code> - Dsn Mechanics </li>
            <li><code>POST /process_audio</code> - Process audio (WAV format, max 16MB)</li>
            <li><code>GET /stream_audio/&lt;file_id&gt;</code> - Download audio response</li>
            <li><code>GET /health</code> - Detailed health check</li>
            <li><code>GET /status</code> - Simple ready status</li>
        </ul>
        
        <h2>Features:</h2>
        <ul>
            <li>Speech-to-Text (Whisper Tiny)</li>
            <li>AI Response Generation (DialoGPT Small)</li>
            <li>Text-to-Speech (gTTS)</li>
            <li>Automatic file cleanup</li>
            <li>Memory optimization</li>
        </ul>
        
        <p><em>Optimized for ESP32 and Hugging Face Spaces</em></p>
        
        <script>
        function updateStatus() {
            fetch('/status')
            .then(r => r.json())
            .then(d => {
                const statusEl = document.getElementById('status');
                if (d.ready) {
                    statusEl.textContent = '✅ Ready';
                    statusEl.className = 'ready';
                } else {
                    statusEl.textContent = '⏳ Loading models...';
                    statusEl.className = 'loading';
                }
            })
            .catch(() => {
                document.getElementById('status').textContent = '❌ Error';
                document.getElementById('status').className = 'error';
            });
        }
        
        updateStatus();
        setInterval(updateStatus, 5000);
        </script>
    </body>
    </html>
    """

@app.errorhandler(Exception)
def handle_exception(e):
    logger.error(f"Unhandled exception: {e}", exc_info=True)
    return jsonify({"error": "Internal server error"}), 500

@app.errorhandler(413)
def handle_large_file(e):
    return jsonify({"error": "Audio file too large (max 16MB)"}), 413

if __name__ == '__main__':
    try:
        logger.info("🚀 Starting Voice AI Assistant Server")
        logger.info(f"Environment: {'Hugging Face Spaces' if IS_HF_SPACE else 'Local'}")
        
        initialize_models()
        logger.info("🎉 Server ready!")
        
    except Exception as e:
        logger.error(f"❌ Startup failed: {e}")
        exit(1)
     
    port = int(os.environ.get('PORT', 7860))
    logger.info(f"🌐 Server starting on port {port}")
     
    app.run(
        host='0.0.0.0', 
        port=port, 
        debug=False,
        threaded=True,
        use_reloader=False
    )
