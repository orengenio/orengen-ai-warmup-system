# Orengen AI Warm-Up System

This repo contains the full self-hosted warm-up system using mailcow and free emails:

#!/usr/bin/env python3

import imaplib
import smtplib
import ssl
import email
import time
import random
import logging
import json
import os
import sys
import datetime
import re
import textwrap
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from email.header import Header
from collections import defaultdict, Counter
from pathlib import Path

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler("email_warmup.log"),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger("email_warmup")

# === PERSONALIZATION CONTENT POOLS ===
# Subject Lines - natural, conversational, non-spammy, personalized
SUBJECTS = [
    # Professional/Business
    "Quick question, {name}",
    "Following up on our conversation, {name}"
    "{name}, thoughts on the latest project?",
    "{name}, can we connect this week?",
    "Need your input on this, {name}",
    "Checking in about yesterday’s meeting, {name}",
    "{name}, feedback on the draft report?",
    "Let’s catch up, {name}",
    "{name}, question about your recent email",
    "Wanted your perspective, {name}",
    "Update on the client situation, {name}",
    "{name}, have you seen this article?",
    "A couple things to discuss, {name}",
    "Need your expertise on something, {name}",
    "{name}, possible collaboration idea",
    "Sharing some thoughts with you, {name}",
    "{name}, your opinion would help",
    "Missed you at the conference, {name}",
    "Circling back on our discussion, {name}",
    "{name}, an update on the Smith account",
    "{name}, introducing you to someone",
    "Quick recommendation question, {name}",
    "Touching base, {name}",
    "Confirming our appointment, {name}",
    "{name}, can we reschedule our meeting?",
    "{name}, important update for the team",
    "Quarterly review notes, {name}",
    "Preparing for next week, {name}",
    "{name}, feedback on your presentation",
    "{name}, invitation to speak at our event",
    "Congrats again, {name}",
    "{name}, thoughts on the industry changes?",
    "Some data you might appreciate, {name}",
    "Sharing the resources you requested, {name}",
    "{name}, checking your availability for next month",
    "Proposal for your consideration, {name}",
    "Responding to your inquiry, {name}",
    "{name}, quick clarification on the timeline",
    "Insights from the market research, {name}",
    "{name}, exploring a partnership idea",
    "Addressing your concerns, {name}",
    "{name}, confirming receipt of the documents",
    "Requesting a brief conversation, {name}",
    "Good news on the project, {name}",
    "{name}, introducing our newest teammate",
    "{name}, invitation to our annual event",
    "Appreciate your recent work, {name}",
    "{name}, scheduling a quick convo",
    "{name}, sharing a relevant case study",
    "Your take on the competitive landscape, {name}",

    # Casual/Friendly
    "Lunch next week, {name}?",
    "{name}, did you see this?",
    "Quick favor to ask, {name}",
    "Catching up, {name}",
    "How’s everything going, {name}?",
    "Long time no chat, {name}",
    "Thought you’d find this interesting, {name}",
    "Checking in, {name}",
    "Coffee sometime soon, {name}?",
    "{name}, remember that thing we discussed?",
    "Funny story you’ll appreciate, {name}",
    "Your thoughts on this, {name}?",
    "Plans this weekend, {name}?",
    "Saw this and thought of you, {name}",
    "How’ve you been, {name}?",
    "Reconnecting after a while, {name}",
    "This reminded me of our conversation, {name}",
    "Touching base, {name}",
    "Quick question, {name}",
    "Wanted to share this with you, {name}",
    "Thinking of you, {name}",
    "{name}, guess what happened yesterday",
    "{name}, quick recommendation when you have time",
    "{name}, you’ll get a kick out of this",
    "Finally got around to it, {name}",
    "Been a minute, {name}",
    "Blast from the past, {name}",
    "Just saying hello, {name}",
    "How’s everything treating you, {name}?",
    "Checking in again, {name}",
    "Thought I’d reach out, {name}",
    "{name}, heard the news?",
    "Interesting development, {name}",
    "Your advice would help, {name}",
    "Catching up after the holidays, {name}",
    "How’s the  position going, {name}?",
    "Missed seeing you at the event, {name}",
    "Wanted to reconnect, {name}",
    "This made me think of you, {name}",
    "Sharing something you might like, {name}",
    "How’s the family, {name}?",
    "{name}, remember when we talked about…",
    "Curious about your thoughts, {name}",
    "Reaching out after a while, {name}",
    "{name}, heard you were in town",
    "Let’s grab coffee sometime, {name}",
    "Update on that situation, {name}",
    "Funny coincidence, {name}",
    "Checking how you’re doing, {name}",

    # Specific/Contextual
    "{name}, about the Johnson project",
    "{name}, your feedback on the design",
    "Next steps for our initiative, {name}",
    "{name}, the conference in Chicago",
    "{name}, that book you recommended",
    "The strategy we discussed, {name}",
    "{name}, your recent article",
    "Our conversation last Thursday, {name}",
    "{name}, the latest software implementation",
    "Your expertise on this, {name}",
    "{name}, the upcoming board meeting",
    "{name}, the client presentation",
    "Your insights on market trends, {name}",
    "{name}, the team restructuring",
    "Your experience with the vendor, {name}",
    "{name}, the budget allocation",
    "{name}, your recommendation for the position",
    "{name}, product launch timeline",
    "{name}, your thoughts on the proposal",
    "{name}, recent policy changes",
    "Your input on the strategic plan, {name}",
    "{name}, the quarterly results",
    "Your perspective on the industry shift, {name}",
    "{name}, the upcoming deadline",
    "{name}, your availability for the event",
    "{name}, partnership agreement",
    "{name}, feedback on the prototype",
    "{name}, customer feedback report",
    "{name}, your opinion on the updated direction",
    "{name}, the training program",
    "{name}, your experience with the platform",
    "{name}, the updated forecast",
    "{name}, your approach to the challenge",
    "{name}, compliance requirements",
    "{name}, your take on the competitive analysis",
    "{name}, project milestones",
    "{name}, suggestions for improvement",
    "{name}, team evaluations",
    "{name}, insights on the research findings",
    "{name}, contract renewal",
    "{name}, thoughts on the org changes",
    "{name}, quality assurance process",
    "{name}, feedback on the user experience",
    "{name}, resource allocation",
    "{name}, your perspective on the market",
    "{name}, implementation strategy",
    "{name}, your input on decision-making",
    "{name}, risk assessment",
    "{name}, recommendations for next steps",

    # Question-Based
    "{name}, have a minute to discuss something?",
    "{name}, can I get your thoughts on this?",
    "{name}, interested in collaborating?",
    "{name}, do you have time this week?",
    "{name}, are you still working on that project?",
    "{name}, have you considered this approach?",
    "{name}, could you share your expertise on this?",
    "{name}, what do you think about this idea?",
    "{name}, when can we schedule a quick conversation?",
    "{name}, how’s your progress on the initiative?",
    "{name}, did you see my last message?",
    "{name}, can we revisit our conversation?",
    "{name}, want to join us for the event?",
    "{name}, have you seen the latest updates?",
    "{name}, could you provide some guidance?",
    "{name}, what’s your availability next week?",
    "{name}, are you familiar with this?",
    "{name}, how would you handle this?",
    "{name}, can I ask for your recommendation?",
    "{name}, would this be of interest?",
    "{name}, have you dealt with this before?",
    "{name}, could we discuss this further?",
    "{name}, what are your thoughts on the proposal?",
    "{name}, any insights on this?",
    "{name}, have you made a decision yet?",
    "{name}, can we align on next steps?",
    "{name}, willing to provide feedback?",
    "{name}, how do you feel about the changes?",
    "{name}, did you review it yet?",
    "{name}, can we touch base this afternoon?",
    "{name}, what’s your take on the situation?",
    "{name}, heard about the recent developments?",
    "{name}, could you share your perspective?",
    "{name}, are we still on for tomorrow?",
    "{name}, have you finalized the details?",
    "{name}, I need your input on something",
    "{name}, would this time work?",
    "{name}, how’s your schedule looking?",
    "{name}, did that info help?",
    "{name}, can we discuss the results?",
    "{name}, what do you recommend?",
    "{name}, have you considered alternatives?",
    "{name}, could you clarify your requirements?",
    "{name}, are you satisfied with progress?",
    "{name}, have you made any headway?",
    "{name}, can we coordinate on this?",
    "{name}, what’s the status of the project?",
    "{name}, did you connect with the team?",
    "{name}, could we revisit this?",

    # Time-Sensitive
    "{name}, before our meeting tomorrow",
    "{name}, quick update before the weekend",
    "{name}, need your input by Thursday",
    "{name}, following up before the deadline",
    "{name}, reminder about tomorrow’s meet",
    "{name}, can you provide feedback within a few hours?",
    "{name}, time-sensitive question",
    "{name}, before you head out",
    "{name}, quick question before the presentation",
    "{name}, heads up for next week",
    "{name}, prepping for Monday’s meeting",
    "{name}, end-of-month weigh-in",
    "{name}, before the quarterly review",
    "{name}, reminder: submission due Friday",
    "{name}, deadline approaching—need your take",
    "{name}, quick update before the holiday",
    "{name}, before the client meeting",
    "{name}, timely info for your review",
    "{name}, reminder about your commitment",
    "{name}, before the board presentation",
    "{name}, need everyone’s input",
    "{name}, quick time-sensitive chat",
    "{name}, before the team announcement",
    "{name}, quick run-down before finalizing",
    "{name}, reminder about our agreement",
    "{name}, before the product launch",
    "{name}, final reminder about the event",
    "{name}, time-sensitive decision point",
    "{name}, clarification needed pretty quick",
    "{name}, before submitting the proposal",
    "{name}, quick confirmation needed",
    "{name}, reminder: registration closing at 5pm",
    "{name}, time-sensitive item to review",
    "{name}, before the press release",
    "{name}, last-minute details",
    "{name}, time-sensitive update",
    "{name}, your approval needed",
    "{name}, before the stakeholder meeting",
    "{name}, quick response requested",
    "{name}, reminder: deadline is near",
    "{name}, need this by EOD",
    "{name}, before the system update",
    "{name}, signature review for approval",
    "{name}, before we make the announcement",
]

