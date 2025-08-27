**Cerebra – AI-Powered Conversational Assistant**

Cerebra is an AI-driven virtual assistant built with Python, Android Studio, and LiveKit sandbox, designed to enable real-time conversations, intelligent responses, and seamless cross-platform interaction. It combines the flexibility of Python with the mobility of Android, making it accessible on both desktop and mobile.

**🔹 Key Features**

Conversational AI – Engages in natural human-like dialogue using NLP.

Voice & Chat Support – Real-time communication powered by LiveKit.

Cross-Platform – Android Studio integration enables mobile app deployment.

API Integrations – Utilizes external APIs for smart, dynamic responses.

Modular & Scalable – Easy to extend with new commands or services.

🔹 **Tech Stack**

Languages: Python, Java/Kotlin (Android Studio)

Frameworks/Tools: LiveKit Sandbox, Android Studio

APIs: Custom API integrations for enhanced assistant features

Libraries: NLP/AI libraries (customizable)

🔹** Use Cases**

Personal AI assistant for daily task automation

Mobile-ready chatbot for real-time interactions

Prototype for AI-powered customer support

Learning platform for experimenting with AI + mobile apps

**🔹 Why This Project?**

Jarvis was developed as a hands-on project to explore how AI can be integrated into real-time communication systems and extended to Android applications. It demonstrates skills in:

Python-based AI development

Android app creation with Android Studio

API handling and real-time communication

Cross-platform integration

**⚡ Future Plans:**

Smarter memory & personalization

IoT device integration

Multi-language support



**HOW TO RUN**
Step1:Windows

1.Install
  Python 3.13 (already done)
  Git
  VS Code
  (Optional) Android Studio if you’ll run this on phone later 
  
2.Create a project & venv
   mkdir jarvis-agent && cd jarvis-agent
  python -m venv venv
  venv\Scripts\activate
  
3.Install dependencies
  pip install --upgrade pip
  pip install livekit-agents livekit-plugins-google sounddevice python-dotenv

  livekit-agents = the framework

livekit-plugins-google = Gemini LLM + Google Cloud STT/TTS

sounddevice = mic/speaker I/O for console testing

python-dotenv = loads your .env

Step2:Keys & .env
A) Google Gemini (LLM)

Get a Gemini API Key (not the Cloud JSON).

Put it in .env as GOOGLE_API_KEY.

B) Google Cloud STT/TTS (voice in/out)

Create a Google Cloud Project

Enable Cloud Text-to-Speech and Speech-to-Text

Create a Service Account and download the JSON key.

Save it somewhere like: C:\keys\gcloud.json

C) LiveKit (for mobile later)

Create a LiveKit Cloud project.

Note LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET (for later token server).

Create .env in your project root:

# LLM
GOOGLE_API_KEY=your_gemini_api_key_here

# Google Cloud STT/TTS (service account JSON)
GOOGLE_APPLICATION_CREDENTIALS=C:\keys\gcloud.json

# LiveKit (only needed when you hook up Android later)
LIVEKIT_URL=wss://<your-livekit-host>
LIVEKIT_API_KEY=<your_key>
LIVEKIT_API_SECRET=<your_secret>


Gemini + LiveKit plugin info: docs show how to set model IDs (e.g., gemini-2.0-flash-001).

Voice pipeline abstraction details here. 
LiveKit Docs

Step:3 Minimal “Jarvis” Console Agent (talk from your PC)

Create agent.py in the project folder:

import os
import asyncio
from dotenv import load_dotenv

from livekit.agents import JobContext
from livekit.agents.voice import Agent, AgentSession
from livekit.agents import llm
from livekit.plugins import google

# Load .env
load_dotenv()

SYSTEM_INSTRUCTIONS = (
    "You are Jarvis, a concise, friendly, real-time assistant. "
    "Speak clearly and keep answers short unless asked to go deep. "
    "If the user asks you to do something unsafe or off-limits, politely refuse."
)

