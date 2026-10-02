import json
import streamlit as st
from google import genai
from google.genai import types
from twilio.rest import Client as TwilioClient

# ============================================================
# MacroSnap — AI Vision Nutrition Chatbot
# Single-file Streamlit application
# ============================================================

MODEL_NAME = "gemini-3.5-flash"

st.set_page_config(
    page_title="MacroSnap",
    page_icon="🥗",
    layout="centered",
    initial_sidebar_state="expanded",
)

# -----------------------------
# Custom UI
# -----------------------------
st.markdown(
    """
    <style>
    .stApp {
        background: linear-gradient(135deg, #f8fff8 0%, #f4f7ff 100%);
    }

    .hero {
        padding: 28px 24px;
        border-radius: 24px;
        background: linear-gradient(135deg, #0f766e, #2563eb);
        color: white;
        margin-bottom: 20px;
        box-shadow: 0 12px 35px rgba(15,118,110,.18);
    }

    .hero h1 {
        margin: 0;
        font-size: 2.5rem;
    }

    .hero p {
        margin: 8px 0 0 0;
        opacity: .92;
        font-size: 1.05rem;
    }

    .card {
        background: rgba(255,255,255,.88);
        border: 1px solid rgba(15,118,110,.10);
        border-radius: 18px;
        padding: 18px;
        margin: 10px 0;
        box-shadow: 0 7px 25px rgba(0,0,0,.05);
    }

    .metric-card {
        background: white;
        border-radius: 16px;
        padding: 16px;
        text-align: center;
        border: 1px solid #e5e7eb;
    }

    .metric-value {
        font-size: 1.7rem;
        font-weight: 700;
        color: #0f766e;
    }

    .metric-label {
        color: #6b7280;
        font-size: .85rem;
    }

    .small-note {
        color: #6b7280;
        font-size: .82rem;
    }

    div[data-testid="stChatMessage"] {
        border-radius: 16px;
    }

    .stButton > button {
        border-radius: 12px;
        font-weight: 600;
    }
    </style>
    """,
    unsafe_allow_html=True,
)

# -----------------------------
# Secrets
# -----------------------------
def get_secret(name, default=None):
    try:
        return st.secrets[name]
    except Exception:
        return default


GEMINI_API_KEY = get_secret("GEMINI_API_KEY")
TWILIO_ACCOUNT_SID = get_secret("TWILIO_ACCOUNT_SID")
TWILIO_AUTH_TOKEN = get_secret("TWILIO_AUTH_TOKEN")
TWILIO_WHATSAPP_FROM = get_secret(
    "TWILIO_WHATSAPP_FROM", "whatsapp:+14155238886"
)
TWILIO_CONTENT_SID = get_secret("TWILIO_CONTENT_SID")


# -----------------------------
# Prompts
# -----------------------------
SYSTEM_PROMPT = """
You are MacroSnap, a friendly AI nutrition buddy.

Your ONLY job is to help the user understand what they're eating by
estimating calories and macros from a meal photo or text description.

If the user asks about anything unrelated to food, nutrition, meals,
or fitness, politely decline and steer the conversation back to food.

When estimating a meal, always include:
1. What the meal appears to be
2. Estimated calories
3. Estimated protein, carbs, and fat
4. A brief note that estimates are approximate

Keep replies short, friendly, useful, and conversational.
Do not present estimates as medically precise facts.
"""


SUMMARY_REQUEST_PROMPT = """
Summarize every meal discussed in this conversation into one
WhatsApp-friendly plain-text message.

For each meal/item:
- Give its estimated calories.

Then provide the combined estimated totals for:
- Calories
- Protein
- Carbohydrates
- Fat

Keep it concise and readable.
Mention that nutrition values are estimates.
Do not use markdown tables.
"""


WELCOME_MESSAGE = """
Hi {name}! 🥗 I'm MacroSnap.

I can look at a meal photo or a text description and give you an
estimated calorie and macro breakdown.

📸 Upload a meal photo
💬 Or type what you ate
📊 I'll estimate calories, protein, carbs and fat

When you're finished, use the WhatsApp button to send yourself a
summary of the conversation.
"""


# -----------------------------
# Cached API clients
# -----------------------------
@st.cache_resource
def get_gemini_client(api_key):
    if not api_key:
        return None
    return genai.Client(api_key=api_key)