# Greetings - Natural greeting variations
GREETINGS = [
    "Hi {name},",
    "Hello {name},",
    "Hey {name},",
    "Good morning {name},",
    "Good afternoon {name},",
    "Good evening {name},",
    "Hi there {name},",
    "Hello there {name},",
    "Hey there {name},",
    "{name},",
    "Dear {name},",
    "Greetings {name},",
    "Hope you're well {name},",
    "Hope this finds you well {name},",
    "Hope you're having a lovely day {name},",
    "Hope you're doing well {name},",
    "Just a quick note {name},",
    "Touching base {name},",
    "Checking in {name},",
    "Happy {day_of_week} {name},"
]

# Closings - Natural closing variations
CLOSINGS = [
    "Best regards,",
    "Regards,",
    "Best,",
    "Cheers,",
    "Thanks,",
    "Thank you,",
    "Sincerely,",
    "Kind regards,",
    "Warm regards,",
    "Best wishes,",
    "Take care,",
    "Talk later,",
    "Until next time,",
    "Appreciate it,",
    "Many thanks,",
    "With appreciation,",
    "Looking forward to hearing from you,",
    "Thanks for your time,",
    "Have a awesome day,",
    "Have a blast of a week,",
    "Yours truly,",
    "Respectfully,",
    "Warmly,",
    "Cordially,"
]

