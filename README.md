import sys
import os
import subprocess
import tempfile
import time
import threading
import tkinter as tk
from tkinter import ttk, messagebox

class TextWithLineNumbers(tk.Frame):
    """محرر أكواد متطور ومستجيب بشكل كامل"""
    def __init__(self, master, **kwargs):
        super().__init__(master, bg="#161e2e")
        
        # شريط أرقام الأسطر
        self.line_numbers = tk.Text(
            self, width=4, padx=5, takefocus=0, border=0,
            background="#0f172a", foreground="#475569",
            state="disabled", font=("Consolas", 11)
        )
        self.line_numbers.pack(side="left", fill="y")

        # منطقة كتابة الكود
        self.text_editor = tk.Text(
            self, font=("Consolas", 11), bg="#030712", fg="#e2e8f0",
            insertbackground="#00f2fe", selectbackground="#334155",
            bd=0, undo=True, wrap="none", **kwargs
        )
        self.text_editor.pack(side="right", fill="both", expand=True)

        self.text_editor.bind("<KeyRelease>", self.update_line_numbers)
        self.text_editor.bind("<Button-1>", self.update_line_numbers)
        self.update_line_numbers()

    def update_line_numbers(self, event=None):
        lines = self.text_editor.get("1.0", "end-1c").split("\n")
        line_count = len(lines)
        line_string = "\n".join(str(i) for i in range(1, line_count + 1))
        
        self.line_numbers.config(state="normal")
        self.line_numbers.delete("1.0", "end")
        self.line_numbers.insert("1.0", line_string)
        self.line_numbers.config(state="disabled")

    def get_code(self):
        return self.text_editor.get("1.0", tk.END)

    def set_code(self, code):
        self.text_editor.delete("1.0", tk.END)
        self.text_editor.insert(tk.END, code)
        self.update_line_numbers()