@st.cache_resource
def get_twilio_client(account_sid, auth_token):
    if not account_sid or not auth_token:
        return None
    return TwilioClient(account_sid, auth_token)


gemini_client = get_gemini_client(GEMINI_API_KEY)
twilio_client = get_twilio_client(
    TWILIO_ACCOUNT_SID,
    TWILIO_AUTH_TOKEN,
)


# -----------------------------
# Session state
# -----------------------------
def init_state():
    defaults = {
        "onboarded": False,
        "name": "",
        "whatsapp_number": "",
        "chat": None,
        "messages": [],
        "meal_count": 0,
    }

    for key, value in defaults.items():
        if key not in st.session_state:
            st.session_state[key] = value


init_state()


# -----------------------------
# Helpers
# -----------------------------
def add_message(role, kind, content):
    st.session_state.messages.append(
        {
            "role": role,
            "kind": kind,
            "content": content,
        }
    )


def render_message(message):
    with st.chat_message(message["role"]):
        if message["kind"] == "text":
            st.markdown(message["content"])
        elif message["kind"] == "image":
            st.image(message["content"], use_container_width=True)


def ask_gemini(parts):
    if gemini_client is None:
        return (
            "Gemini is not configured yet. Add GEMINI_API_KEY to "
            ".streamlit/secrets.toml and restart the app."
        )

    if st.session_state.chat is None:
        return "Your chat session is not initialized. Please restart the app."

    try:
        response = st.session_state.chat.send_message(parts)
        return response.text or "I couldn't generate a response."
    except Exception as error:
        return f"Sorry, something went wrong while contacting Gemini: {error}"


def clean_whatsapp_text(text):
    if not text:
        return "No nutrition summary available."

    text = " ".join(str(text).split())

    if len(text) > 1500:
        return text[:1500] + "..."

    return text


def send_whatsapp(to_number, user_name, summary):
    if twilio_client is None:
        return False, "Twilio is not configured."

    if not TWILIO_CONTENT_SID:
        return False, "TWILIO_CONTENT_SID is missing."

    try:
        variables = json.dumps(
            {
                "1": user_name,
                "2": clean_whatsapp_text(summary),
            },
            ensure_ascii=False,
        )

        message = twilio_client.messages.create(
            from_=TWILIO_WHATSAPP_FROM,
            to=f"whatsapp:{to_number}",
            content_sid=TWILIO_CONTENT_SID,
            content_variables=variables,
        )

        return True, message.sid

    except Exception as error:
        return False, str(error)


def start_chat(name, whatsapp_number):
    st.session_state.name = name.strip()
    st.session_state.whatsapp_number = whatsapp_number.strip()

    st.session_state.chat = gemini_client.chats.create(
        model=MODEL_NAME,
        config=types.GenerateContentConfig(
            system_instruction=SYSTEM_PROMPT
        ),
    )

    st.session_state.messages = []
    st.session_state.meal_count = 0
    st.session_state.onboarded = True

    add_message(
        "assistant",
        "text",
        WELCOME_MESSAGE.format(name=st.session_state.name),
    )


def reset_app():
    for key in [
        "onboarded",
        "name",
        "whatsapp_number",
        "chat",
        "messages",
        "meal_count",
    ]:
        if key in st.session_state:
            del st.session_state[key]

    st.rerun()


# ============================================================
# SIDEBAR
# ============================================================
with st.sidebar:
    st.markdown("## 🥗 MacroSnap")

    st.markdown(
        """
        <div class="card">
        <b>AI Nutrition Buddy</b><br>
        Upload a meal photo or describe your meal and get an estimated
        calorie and macro breakdown.
        </div>
        """,
        unsafe_allow_html=True,
    )

    if st.session_state.onboarded:
        st.markdown("### Your session")
        st.write(f"👤 **{st.session_state.name}**")
        st.write(f"📱 **{st.session_state.whatsapp_number}**")
        st.write(f"🍽️ Meals/questions: **{st.session_state.meal_count}**")

        st.divider()

        if st.button("🔄 Start New Session", use_container_width=True):
            reset_app()

    st.divider()

    st.caption(
        "Nutrition values are estimates and should not be treated as "
        "medical advice."
    )