# Common name mappings for email addresses
NAME_MAPPINGS = {
    # All Andre variants are fully branded
    "andre":        "Andre Mandel, orengenio on All Platforms",
    "man":          "Andre Mandel, orengenio on All Platforms",
    "amandel":      "Andre Mandel, orengenio on All Platforms",
    "mandel":       "Andre Mandel, orengenio on All Platforms",
}


# Topics - Common business and personal topics
TOPICS = [
    "the project", "our meeting", "the proposal", "your feedback",
    "the recent changes", "the timeline", "our collaboration",
    "the latest initiative", "the report", "our discussion",
    "the presentation", "the strategy", "the recent update",
    "the team meeting", "the quarterly review", "the client ask",
    "the updated approach", "the analysis", "the research findings",
    "the implementation plan", "the budget allocation", "the market trends",
    "the growth and failure metrics", "the customer feedback", "the product launch",
    "the conference", "the training program", "the partnership proposition",
    "the industry event", "the company policy", "the 2025 regulations",
    "the software update", "the design concept", "the campaign",
    "the projected figures", "the team restructuring", "the client presentation",
    "the workflow improvements", "the quality standards", "the competitive analysis",
    "the resource allocation", "the strategic planning", "the upcoming deadline",
    "the contract renewal", "the vendor selection", "the compliance requirements",
    "the system integration", "the user experience", "the technical specifications",
    "the risk assessment", "the operational efficiency"
]

