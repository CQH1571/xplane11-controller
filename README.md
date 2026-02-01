import tkinter as tk
from tkinter import ttk, filedialog, messagebox, scrolledtext
from tkinter import font as tkfont
import sqlite3
import json
import os
import threading
import time
from datetime import datetime, timedelta
import requests
import re

# ================= 配置层 =================
class Config:
    APP_NAME = "AI学科助教"
    VERSION = "1.2.0"
    SUBJECTS = {
        'math': '数学', 'physics': '物理', 'chemistry': '化学',
        'english': '英语', 'chinese': '语文', 'biology': '生物',
        'geography': '地理', 'history': '历史', 'morality': '道法'
    }
    SUBJECT_ICONS = {
        'math': '🔢', 'physics': '⚛️', 'chemistry': '🧪',
        'english': '🔤', 'chinese': '📖', 'biology': '🧬',
        'geography': '🌍', 'history': '📜', 'morality': '⚖️'
    }
    SUBJECT_COLORS = {
        'math': '#4CAF50', 'physics': '#2196F3', 'chemistry': '#9C27B0',
        'english': '#FF9800', 'chinese': '#F44336', 'biology': '#4CAF50',
        'geography': '#009688', 'history': '#795548', 'morality': '#3F51B5'
    }
    DB_FILE = "study_assistant.db"
    THEMES = {
        'light': {
            'bg': '#F0F2F5', 'fg': '#333333', 'primary': '#2196F3',
            'card_bg': '#FFFFFF', 'text': '#212121', 'border': '#E0E0E0',
            'success': '#4CAF50', 'warning': '#FF9800', 'error': '#F44336'
        }
    }