# ============================================================
# ONBOARDING
# ============================================================
if not st.session_state.onboarded:

    st.markdown(
        """
        <div class="hero">
            <h1>🥗 MacroSnap</h1>
            <p>Snap it. Track it. Send your nutrition summary.</p>
        </div>
        """,
        unsafe_allow_html=True,
    )

    st.markdown(
        """
        <div class="card">
        <h3>👋 Welcome!</h3>
        <p>
        MacroSnap uses Gemini vision and chat to estimate calories and
        macros from food photos or descriptions.
        </p>
        </div>
        """,
        unsafe_allow_html=True,
    )

    if not GEMINI_API_KEY:
        st.warning(
            "Gemini API key is not configured. You can enter your details "
            "below, but AI analysis will not work until the key is added."
        )

    with st.form("onboarding_form"):
        name = st.text_input(
            "Your name",
            placeholder="Enter your name",
        )

        whatsapp_number = st.text_input(
            "WhatsApp number",
            placeholder="+91XXXXXXXXXX",
            help="Include your country code.",
        )

        submitted = st.form_submit_button(
            "Let's Get Started 🚀",
            use_container_width=True,
        )

    if submitted:
        if not name.strip() or not whatsapp_number.strip():
            st.error("Please enter both your name and WhatsApp number.")

        elif gemini_client is None:
            st.error(
                "Gemini API key is missing. Add GEMINI_API_KEY to "
                ".streamlit/secrets.toml and restart the app."
            )

        else:
            start_chat(name, whatsapp_number)
            st.rerun()

    st.stop()


# ============================================================
# MAIN HEADER
# ============================================================
col1, col2 = st.columns([5, 2], vertical_alignment="center")

with col1:
    st.markdown(
        """
        <div>
            <h1 style="margin-bottom:0;">🥗 MacroSnap</h1>
            <p style="color:#6b7280;margin-top:3px;">
                Your AI meal & macro decoder
            </p>
        </div>
        """,
        unsafe_allow_html=True,
    )

with col2:
    has_real_chat = len(st.session_state.messages) > 1

    if st.button(
        "📤 Send Summary",
        disabled=not has_real_chat,
        use_container_width=True,
    ):
        with st.spinner("Preparing your nutrition summary..."):
            summary = ask_gemini([SUMMARY_REQUEST_PROMPT])

        with st.spinner("Sending to WhatsApp..."):
            success, info = send_whatsapp(
                st.session_state.whatsapp_number,
                st.session_state.name,
                summary,
            )

        if success:
            st.success("Summary sent to WhatsApp! 📲")
        else:
            st.error(f"WhatsApp send failed: {info}")


# ============================================================
# QUICK INFO
# ============================================================
m1, m2, m3 = st.columns(3)

with m1:
    st.markdown(
        f"""
        <div class="metric-card">
            <div class="metric-value">📸</div>
            <div class="metric-label">Photo Analysis</div>
        </div>
        """,
        unsafe_allow_html=True,
    )

with m2:
    st.markdown(
        f"""
        <div class="metric-card">
            <div class="metric-value">🤖</div>
            <div class="metric-label">Gemini AI</div>
        </div>
        """,
        unsafe_allow_html=True,
    )

with m3:
    st.markdown(
        f"""
        <div class="metric-card">
            <div class="metric-value">{st.session_state.meal_count}</div>
            <div class="metric-label">Logged Inputs</div>
        </div>
        """,
        unsafe_allow_html=True,
    )


st.divider()


# ============================================================
# CHAT HISTORY
# ============================================================
for message in st.session_state.messages:
    render_message(message)


# ============================================================
# CHAT INPUT
# ============================================================
user_input = st.chat_input(
    "Ask about a meal or attach a food photo 📷",
    accept_file=True,
    file_type=["jpg", "jpeg", "png"],
)

if user_input:

    photo = user_input.files[0] if user_input.files else None
    text = user_input.text.strip() if user_input.text else ""

    parts = []

    if photo is not None:
        photo_bytes = photo.getvalue()

        add_message(
            "user",
            "image",
            photo_bytes,
        )

        parts.append(
            types.Part.from_bytes(
                data=photo_bytes,
                mime_type=photo.type,
            )
        )

    if text:
        add_message(
            "user",
            "text",
            text,
        )
        parts.append(text)

    elif photo is not None:
        parts.append(
            "What is this meal? Give me the estimated calories, "
            "protein, carbs and fat."
        )

    if parts:
        st.session_state.meal_count += 1

        with st.spinner("🔎 Analyzing your meal..."):
            answer = ask_gemini(parts)

        add_message(
            "assistant",
            "text",
            answer,
        )

        st.rerun()