# Questions - Natural question variations to include in emails
QUESTIONS = [
    "What do you think?",
    "Does that make sense?",
    "Would love your thoughts on this.",
    "Let me know what you think.",
    "Any feedback would be appreciated.",
    "What's your take on this?",
    "Curious to hear your perspective.",
    "Would you agree?",
    "Does this align with your thinking?",
    "How does this sound to you?",
    "Would you approach this differently?",
    "Any suggestions?",
    "Does this work?",
    "Thoughts?",
    "Make sense?",
    "Sound good?",
    "What's your opinion?",
    "Would you add anything?",
    "Any concerns with this approach?",
    "Do you see any issues with this?"
]

# Body Paragraphs - Core content for email bodies
BODY_PARAGRAPHS = [
    "I wanted to follow up on our conversation from last week about {topic}. I've been thinking about your suggestions, and I think you raised some excellent points.",
    "Just wanted to see how things are progressing with {topic}. We're making good headway on our end and should have some updates to share.",
    "I recently came across an interesting article related to {topic} that reminded me of our discussion last month. I thought you might find it relevant to what you're working on.",
    "I've been reflecting on our last meeting about {topic} and had a few additional thoughts I wanted to share with you when you have a moment.",
    "Quick update on {topic} - we've made some progress since we last spoke and I wanted to keep you in the loop on where things stand.",
    "I was reviewing some notes on {topic} earlier and realized we haven't finalized our approach. I'm curious if you've had any further thoughts on this.",
    "I hope your week is going well. I wanted to touch base about {topic} and see if you needed any additional information from my end.",
    "I've been working on {topic} this week and wanted to see your perspective on a couple of ideas I've been developing.",
    "Following up on our discussion about {topic} - I've had a moment to look into the questions you raised and wanted to share what I found.",
    "I wanted to circle back on {topic} before our meeting next week. There are a few points I think we should address before moving forward.",
    "Just checking in to see if you've had a moment to review the information I sent over about {topic}. No rush, just wanted to make sure it wasn't misplaced in your inbox.",
    "I've been thinking about the challenges we discussed about {topic} and had an idea that might help address some of the concerns you mentioned.",
    "I hope you're doing well. I wanted to follow up on {topic} and see if you've made any progress or if there's anything you need from my end.",
    "I was talking with the team about {topic} yesterday, and you were mentioned as someone who might have valuable insights on this. Would love to know your thoughts when you have a moment.",
    "Quick question about {topic} - I'm trying to better understand how this fits into the broader strategy. Your perspective would be really helpful."
]

# === CONFIGURATION AND SETUP ===

# Default paths
DEFAULT_CONFIG_PATH = "/root/email_warmup_responder/free_account_config.json"
MAILCOW_CONFIG_PATH = "/root/email_warmup_responder/mailcow_accounts.json"
DEFAULT_PASSWORD = os.getenv("MAILCOW_PASSWORD")  # fallback
DEFAULT_SMTP_PORT = 587
DEFAULT_IMAP_PORT = 993
MAX_RETRIES = 2
RETRY_DELAY = 5  # seconds