async def entrypoint(ctx: JobContext):
    # --- LLM (Gemini) ---
    gemini = google.LLM(
        api_key=os.getenv("GOOGLE_API_KEY"),
        model="gemini-2.0-flash-001",   # good balance of speed+quality
        # temperature=0.7,               # optional tuning
    )

    # --- STT / TTS (Google Cloud) ---
    # Uses GOOGLE_APPLICATION_CREDENTIALS from env
    stt = google.STT()  # default uses Google Cloud credentials from env
    tts = google.TTS(voice_name="en-US-Standard-B")  # pick any available voice

    voice_agent = Agent(
        llm=gemini,
        stt=stt,
        tts=tts,
        instructions=SYSTEM_INSTRUCTIONS,
        # You can also pass tools=[], chat_ctx=llm.ChatContext(), etc.
    )

    session = AgentSession()
    # IMPORTANT: When you run with "console" subcommand, AgentSession will
    # attach a local ChatCLI mic/speaker. (See docs)
    await session.start(voice_agent)

if __name__ == "__main__":
    # Run in console mode:
    #   python agent.py console
    # (The live console is wired inside AgentSession when CLI flag "console" is used)
    import livekit.agents.cli as cli
    cli.run_app(entrypoint)


The console mode auto-wires your mic/speaker via sounddevice (ChatCLI). 
LiveKit Docs

Run it:

python agent.py console


You should see logs like “LiveKit Agents – Console” and hear Jarvis respond in real time.

Step:4 Fixing Windows mic issues (PortAudioError -9999)

If you hit:

sounddevice.PortAudioError: Error opening InputStream: Unanticipated host error [PaErrorCode -9999]: 'Undefined external error.' [MME error 1]


Try these (most common fixes):

Set default recording device in Windows > Sound > Input (pick your mic).

Match sample rate: set mic to 44100 Hz (Windows Sound > Recording device > Properties > Advanced).

Disable “Exclusive Mode” for the mic in the same dialog.

Allow microphone access: Windows Privacy > Microphone ON.

Close apps that might lock the mic (MS Teams, Zoom, etc.).

Change host API (WASAPI is often safer than MME). With sounddevice, WASAPI is used by default on newer systems; if not, try specifying a device index (list devices with a small Python snippet).

Update audio drivers (Device Manager) and reboot.

The console wiring and ChatCLI behavior are described in LiveKit’s voice session docs. 
LiveKit Docs
GitHub

Step:5 (Optional) Token server for Android/iOS/web clients

When you move beyond local console, your mobile client must join a LiveKit room with a short-lived access token minted by your server.

Create token_server.py:

import os
from dotenv import load_dotenv
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from livekit import api as lk_api

load_dotenv()

LIVEKIT_URL = os.getenv("LIVEKIT_URL")
LIVEKIT_API_KEY = os.getenv("LIVEKIT_API_KEY")
LIVEKIT_API_SECRET = os.getenv("LIVEKIT_API_SECRET")

app = FastAPI()
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], allow_credentials=True,
    allow_methods=["*"], allow_headers=["*"],
)

class TokenReq(BaseModel):
    identity: str
    room: str = "jarvis-room"

@app.post("/token")
def create_token(req: TokenReq):
    at = lk_api.AccessToken(LIVEKIT_API_KEY, LIVEKIT_API_SECRET)
    at.identity = req.identity
    at.add_grant(lk_api.VideoGrant(room=req.room, room_join=True))
    token = at.to_jwt()
    return {"token": token, "url": LIVEKIT_URL}


Install and run:

pip install fastapi uvicorn "livekit-api>=0.10"
uvicorn token_server:app --host 0.0.0.0 --port 8787


Your Android app requests POST /token to get { token, url }, then joins the room.

LiveKit token flow: see official docs. 
GitHub

Step:6 Android Studio integration (high level)

Open the LiveKit Android quickstart/app and add your token server URL.

On app launch, your client fetches a token for identity=<device or user> and room=jarvis-room.

Client joins the room at LIVEKIT_URL using the token.

Your Python agent also joins or acts as the orchestrator for voice I/O through the Agents framework (depending on architecture you choose).

Send/receive audio and (optionally) text/data streams. 
LiveKit Docs

Step:7 Variations & upgrades

Realtime LLM: You can swap the STT→LLM→TTS pipeline for Gemini Live (realtime) if you want server-side streaming LLM with turn-taking; see plugin docs.

Tool use / functions: Add function tools (e.g., weather, system commands) via llm.FunctionTool and agent.update_tools(...). 
LiveKit Docs

RAG: Add a knowledge base (vector search) and call it from your tool functions.

Deployment: Run your agent behind livekit-agents worker in the cloud, and keep the mobile client thin.

Quick sanity test

With venv active and .env set, run:

python agent.py console


Speak into your mic: “Hey Jarvis, what can you do?”
You should hear a spoken reply.
 