class PyMasterInteractive:
    def __init__(self, root):
        self.root = root
        self.root.title("PyMaster Interactive Studio - البيئة المتطورة")
        self.root.geometry("1240x800")
        self.root.minsize(1050, 700)
        self.root.configure(bg="#0b0f19")

        self.user_score = 0
        self.completed_lessons = set()

        # ألوان التصميم الداكن
        self.bg_main = "#0b0f19"
        self.card_bg = "#161e2e"
        self.card_light = "#1f293d"
        self.accent_color = "#00f2fe"
        self.success_color = "#4ade80"
        self.text_primary = "#f8fafc"

        # المنهج والدروس
        self.curriculum = {
            "المرحلة 1: البداية المباشرة": [
                {
                    "id": 1,
                    "title": "1.1 طباعة النصوص والترحيب",
                    "explanation": "أهلاً بك في أولى خطواتك لتعلم بايثون!\n\nالدالة print() هي أهم أداة للمبرمج، وظيفتها عرض النصوص أو نتائج الحسابات على الشاشة.\nلكتابة أي نص داخل print، يجب وضعه داخل علامات تنصيص مثل \"نص\" أو 'نص'.\n\nمثال:\nprint(\"Hello World\")",
                    "task": "🎯 التحدي المطلوب:\nقم بكتابة كود يطبع عبارة: Python Is Awesome بالضبط للحصول على 100 نقطة!",
                    "initial_code": 'print("Hello World")\n',
                    "expected_output": "Python Is Awesome",
                    "points": 100
                },
                {
                    "id": 2,
                    "title": "1.2 العمليات الحسابية والرياضيات",
                    "explanation": "تستطيع بايثون إجراء جميع العمليات الرياضية مثل الآلة الحاسبة وأكثر!\n\n- الجمع: +\n- الطرح: -\n- الضرب: *\n- القسمة: /\n\nعند طباعة ناتج عملية حسابية، لا نضع أرقام المعاملات داخل علامات تنصيص.",
                    "task": "🎯 التحدي المطلوب:\nاكتب كود يطبع ناتج ضرب 12 في 5 مباشرة ليتم تقييمك!",
                    "initial_code": 'print(12 * 5)\n',
                    "expected_output": "60",
                    "points": 100
                }
            ],
            "المرحلة 2: المتغيرات والقرارات": [
                {
                    "id": 3,
                    "title": "2.1 المتغيرات (Variables)",
                    "explanation": "المتغير هو مثل صندوق مخزن في ذاكرة الكمبيوتر نضع فيه البيانات لنستخدمها لاحقاً.\n\nتخزين النص:\nname = \"Ahmed\"\n\nتخزين الرقم:\nage = 20\n\nويمكننا طباعة المتغير مباشرة باسمه بدون علامات تنصيص.",
                    "task": "🎯 التحدي المطلوب:\nقم بإنشاء متغير اسمه score وقيمته الرقمية 50، ثم قم بطباعة المتغير score.",
                    "initial_code": 'score = 50\nprint(score)\n',
                    "expected_output": "50",
                    "points": 150
                },
                {
                    "id": 4,
                    "title": "2.2 الجمل الشرطية (If Condition)",
                    "explanation": "نستخدم if لجعل البرنامج يتخذ قراراً بناءً على شرط معين.\n\nمثال:\nx = 10\nif x > 5:\n    print(\"صحيح\")\n\nلاحظ وجود المسافة البادئة (Indentation) قبل print!",
                    "task": "🎯 التحدي المطلوب:\nأنشئ متغير speed بقيمة 100، وإذا كانت speed أكبر من 80 اطبـع كلمة: Fast",
                    "initial_code": 'speed = 100\nif speed > 80:\n    print("Fast")\n',
                    "expected_output": "Fast",
                    "points": 150
                }
            ]
        }

        self.setup_ui()

    def setup_ui(self):
        # 1. الشريط العلوي
        header = tk.Frame(self.root, bg=self.card_bg, height=65)
        header.pack(fill="x", side="top")
        
        logo = tk.Label(header, text="⚡ PyMaster Interactive Studio", font=("Segoe UI", 15, "bold"), bg=self.card_bg, fg=self.accent_color, padx=15)
        logo.pack(side="left")

        self.score_label = tk.Label(header, text="🏆 النقاط: 0 | الدروس المكتملة: 0", font=("Segoe UI", 10, "bold"), bg="#1e293b", fg=self.success_color, padx=12, pady=6)
        self.score_label.pack(side="right", padx=15)

        # 2. قائمة المراحل العلوية
        nav_bar = tk.Frame(self.root, bg="#0f172a", height=40)
        nav_bar.pack(fill="x")

        for phase_name in self.curriculum.keys():
            btn_phase = tk.Button(
                nav_bar, text=f"📍 {phase_name}", font=("Segoe UI", 9, "bold"),
                bg="#1e293b", fg=self.text_primary, bd=0, padx=15, cursor="hand2",
                command=lambda p=phase_name: self.filter_by_phase(p)
            )
            btn_phase.pack(side="left", padx=5, pady=5)

        # 3. الحاوية الرئيسية
        main_box = tk.Frame(self.root, bg=self.bg_main)
        main_box.pack(fill="both", expand=True, padx=12, pady=12)

        # القائمة الجانبية
        sidebar = tk.Frame(main_box, bg=self.card_bg, width=280)
        sidebar.pack(side="left", fill="y", padx=(0, 10))

        sb_title = tk.Label(sidebar, text="المسار التعليمي", font=("Segoe UI", 11, "bold"), bg=self.card_bg, fg=self.text_primary, pady=10)
        sb_title.pack(fill="x")

        self.lesson_listbox = tk.Listbox(
            sidebar, bg=self.card_bg, fg=self.text_primary, 
            selectbackground=self.card_light, selectforeground=self.accent_color,
            font=("Segoe UI", 10), bd=0, highlightthickness=0, activestyle="none"
        )
        self.lesson_listbox.pack(fill="both", expand=True, padx=8, pady=8)
        self.lesson_listbox.bind("<<ListboxSelect>>", self.on_lesson_click)

        # اللوحة اليمنى
        right_panel = tk.Frame(main_box, bg=self.bg_main)
        right_panel.pack(side="right", fill="both", expand=True)

        # قسم الشرح
        explanation_card = tk.LabelFrame(
            right_panel, text=" 📖 الشرح التفصيلي للدرس ", 
            font=("Segoe UI", 10, "bold"), bg=self.card_bg, fg=self.accent_color, padx=12, pady=8
        )
        explanation_card.pack(fill="x", pady=(0, 10))

        self.explanation_text = tk.Text(
            explanation_card, font=("Segoe UI", 10), bg=self.card_bg, fg=self.text_primary,
            bd=0, height=4, wrap="word", state="disabled"
        )
        self.explanation_text.pack(fill="x")

        # قسم التحدي والمشروع
        task_card = tk.Frame(right_panel, bg="#1e1b4b", padx=12, pady=8)
        task_card.pack(fill="x", pady=(0, 10))

        self.task_label = tk.Label(task_card, text="", font=("Segoe UI", 9, "bold"), bg="#1e1b4b", fg="#a5b4fc", justify="left", wraplength=700)
        self.task_label.pack(anchor="w")

        # -------------------------------------------------------------------
        #  استخدام PanedWindow لتقسيم محرر الأكواد وشاشة المخرجات بسحب تفاعلي
        # -------------------------------------------------------------------
        paned_workspace = tk.PanedWindow(
            right_panel, orient=tk.VERTICAL, bg=self.bg_main, 
            sashwidth=6, sashrelief=tk.RAISED
        )
        paned_workspace.pack(fill="both", expand=True)

        # --- الجزء العلوي: محرر الأكواد + أزرار التحكم ---
        editor_group = tk.LabelFrame(
            paned_workspace, text=" 💻 محرر الأكواد التفاعلي ", 
            font=("Segoe UI", 10, "bold"), bg=self.card_bg, fg=self.accent_color, padx=10, pady=8
        )

        toolbar = tk.Frame(editor_group, bg=self.card_bg)
        toolbar.pack(fill="x", pady=(0, 5))

        btn_run_free = tk.Button(
            toolbar, text="▶ تشغيل الكود (Run Code)", 
            bg="#10b981", fg="white", font=("Segoe UI", 9, "bold"), 
            bd=0, padx=15, pady=4, cursor="hand2", command=self.execute_python_code
        )
        btn_run_free.pack(side="left", padx=(0, 5))

        btn_clear = tk.Button(
            toolbar, text="🗑️ مسح الشاشة", 
            bg="#334155", fg="white", font=("Segoe UI", 9), 
            bd=0, padx=10, pady=4, cursor="hand2", command=self.clear_output
        )
        btn_clear.pack(side="left")

        btn_run_eval = tk.Button(
            toolbar, text="🚀 تسليم وتقييم التحدي (Submit)", 
            bg="#0284c7", fg="white", font=("Segoe UI", 9, "bold"), 
            bd=0, padx=15, pady=4, cursor="hand2", command=self.evaluate_submission
        )
        btn_run_eval.pack(side="right")

        self.editor = TextWithLineNumbers(editor_group)
        self.editor.pack(fill="both", expand=True)

        paned_workspace.add(editor_group, minsize=200)

        # --- الجزء السفلي: شاشة المخرجات (Terminal) ---
        output_group = tk.LabelFrame(
            paned_workspace, text=" 🖥️ شاشة المخرجات (Terminal Window) ", 
            font=("Segoe UI", 10, "bold"), bg=self.card_bg, fg=self.success_color, padx=10, pady=5
        )

        self.output_terminal = tk.Text(
            output_group, font=("Consolas", 11), bg="#000000", fg="#4ade80", 
            bd=0, state="disabled"
        )
        self.output_terminal.pack(fill="both", expand=True)

        paned_workspace.add(output_group, minsize=120)

        # تحميل البيانات الأولوية
        self.reload_lesson_tree()
        self.load_lesson_by_id(1)

    def reload_lesson_tree(self):
        self.lesson_listbox.delete(0, tk.END)
        self.flat_lessons = []

        for phase, lessons in self.curriculum.items():
            for idx, lesson in enumerate(lessons):
                self.flat_lessons.append(lesson)
                
                is_unlocked = (lesson["id"] == 1) or ((lesson["id"] - 1) in self.completed_lessons)
                
                status_icon = "✅" if lesson["id"] in self.completed_lessons else ("🔓" if is_unlocked else "🔒")
                display_text = f"{status_icon} {lesson['title']}"
                self.lesson_listbox.insert(tk.END, display_text)

    def filter_by_phase(self, phase_name):
        lessons = self.curriculum[phase_name]
        first_lesson_id = lessons[0]["id"]
        
        if first_lesson_id != 1 and (first_lesson_id - 1) not in self.completed_lessons:
            messagebox.showwarning("مغلق 🔒", "يجب عليك إنهاء الدروس السابقة أولاً لتنفتح هذه المرحلة!")
            return
        
        self.load_lesson_by_id(first_lesson_id)

    def on_lesson_click(self, event):
        selection = self.lesson_listbox.curselection()
        if not selection:
            return
        
        index = selection[0]
        selected_lesson = self.flat_lessons[index]
        
        if selected_lesson["id"] != 1 and (selected_lesson["id"] - 1) not in self.completed_lessons:
            messagebox.showwarning("درس مغلق 🔒", "عذراً! يجب عليك النجاح في التحدي الخاص بالدرس السابق أولاً.")
            return

        self.load_lesson_by_id(selected_lesson["id"])

    def load_lesson_by_id(self, lesson_id):
        for lesson in self.flat_lessons:
            if lesson["id"] == lesson_id:
                self.current_lesson = lesson
                break

        self.task_label.config(text=self.current_lesson["task"])
        self.editor.set_code(self.current_lesson["initial_code"])
        self.start_typewriter_animation(self.current_lesson["explanation"])

    def start_typewriter_animation(self, text_to_type):
        self.explanation_text.config(state="normal")
        self.explanation_text.delete("1.0", tk.END)

        def type_effect():
            for char in text_to_type:
                self.explanation_text.insert(tk.END, char)
                self.explanation_text.see(tk.END)
                time.sleep(0.01)
            self.explanation_text.config(state="disabled")

        threading.Thread(target=type_effect, daemon=True).start()

    def run_code_in_subprocess(self, code):
        """تشغيل الكود عبر عملية بايثون منفصلة وتجميع المخرجات"""
        with tempfile.NamedTemporaryFile(mode="w", suffix=".py", delete=False, encoding="utf-8") as temp_file:
            temp_file.write(code)
            temp_path = temp_file.name

        try:
            result = subprocess.run(
                [sys.executable, temp_path],
                capture_output=True,
                text=True,
                timeout=5,
                encoding="utf-8"
            )
            stdout = result.stdout
            stderr = result.stderr
        except subprocess.TimeoutExpired:
            stdout = ""
            stderr = "❌ أخذ الكود وقتاً طويلاً جداً للرد (توقف بسبب Timeout)!"
        except Exception as e:
            stdout = ""
            stderr = f"❌ خطأ في نظام التشغيل: {e}"
        finally:
            if os.path.exists(temp_path):
                os.remove(temp_path)

        return stdout, stderr

    def execute_python_code(self):
        """تشغيل الكود بحرية وإظهار النتيجة فوراً في شاشة المخرجات المقسمة"""
        user_code = self.editor.get_code()
        
        stdout, stderr = self.run_code_in_subprocess(user_code)

        self.output_terminal.config(state="normal")
        self.output_terminal.delete("1.0", tk.END)

        if stderr:
            self.output_terminal.config(fg="#f87171")
            self.output_terminal.insert(tk.END, f"> Output (Error):\n{stderr}")
        else:
            self.output_terminal.config(fg="#4ade80")
            out_text = stdout if stdout.strip() != "" else "[تم تنفيذ الكود بنجاح دون أي مخرجات]"
            self.output_terminal.insert(tk.END, f"> Output:\n{out_text}")

        self.output_terminal.config(state="disabled")

    def clear_output(self):
        self.output_terminal.config(state="normal")
        self.output_terminal.delete("1.0", tk.END)
        self.output_terminal.config(state="disabled")

    def evaluate_submission(self):
        """تقييم إجابة الطالب بالنسبة للتحدي المطلوب"""
        user_code = self.editor.get_code()
        stdout, stderr = self.run_code_in_subprocess(user_code)

        self.output_terminal.config(state="normal")
        self.output_terminal.delete("1.0", tk.END)

        if stderr:
            self.output_terminal.config(fg="#f87171")
            self.output_terminal.insert(tk.END, f"> خطأ أثناء التشغيل:\n{stderr}")
            self.output_terminal.config(state="disabled")
            return

        result = stdout.strip()
        expected = self.current_lesson["expected_output"].strip()

        self.output_terminal.insert(tk.END, f"> ناتج كودك:\n{result}\n")

        if result == expected:
            if self.current_lesson["id"] not in self.completed_lessons:
                self.completed_lessons.add(self.current_lesson["id"])
                self.user_score += self.current_lesson["points"]

            self.output_terminal.insert(tk.END, f"\n🎉 ممتاز جداً! الإجابة صحيحة. (+{self.current_lesson['points']} نقطة)")
            self.output_terminal.config(fg="#4ade80")
            
            self.score_label.config(text=f"🏆 النقاط: {self.user_score} | الدروس المكتملة: {len(self.completed_lessons)}")
            self.reload_lesson_tree()
            
            messagebox.showinfo("نجاح! 🌟", f"رائع! أتممت التحدي بنجاح وانفتحت لك المرحلة التالية.\nحصلت على {self.current_lesson['points']} نقطة!")
        else:
            self.output_terminal.insert(tk.END, f"\n❌ المخرج غير مطابِق للتحدي.\nالمطلوب طباعته بالضبط: '{expected}'")
            self.output_terminal.config(fg="#f87171")

        self.output_terminal.config(state="disabled")


if __name__ == "__main__":
    root = tk.Tk()
    app = PyMasterInteractive(root)
    root.mainloop()