# Mailcow hostnames (same for all your domains)
MAILCOW_SMTP_HOST = "mail.orengenmail.com"
MAILCOW_IMAP_HOST = "mail.orengenmail.com"

# === HELPER FUNCTIONS ===

def get_day_of_week():
    """Get the current day of the week."""
    days = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"]
    return days[datetime.datetime.now().weekday()]

def get_time_appropriate_greeting():
    """Get a greeting appropriate for the current time of day."""
    hour = datetime.datetime.now().hour
    if 5 <= hour < 12:
        return "Good morning"
    elif 12 <= hour < 18:
        return "Good afternoon"
    else:
        return "Good evening"

def extract_name_from_email(email_address):
    """Extract and format name from email address."""
    # Get the part before @
    local_part = email_address.split('@')[0].lower()

    # Check if we have a known mapping
    if local_part in NAME_MAPPINGS:
        return NAME_MAPPINGS[local_part]

    # Clean up the local part
    # Remove numbers
    name_part = re.sub(r'\d+', '', local_part)

    # Replace common separators with spaces
    name_part = name_part.replace('.', ' ').replace('_', ' ').replace('-', ' ')

    # Split camelCase or PascalCase
    name_part = re.sub(r'(?<!^)(?=[A-Z])', ' ', name_part)

    # Title case each word
    name_parts = name_part.split()
    if name_parts:
        formatted_name = ' '.join(word.title() for word in name_parts if word)

        # If the name is too short or empty after cleaning, use the original
        if len(formatted_name) < 2:
            formatted_name = local_part.title()

        return formatted_name

    # Fallback to the original local part
    return local_part.title()

def extract_name_from_email_header(email_header):
    """Extract name from email header (e.g., 'John <john@example.com>')."""
    if not email_header:
        return None

    # Parse the email header
    name, email_address = email.utils.parseaddr(email_header)

    # If we have a name in the header, use it
    if name and name.strip():
        return name.strip()

    # Otherwise, extract from email address
    if email_address:
        return extract_name_from_email(email_address)

    return None

def generate_signature(email_address, sender_name=None):
    """Generate a natural-looking email signature."""
    # Use provided name or extract from email
    if sender_name:
        name = sender_name
    else:
        name = extract_name_from_email(email_address)

    # Get domain for company name
    domain = email_address.split('@')[1]
    company = domain.split('.')[0].title()

    # Randomly decide signature format
    signature_type = random.randint(1, 3)

    if signature_type == 1:
        # Simple name only
        return name
    elif signature_type == 2:
        # Name with company
        return f"{name}\n{company}"
    else:
        # Name with title and company
        titles = ["Manager", "Director", "Specialist", "Coordinator", "Consultant", "Advisor", "Lead", "Associate"]
        departments = ["Marketing", "Sales", "Operations", "Support", "Development", "Customer Success", "Business", "Product"]
        title = f"{random.choice(departments)} {random.choice(titles)}"
        return f"{name}\n{title}\n{company}"

DEFAULT_DOMAINS_PATH = "domain_list.txt"  # only if you still use it

def load_domains(file_path=DEFAULT_DOMAINS_PATH):
    """Load domain list from file (one per line)."""
    domains = []
    try:
        with open(file_path, "r") as f:
            for line in f:
                d = line.strip()
                if d and not d.startswith("#"):
                    domains.append(d)
    except Exception as e:
        logger.error(f"Error loading domains from {file_path}: {e}")
    return domains


def load_free_accounts(file_path=DEFAULT_CONFIG_PATH):
    """Load FREE email accounts from JSON."""
    try:
        with open(file_path, "r") as f:
            return json.load(f)
    except Exception as e:
        logger.error(f"Error loading free accounts from {file_path}: {e}")
        return []


def load_mailcow_accounts(file_path=MAILCOW_CONFIG_PATH):
    """Load Mailcow domain accounts from JSON."""
    try:
        with open(file_path, "r") as f:
            return json.load(f)
    except Exception as e:
        logger.error(f"Error loading mailcow accounts from {file_path}: {e}")
        return []


