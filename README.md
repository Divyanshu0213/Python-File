import tkinter as tk
from tkinter import messagebox
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
import speech_recognition as sr
import pyttsx3
import feedparser
import threading
from langdetect import detect
from transformers import pipeline

GOOGLE_NEWS_RSS = "https://news.google.com/rss?hl=en-US&gl=US&ceid=US:en"

analyzer = SentimentIntensityAnalyzer()
engine = pyttsx3.init()
fake_news_classifier = None

root = tk.Tk()
root.title("📰 News Sentiment & Myth Detector")
root.geometry("800x600")

dark_mode = False

def toggle_dark_mode():
    global dark_mode
    dark_mode = not dark_mode
    bg = "#2e2e2e" if dark_mode else "SystemButtonFace"
    fg = "white" if dark_mode else "black"
    dark_widgets = [text_input, result_label, details_label, heading, country_dropdown] + list(buttons_frame.children.values())
    for widget in dark_widgets:
        widget.configure(bg=bg, fg=fg)

def capture_speech():
    r = sr.Recognizer()
    try:
        with sr.Microphone() as source:
            set_status("Listening...")
            audio = r.listen(source, timeout=5, phrase_time_limit=10)
            text = r.recognize_google(audio)
            text_input.delete("1.0", tk.END)
            text_input.insert(tk.END, text)
            set_status("Ready")
    except Exception as e:
        messagebox.showerror("Error", f"Speech recognition failed: {e}")
        set_status("Ready")

def check_language(text):
    try:
        lang = detect(text)
        if lang != 'en':
            messagebox.showwarning("Language Warning", f"Detected language is {lang}. This app works best with English.")
            return False
        return True
    except Exception as e:
        messagebox.showerror("Language Error", f"Failed to detect language: {e}")
        return False

def is_myth_bert(text):
    global fake_news_classifier
    if fake_news_classifier is None:
        fake_news_classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")
    candidate_labels = ["fake news", "real news"]
    result = fake_news_classifier(text, candidate_labels)
    return result['labels'][0] == "fake news"

def analyze_sentiment():
    text = text_input.get("1.0", tk.END).strip()
    if not text:
        messagebox.showwarning("Input Required", "Please enter or speak text first.")
        return

    if not check_language(text):
        return

    if is_myth_bert(text):
        result_label.config(text="🧪 Myth (Fake News)", fg="purple")
        details_label.config(text="Detected as Fake News using BERT.")
        engine.say("This is likely fake news.")
        engine.runAndWait()
        return

    scores = analyzer.polarity_scores(text)
    compound = scores['compound']

    if compound >= 0.05:
        sentiment = "😊 Positive"
        result_label.config(fg="green")
    elif compound <= -0.05:
        sentiment = "😠 Negative"
        result_label.config(fg="red")
    else:
        sentiment = "😐 Neutral"
        result_label.config(fg="gray")

    result_label.config(text=f"Sentiment: {sentiment}")
    details_label.config(text=f"Scores:\n{scores}")
    engine.say(f"The sentiment is {sentiment}")
    engine.runAndWait()

def on_text_change(event):
    text = text_input.get("1.0", tk.END).strip()
    if not text:
        result_label.config(text="", fg="black")
        details_label.config(text="")
        return
    scores = analyzer.polarity_scores(text)
    compound = scores['compound']

    if compound >= 0.05:
        sentiment = "😊 Positive"
        result_label.config(fg="green")
    elif compound <= -0.05:
        sentiment = "😠 Negative"
        result_label.config(fg="red")
    else:
        sentiment = "😐 Neutral"
        result_label.config(fg="gray")

    result_label.config(text=f"Live Sentiment: {sentiment}")
    details_label.config(text=f"Live Scores: {scores}")

def clear_text():
    text_input.delete("1.0", tk.END)
    result_label.config(text="")
    details_label.config(text="")