# ================= 数据库层 =================
class Database:
    def __init__(self, db_file=Config.DB_FILE):
        self.db_file = db_file
        self.init_db()

    def init_db(self):
        with sqlite3.connect(self.db_file) as conn:
            cursor = conn.cursor()
            cursor.execute('''CREATE TABLE IF NOT EXISTS questions (
                id INTEGER PRIMARY KEY AUTOINCREMENT, subject TEXT, 
                question_text TEXT, answer_text TEXT, is_correct INTEGER DEFAULT 1,
                difficulty INTEGER DEFAULT 3, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
            cursor.execute('''CREATE TABLE IF NOT EXISTS statistics (
                id INTEGER PRIMARY KEY AUTOINCREMENT, date DATE UNIQUE, 
                questions_count INTEGER DEFAULT 0, correct_count INTEGER DEFAULT 0)''')
            cursor.execute('''CREATE TABLE IF NOT EXISTS settings (
                key TEXT UNIQUE, value TEXT)''')
            conn.commit()

    def add_question(self, subject, question, answer, is_correct=True):
        with sqlite3.connect(self.db_file) as conn:
            cursor = conn.cursor()
            cursor.execute('INSERT INTO questions (subject, question_text, answer_text, is_correct) VALUES (?,?,?,?)',
                           (subject, question, answer, 1 if is_correct else 0))
            today = datetime.now().strftime('%Y-%m-%d')
            cursor.execute('''INSERT INTO statistics (date, questions_count, correct_count) VALUES (?, 1, ?)
                              ON CONFLICT(date) DO UPDATE SET questions_count=questions_count+1, 
                              correct_count=correct_count+?''', (today, 1 if is_correct else 0, 1 if is_correct else 0))
            conn.commit()

    def get_today_stats(self):
        today = datetime.now().strftime('%Y-%m-%d')
        with sqlite3.connect(self.db_file) as conn:
            cursor = conn.cursor()
            cursor.execute('SELECT questions_count, correct_count FROM statistics WHERE date=?', (today,))
            res = cursor.fetchone()
            return res if res else (0, 0)

    def save_setting(self, key, value):
        with sqlite3.connect(self.db_file) as conn:
            conn.execute('INSERT OR REPLACE INTO settings VALUES (?, ?)', (key, str(value)))

    def load_setting(self, key, default=None):
        with sqlite3.connect(self.db_file) as conn:
            res = conn.execute('SELECT value FROM settings WHERE key=?', (key,)).fetchone()
            return res[0] if res else default

# ================= AI 服务层 =================
class AIService:
    def __init__(self, api_key=None):
        self.api_key = api_key
        self.api_url = "https://open.bigmodel.cn/api/paas/v4/chat/completions"

    def ask_question(self, subject, question):
        if not self.api_key:
            time.sleep(1) # 模拟网络延迟
            return f"【模拟回答】针对{Config.SUBJECTS[subject]}问题：{question}\n请在设置中配置API Key以获取真实AI解析。"

        headers = {"Authorization": f"Bearer {self.api_key}", "Content-Type": "application/json"}
        data = {
            "model": "glm-4",
            "messages": [
                {"role": "system", "content": f"你是一个{Config.SUBJECTS[subject]}老师，请提供详细的步骤化解答。"},
                {"role": "user", "content": question}
            ]
        }
        try:
            resp = requests.post(self.api_url, headers=headers, json=data, timeout=15)
            return resp.json()['choices'][0]['message']['content']
        except Exception as e:
            return f"请求失败: {str(e)}"

# ================= UI 界面层 =================
class AIStudyAssistant:
    def __init__(self, root):
        self.root = root
        self.root.title(f"{Config.APP_NAME} v{Config.VERSION}")
        self.root.geometry("1100x750")
        
        self.db = Database()
        self.current_theme = self.db.load_setting('theme', 'light')
        self.api_key = self.db.load_setting('api_key', '')
        self.ai_service = AIService(self.api_key)
        self.current_subject = 'math'
        
        self.setup_styles()
        self.setup_ui()
        self.update_dashboard()

    def setup_styles(self):
        self.style = ttk.Style()
        theme = Config.THEMES[self.current_theme]
        self.root.configure(bg=theme['bg'])
        
        # 自定义卡片样式
        self.style.configure("Card.TFrame", background=theme['card_bg'], relief="flat")
        self.style.configure("Sidebar.TFrame", background="#2C3E50") # 深色侧边栏
        self.style.configure("TNotebook", background=theme['bg'], borderwidth=0)
        self.style.configure("TNotebook.Tab", padding=[20, 8], font=("微软雅黑", 10))

    def setup_ui(self):
        # 1. 侧边栏
        sidebar = ttk.Frame(self.root, style="Sidebar.TFrame", width=220)
        sidebar.pack(side=tk.LEFT, fill=tk.Y)
        sidebar.pack_propagate(False)

        tk.Label(sidebar, text=Config.APP_NAME, font=("微软雅黑", 16, "bold"), 
                 bg="#2C3E50", fg="white", pady=20).pack()

        # 学科选择器
        subject_group = tk.Frame(sidebar, bg="#2C3E50")
        subject_group.pack(fill=tk.X, padx=10)
        
        self.subject_var = tk.StringVar(value='math')
        for code, name in Config.SUBJECTS.items():
            btn = tk.Radiobutton(subject_group, text=f"{Config.SUBJECT_ICONS[code]} {name}", 
                                 variable=self.subject_var, value=code,
                                 indicatoron=0, bg="#34495E", fg="white", selectcolor=Config.SUBJECT_COLORS[code],
                                 font=("微软雅黑", 10), relief="flat", command=self.on_subject_change,
                                 activebackground="#3e5871", pady=8)
            btn.pack(fill=tk.X, pady=2)

        # 侧边栏底部统计
        self.stats_label = tk.Label(sidebar, text="", font=("微软雅黑", 9), bg="#2C3E50", fg="#BDC3C7", justify=tk.LEFT)
        self.stats_label.pack(side=tk.BOTTOM, pady=20, padx=10)

        # 2. 主内容区
        self.main_area = ttk.Frame(self.root)
        self.main_area.pack(side=tk.RIGHT, expand=True, fill=tk.BOTH, padx=15, pady=15)

        self.notebook = ttk.Notebook(self.main_area)
        self.notebook.pack(expand=True, fill=tk.BOTH)

        self.setup_ask_tab()
        self.setup_settings_tab()

    def setup_ask_tab(self):
        ask_frame = tk.Frame(self.notebook, bg=Config.THEMES[self.current_theme]['bg'])
        self.notebook.add(ask_frame, text=" 💬 智能问答 ")

        # 顶部学科提示
        self.subject_banner = tk.Label(ask_frame, text="当前学科：数学", font=("微软雅黑", 12, "bold"),
                                       bg=Config.SUBJECT_COLORS['math'], fg="white", pady=10)
        self.subject_banner.pack(fill=tk.X, pady=(0, 10))

        # 输入区卡片
        input_card = ttk.Frame(ask_frame, style="Card.TFrame", padding=15)
        input_card.pack(fill=tk.X, padx=10, pady=5)
        
        tk.Label(input_card, text="输入你的问题或题目描述：", font=("微软雅黑", 10), bg="white").pack(anchor="w")
        self.q_input = tk.Text(input_card, height=4, font=("微软雅黑", 11), relief="flat", bg="#F8F9FA", pady=5, padx=5)
        self.q_input.pack(fill=tk.X, pady=10)

        self.ask_btn = tk.Button(input_card, text="✨ 获取AI详细解答", bg=Config.THEMES[self.current_theme]['primary'],
                                fg="white", font=("微软雅黑", 10, "bold"), relief="flat", padx=20, pady=8,
                                command=self.handle_ask_ai, cursor="hand2")
        self.ask_btn.pack(side=tk.RIGHT)

        # 输出区卡片
        output_card = ttk.Frame(ask_frame, style="Card.TFrame", padding=15)
        output_card.pack(expand=True, fill=tk.BOTH, padx=10, pady=15)
        
        tk.Label(output_card, text="AI 老师的解答：", font=("微软雅黑", 10), bg="white").pack(anchor="w")
        self.a_output = scrolledtext.ScrolledText(output_card, font=("微软雅黑", 11), relief="flat", bg="white", padx=5)
        self.a_output.pack(expand=True, fill=tk.BOTH, pady=10)

    def setup_settings_tab(self):
        settings_frame = tk.Frame(self.notebook, bg="white", padding=30)
        self.notebook.add(settings_frame, text=" ⚙️ 系统设置 ")

        tk.Label(settings_frame, text="API Key 配置 (智谱AI/GLM)", font=("微软雅黑", 11, "bold"), bg="white").pack(anchor="w")
        self.api_entry = ttk.Entry(settings_frame, width=50, show="*")
        self.api_entry.insert(0, self.api_key)
        self.api_entry.pack(anchor="w", pady=10)

        save_btn = ttk.Button(settings_frame, text="保存配置", command=self.save_api_settings)
        save_btn.pack(anchor="w", pady=20)

    # ================= 业务逻辑 =================
    def on_subject_change(self):
        self.current_subject = self.subject_var.get()
        color = Config.SUBJECT_COLORS.get(self.current_subject, "#2196F3")
        self.subject_banner.config(text=f"当前学科：{Config.SUBJECTS[self.current_subject]}", bg=color)

    def update_dashboard(self):
        q_count, c_count = self.db.get_today_stats()
        acc = (c_count / q_count * 100) if q_count > 0 else 0
        self.stats_label.config(text=f"📅 今日学习统计\n提问总数：{q_count}\n完成进度：{acc:.1f}%")

    def handle_ask_ai(self):
        question = self.q_input.get("1.0", tk.END).strip()
        if not question: return
        
        self.ask_btn.config(state=tk.DISABLED, text="⌛ AI 正在思考中...")
        self.a_output.delete("1.0", tk.END)
        self.a_output.insert(tk.END, "思考中，请稍后...\n" + "—"*30 + "\n")
        
        # 开启多线程防止界面卡死
        thread = threading.Thread(target=self._run_ai_task, args=(question,))
        thread.start()

    def _run_ai_task(self, question):
        answer = self.ai_service.ask_question(self.current_subject, question)
        # 回到主线程更新UI
        self.root.after(0, lambda: self._update_ui_result(question, answer))

    def _update_ui_result(self, question, answer):
        self.a_output.delete("1.0", tk.END)
        self.a_output.insert(tk.END, answer)
        self.ask_btn.config(state=tk.NORMAL, text="✨ 获取AI详细解答")
        self.db.add_question(self.current_subject, question, answer)
        self.update_dashboard()

    def save_api_settings(self):
        new_key = self.api_entry.get().strip()
        self.db.save_setting('api_key', new_key)
        self.ai_service.api_key = new_key
        messagebox.showinfo("成功", "配置已保存")

if __name__ == "__main__":
    root = tk.Tk()
    # 尝试设置高DPI适配（Windows）
    try:
        from ctypes import windll
        windll.shcore.SetProcessDpiAwareness(1)
    except: pass
    
    app = AIStudyAssistant(root)
    root.mainloop()