def build_mailcow_inboxes(domain_list, templates=MAILCOW_TEMPLATES, password=DEFAULT_PASSWORD):
    """
    OPTIONAL generator if you want to build mailcow accounts from domains.txt.
    If you're loading mailcow_accounts.json, you can delete this.
    """
    inboxes = []
    for domain in domain_list:
        for template in templates:
            addr = template.format(domain=domain)
            inboxes.append({
                "email": addr,
                "password": password,
                "smtp": MAILCOW_SMTP_HOST,
                "imap": MAILCOW_IMAP_HOST,
                "provider": "mailcow"
            })
    return inboxes


def detect_provider(email_addr):
    """
    Returns only: 'free' or 'mailcow'
    FREE list is explicit. Everything else = mailcow.
    """
    e = (email_addr or "").lower()
    free_domains = ("@gmail.com", "@yahoo.com", "@outlook.com", "@hotmail.com", "@icloud.com")
    if any(fd in e for fd in free_domains):
        return "free"
    return "mailcow"

# === EMAIL GENERATION FUNCTIONS ===

def generate_email_subject(recipient_first_name):
    safe_name = (recipient_first_name or "there").split()[0].title()
    return random.choice(SUBJECTS).format(name=safe_name)

def generate_email_body(sender_email, recipient_email, recipient_first_name=None):
    """
    Body rules:
    - Greeting uses FIRST NAME only per get_effective_recipient_first_name().
    - Signature fixed per provider.
    - FREE senders greet Andre, and can mention their own first name in body naturally.
    """
    effective_recipient_name = get_effective_recipient_first_name(
        sender_email, recipient_email, recipient_first_name
    )

    _, signature = get_sender_identity(sender_email)

    greeting_template = random.choice(GREETINGS)
    topic = random.choice(TOPICS)
    main_paragraph = random.choice(BODY_PARAGRAPHS).format(topic=topic)
    question = random.choice(QUESTIONS)
    closing = random.choice(CLOSINGS)

    greeting = greeting_template.format(
        name=effective_recipient_name,
        day_of_week=get_day_of_week()
    )

    if random.random() < 0.7:
        second_paragraph = random.choice(BODY_PARAGRAPHS).format(topic=topic)
        body_core = (
            f"{greeting}\n\n"
            f"{main_paragraph}\n\n"
            f"{second_paragraph}\n\n"
            f"{question}\n\n"
            f"{closing}"
        )
    else:
        body_core = (
            f"{greeting}\n\n"
            f"{main_paragraph}\n\n"
            f"{question}\n\n"
            f"{closing}"
        )

    return f"{body_core}\n{signature}" if signature else body_core

def send_email(sender, recipient):
    """Single-attempt send. No retries. Keeps rotation moving."""
    try:
        sender_email = sender["email"]
        recipient_email = recipient["email"]

        recip_fn_for_subject = get_effective_recipient_first_name(
            sender_email, recipient_email, recipient.get("first_name")
        )

        subject = generate_email_subject(recip_fn_for_subject)
        body = generate_email_body(
            sender_email,
            recipient_email,
            recipient_first_name=recipient.get("first_name")
        )

        wrapped_body = "\n".join(
            textwrap.fill(line, width=70) if line.strip() else line
            for line in body.split("\n")
        )

        from_name, _ = get_sender_identity(sender_email)

        msg = MIMEText(wrapped_body)
        msg["Subject"] = subject
        # HARD RULE: VMIN From never shows email. FREE From is "Andre".
        msg["From"] = from_name
        msg["To"] = recipient_email

        msg["Message-ID"] = (
            f"<{random.randint(10000000, 99999999)}."
            f"{random.randint(1000000, 9999999)}@{sender_email.split('@')[1]}>"
        )

        server = smtplib.SMTP(sender["smtp"], DEFAULT_SMTP_PORT)
        server.starttls()
        server.login(sender_email, sender["password"])
        server.send_message(msg)
        server.quit()

        logger.info(f"[✅ SENT] {sender_email} → {recipient_email} (to: {recip_fn_for_subject}) | {subject}")
        return True

    except Exception as e:
        logger.error(f"[❌ FAIL] {sender.get('email')} to {recipient.get('email')}: {e}")
        return False