def copy_to_clipboard():
    text = text_input.get("1.0", tk.END).strip()
    sentiment = result_label.cget("text")
    scores = details_label.cget("text")
    result_text = f"{text}\n\n{sentiment}\n{scores}"
    root.clipboard_clear()
    root.clipboard_append(result_text)
    messagebox.showinfo("Copied", "Result copied to clipboard.")

def save_analysis():
    text = text_input.get("1.0", tk.END).strip()
    sentiment = result_label.cget("text")
    scores = details_label.cget("text")
    if not text or not sentiment:
        messagebox.showwarning("Warning", "Nothing to save.")
        return
    try:
        with open("sentiment_analysis_result.txt", "w", encoding="utf-8") as f:
            f.write(f"Text:\n{text}\n\n{sentiment}\n{scores}\n")
        messagebox.showinfo("Saved", "Analysis saved to 'sentiment_analysis_result.txt'")
    except Exception as e:
        messagebox.showerror("File Error", f"Could not save file: {e}")

def fetch_news():
    def fetch():
        try:
            country_code = country_var.get()
            rss_url = f"https://news.google.com/rss?hl=en-{country_code.upper()}&gl={country_code.upper()}&ceid={country_code.upper()}:en"
            feed = feedparser.parse(rss_url)
            if not feed.entries:
                raise ValueError("No news entries found.")

            headlines = [entry.title for entry in feed.entries[:5]]
            news_text = "\n".join(f"• {title}" for title in headlines)

            root.after(0, lambda: text_input.delete("1.0", tk.END))
            root.after(0, lambda: text_input.insert(tk.END, news_text))
            root.after(0, lambda: messagebox.showinfo("Success", "Top Google News headlines fetched."))

        except Exception as e:
            root.after(0, lambda: messagebox.showerror("Error", f"Could not fetch news: {e}"))

    threading.Thread(target=fetch, daemon=True).start()

def set_status(msg):
    status_label.config(text=msg)

heading = tk.Label(root, text="📰 Sentiment & Myth Detection for News & Politics", font=("Arial", 14, "bold"))
heading.pack(pady=10)

country_var = tk.StringVar(value="us")
country_dropdown = tk.OptionMenu(root, country_var, "us", "gb", "in", "ca", "au")
country_dropdown.config(font=("Arial", 10))
country_dropdown.pack()

text_input = tk.Text(root, height=10, width=90, font=("Arial", 10))
text_input.pack(pady=10)
text_input.bind("<KeyRelease>", on_text_change)

buttons_frame = tk.Frame(root)
buttons_frame.pack(pady=10)

tk.Button(buttons_frame, text="Analyze Sentiment", command=analyze_sentiment, font=("Arial", 12), bg="blue", fg="white").grid(row=0, column=0, padx=5)
tk.Button(buttons_frame, text="🎤 Speak Now", command=capture_speech, font=("Arial", 12), bg="green", fg="white").grid(row=0, column=1, padx=5)
tk.Button(buttons_frame, text="Save", command=save_analysis, font=("Arial", 12), bg="orange", fg="white").grid(row=0, column=2, padx=5)
tk.Button(buttons_frame, text="🧹 Clear", command=clear_text, font=("Arial", 12), bg="gray", fg="white").grid(row=0, column=3, padx=5)
tk.Button(buttons_frame, text="📋 Copy", command=copy_to_clipboard, font=("Arial", 12), bg="teal", fg="white").grid(row=0, column=4, padx=5)
tk.Button(buttons_frame, text="Fetch News", command=fetch_news, font=("Arial", 12), bg="purple", fg="white").grid(row=0, column=5, padx=5)
tk.Button(buttons_frame, text="🌙 Toggle Dark Mode", command=toggle_dark_mode, font=("Arial", 12), bg="black", fg="white").grid(row=1, column=0, columnspan=6, pady=10)

result_label = tk.Label(root, text="", font=("Arial", 12))
result_label.pack(pady=5)

details_label = tk.Label(root, text="", font=("Arial", 10))
details_label.pack(pady=5)

status_label = tk.Label(root, text="Ready", font=("Arial", 10), anchor="w")
status_label.pack(fill="x", side="bottom")

root.mainloop()