# =========================
# INBOX CHECKING / PROCESSING
# =========================

def check_inbox(account):
    """Check inbox for unread emails and return parsed data."""
    try:
        context = ssl.create_default_context()
        mail = imaplib.IMAP4_SSL(account["imap"], DEFAULT_IMAP_PORT, ssl_context=context)
        mail.login(account["email"], account["password"])
        mail.select("INBOX")

        status, messages = mail.search(None, "UNSEEN")
        if status != "OK":
            logger.error(f"Failed to search inbox: {account['email']}")
            mail.logout()
            return []

        email_ids = messages[0].split()
        processed_emails = []

        for email_id in email_ids:
            status, msg_data = mail.fetch(email_id, "(RFC822)")
            if status != "OK":
                logger.error(f"Failed to fetch email {email_id}: {account['email']}")
                continue

            raw_email = msg_data[0][1]
            email_message = email.message_from_bytes(raw_email)

            sender_header = email_message.get("From", "")
            sender_address = email.utils.parseaddr(sender_header)[1]

            # HARD RULE: first name only
            sender_first_name = (
                extract_first_name_from_email_header(sender_header)
                or extract_first_name_from_email(sender_address)
                or "there"
            )
            sender_first_name = sender_first_name.split()[0].title()

            subject = email_message.get("Subject", "")

            body = ""
            if email_message.is_multipart():
                for part in email_message.walk():
                    if part.get_content_type() == "text/plain":
                        body = part.get_payload(decode=True).decode(errors="ignore")
                        break
            else:
                body = email_message.get_payload(decode=True).decode(errors="ignore")

            processed_emails.append({
                "id": email_id,
                "sender": sender_address,
                "sender_first_name": sender_first_name,
                "subject": subject,
                "body": body
            })

            mail.store(email_id, "+FLAGS", "\\Seen")
            if random.random() < 0.3:
                mail.store(email_id, "+FLAGS", "\\Flagged")

        mail.logout()
        return processed_emails

    except Exception as e:
        logger.error(f"Error checking inbox for {account['email']}: {e}")
        return []


def process_and_reply(account, emails, all_accounts):
    """Process received emails and reply to some."""
    for email_data in emails:
        if random.random() < 0.7:
            sender_address = email_data["sender"]
            recipient = next((acc for acc in all_accounts if acc["email"] == sender_address), None)
            if not recipient:
                continue

            # HARD RULE: first name only
            recipient_first_name = (
                email_data.get("sender_first_name")
                or extract_first_name_from_email(sender_address)
                or "there"
            )
            recipient_first_name = recipient_first_name.split()[0].title()

            original_subject = email_data.get("subject", "")
            subject = original_subject if original_subject.lower().startswith("re:") else f"Re: {original_subject}"

            body = generate_email_body(
                account["email"],
                recipient["email"],
                recipient_first_name=recipient_first_name
            )

            # Insert quick acknowledgment after greeting
            body_lines = body.split("\n")
            intro_line = f"Thanks for your email about {original_subject.lower()}."
            if len(body_lines) > 2:
                body_lines.insert(2, intro_line)
            body = "\n".join(body_lines)

            from_name, _ = get_sender_identity(account["email"])

            msg = MIMEText(body)
            msg["Subject"] = subject
            msg["From"] = from_name
            msg["To"] = recipient["email"]

            try:
                server = smtplib.SMTP(account["smtp"], DEFAULT_SMTP_PORT)
                server.starttls()
                server.login(account["email"], account["password"])
                server.send_message(msg)
                server.quit()

                logger.info(f"[✅ REPLY] {account['email']} → {recipient['email']} (to: {recipient_first_name}) | {subject}")
            except Exception as e:
                logger.error(f"[❌ FAIL REPLY] {account['email']} to {recipient['email']}: {e}")

# =========================
# MAIN WARMUP FUNCTIONS
# =========================

def select_sender_recipient_pair(all_accounts):
    """
    Pairing rules:
    - FREE NEVER sends to FREE.
    - MAILCOW NEVER sends to MAILCOW.
    - FREE sends ONLY to MAILCOW.
    - MAILCOW can send to FREE.
    """
    free_accounts = [a for a in all_accounts if detect_provider(a["email"]) == "free"]
    mailcow_accounts = [a for a in all_accounts if detect_provider(a["email"]) == "mailcow"]

    provider_weights = {
        "free-mailcow": 50,
        "mailcow-free": 50,
    }

    weighted_pairs = []
    if free_accounts and mailcow_accounts:
        weighted_pairs.extend(["free-mailcow"] * provider_weights["free-mailcow"])
        weighted_pairs.extend(["mailcow-free"] * provider_weights["mailcow-free"])

    # Fallback if one pool is empty
    if not weighted_pairs:
        sender = random.choice(all_accounts)
        recipient_pool = [a for a in all_accounts if a["email"] != sender["email"]]
        recipient = random.choice(recipient_pool) if recipient_pool else None
        return sender, recipient

    pair = random.choice(weighted_pairs)
    sender_provider, recipient_provider = pair.split("-")

    sender_pool = free_accounts if sender_provider == "free" else mailcow_accounts
    recipient_pool = mailcow_accounts if recipient_provider == "mailcow" else free_accounts

    sender = random.choice(sender_pool)
    recipient_pool = [a for a in recipient_pool if a["email"] != sender["email"]]
    recipient = random.choice(recipient_pool) if recipient_pool else None

    return sender, recipient


def warmup_loop():
    """Main warmup loop with natural timing."""
    free_accounts = load_free_accounts(DEFAULT_CONFIG_PATH)
    mailcow_accounts = load_mailcow_accounts(MAILCOW_CONFIG_PATH)

    # OPTIONAL: if you want to regenerate from domains.txt instead of JSON:
    # domains = load_domains(DEFAULT_DOMAINS_PATH)
    # mailcow_accounts = build_mailcow_inboxes(domains)

    all_accounts = mailcow_accounts + free_accounts

    logger.info(f"Loaded {len(free_accounts)} free accounts and {len(mailcow_accounts)} mailcow accounts")

    stats = {
        "emails_sent": 0,
        "emails_received": 0,
        "errors": 0,
        "provider_distribution": defaultdict(int)
    }

    try:
        while True:
            sender, recipient = select_sender_recipient_pair(all_accounts)

            if not sender or not recipient:
                logger.error("Failed to select valid sender-recipient pair")
                time.sleep(random.randint(10, 30))
                continue

            success = send_email(sender, recipient)

            if success:
                stats["emails_sent"] += 1
                sp = detect_provider(sender["email"])
                rp = detect_provider(recipient["email"])
                stats["provider_distribution"][f"{sp}-{rp}"] += 1
            else:
                stats["errors"] += 1

            # Randomly check inboxes
            if random.random() < 0.3:
                check_account = random.choice(all_accounts)
                emails = check_inbox(check_account)

                if emails:
                    stats["emails_received"] += len(emails)
                    process_and_reply(check_account, emails, all_accounts)

            if stats["emails_sent"] % 10 == 0 and stats["emails_sent"] > 0:
                logger.info(
                    f"Stats: {stats['emails_sent']} sent, "
                    f"{stats['emails_received']} received, "
                    f"{stats['errors']} errors"
                )
                logger.info(f"Provider distribution: {dict(stats['provider_distribution'])}")

            delay = random.randint(12, 27)
            logger.info(f"Waiting {delay} seconds until next email...")
            time.sleep(delay)

    except KeyboardInterrupt:
        logger.info("Warmup loop interrupted by user")
    except Exception as e:
        logger.error(f"Error in warmup loop: {e}")

# =========================
# ENTRY POINT
# =========================

if __name__ == "__main__":
    logger.info("Starting Mailcow Email Warmup — First-Name-Only + Branded Identity")
    warmup_loop()
