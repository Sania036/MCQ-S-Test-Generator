import tkinter as tk
from tkinter import ttk, messagebox
import random
import nltk
from nltk.tokenize import sent_tokenize, word_tokenize
from nltk import pos_tag

nltk.download('punkt')
nltk.download('averaged_perceptron_tagger')

class MCQGeneratorApp:
    def __init__(self, root):
        self.root = root
        self.root.title("MCQ Generator - NLP Based")
        self.root.geometry("850x700")
        self.root.config(bg="#fdf6e3")

        self.questions = []
        self.user_answers = []
        self.current_question = 0
        self.score = 0

        self.create_widgets()

    def create_widgets(self):
        title = tk.Label(self.root, text="MCQ Generator (NLP-Based)", font=("Helvetica", 20, "bold"),
                         bg="#fdf6e3", fg="#8B4513")
        title.pack(pady=20)

        input_frame = tk.Frame(self.root, bg="#fffaf0", bd=2, relief=tk.RIDGE)
        input_frame.pack(padx=20, pady=10, fill="x")

        tk.Label(input_frame, text="Paste Text Below:", font=("Arial", 12), bg="#fffaf0", fg="#8B4513").pack(anchor="w", padx=10, pady=5)
        self.input_text = tk.Text(input_frame, height=10, font=("Arial", 12), wrap=tk.WORD)
        self.input_text.pack(padx=10, pady=5, fill="x")

        control_frame = tk.Frame(input_frame, bg="#fffaf0")
        control_frame.pack(pady=10)
        tk.Label(control_frame, text="Number of MCQs: ", bg="#fffaf0", fg="#8B4513").pack(side=tk.LEFT)
        self.num_mcqs = ttk.Combobox(control_frame, values=[5, 10, 15, 20, 25, 30], width=5)
        self.num_mcqs.set(5)
        self.num_mcqs.pack(side=tk.LEFT, padx=10)
        ttk.Button(control_frame, text="Generate MCQs", command=self.generate_mcqs).pack(side=tk.LEFT, padx=10)

        # Add Scrollable Question Frame
        container = tk.Frame(self.root)
        container.pack(fill="both", expand=True, padx=20, pady=10)

        canvas = tk.Canvas(container, bg="#f5f5dc")
        scrollbar = ttk.Scrollbar(container, orient="vertical", command=canvas.yview)
        self.q_frame = tk.Frame(canvas, bg="#f5f5dc")

        self.q_frame.bind("<Configure>", lambda e: canvas.configure(scrollregion=canvas.bbox("all")))

        canvas.create_window((0, 0), window=self.q_frame, anchor="nw")
        canvas.configure(yscrollcommand=scrollbar.set)

        canvas.pack(side="left", fill="both", expand=True)
        scrollbar.pack(side="right", fill="y")

        self.canvas = canvas  # Save reference to scroll programmatically if needed

    def generate_mcqs(self):
        text = self.input_text.get("1.0", tk.END).strip()
        try:
            num = int(self.num_mcqs.get())
        except:
            messagebox.showerror("Error", "Invalid number of MCQs selected.")
            return

        if not text or len(text.split()) < 10:
            messagebox.showerror("Error", "Please enter a longer text.")
            return

        sentences = sent_tokenize(text)
        random.shuffle(sentences)

        noun_pool = []
        for sentence in sentences:
            tagged = pos_tag(word_tokenize(sentence))
            for word, tag in tagged:
                if tag.startswith('NN') and word.isalpha():
                    noun_pool.append(word)

        self.questions.clear()
        self.user_answers.clear()
        self.score = 0
        self.current_question = 0

        for sentence in sentences:
            words = word_tokenize(sentence)
            tagged = pos_tag(words)
            blanks = [word for word, tag in tagged if tag.startswith('NN') and word.isalpha()]
            if not blanks:
                continue

            correct_answer = random.choice(blanks)
            blanked_sentence = sentence.replace(correct_answer, "_____")

            distractors = list(set([w for w in noun_pool if w != correct_answer]))
            if len(distractors) < 2:
                continue

            options = random.sample(distractors, 2) + [correct_answer]
            random.shuffle(options)

            correct_index = options.index(correct_answer)

            self.questions.append({
                "question": blanked_sentence,
                "options": options,
                "answer_index": correct_index
            })

            if len(self.questions) >= num:
                break

        if not self.questions:
            messagebox.showinfo("No Questions", "Could not generate MCQs from the given text.")
            return

        self.show_question()

    def show_question(self):
        for widget in self.q_frame.winfo_children():
            widget.destroy()

        if self.current_question >= len(self.questions):
            self.show_results()
            return

        q = self.questions[self.current_question]
        tk.Label(self.q_frame, text=f"Q{self.current_question + 1}: {q['question']}",
                 font=("Arial", 14, "bold"), bg="#f5f5dc", fg="#654321", wraplength=750, justify="left").pack(pady=10)

        self.selected_option = tk.IntVar()

        for idx, option in enumerate(q["options"]):
            rb = tk.Radiobutton(self.q_frame, text=option, variable=self.selected_option, value=idx,
                                font=("Arial", 12), bg="#f5f5dc", fg="#4b3832", anchor="w", wraplength=700, justify="left")
            rb.pack(anchor="w", padx=20, pady=5)

        tk.Button(self.q_frame, text="Next", command=self.next_question,
                  bg="#8B4513", fg="white", font=("Arial", 12, "bold")).pack(pady=10)

    def next_question(self):
        selected = self.selected_option.get()
        q = self.questions[self.current_question]
        self.user_answers.append(selected)

        if selected == q["answer_index"]:
            self.score += 1

        self.current_question += 1
        self.show_question()

    def show_results(self):
        for widget in self.q_frame.winfo_children():
            widget.destroy()

        tk.Label(self.q_frame, text=f"🎯 Your Score: {self.score} out of {len(self.questions)}",
                 font=("Arial", 16, "bold"), bg="#f5f5dc", fg="green").pack(pady=10)

        for i, q in enumerate(self.questions):
            user_ans = self.user_answers[i]
            correct = q["answer_index"]
            color = "#e0ffe0" if user_ans == correct else "#ffe0e0"

            summary = tk.Frame(self.q_frame, bg=color, bd=1, relief=tk.SOLID)
            summary.pack(fill="x", pady=5, padx=5)

            tk.Label(summary, text=f"Q{i + 1}: {q['question']}", font=("Arial", 12, "bold"),
                     bg=summary["bg"], wraplength=750, justify="left").pack(anchor="w", padx=10, pady=3)

            for idx, opt in enumerate(q["options"]):
                tag = ""
                if idx == correct:
                    tag = "✅ Correct"
                elif idx == user_ans:
                    tag = "❌ Your choice"

                display_text = f"{chr(65 + idx)}. {opt} {tag}"
                tk.Label(summary, text=display_text, font=("Arial", 11), bg=summary["bg"],
                         wraplength=730, justify="left").pack(anchor="w", padx=20)

        tk.Button(self.q_frame, text="Restart", command=self.restart, bg="#8B4513",
                  fg="white", font=("Arial", 12, "bold")).pack(pady=10)

    def restart(self):
        self.input_text.delete("1.0", tk.END)
        for widget in self.q_frame.winfo_children():
            widget.destroy()
        self.score = 0
        self.questions = []
        self.user_answers = []
        self.current_question = 0

if __name__ == "__main__":
    root = tk.Tk()
    app = MCQGeneratorApp(root)
    root.mainloop()

# MCQ-S-Test-Generator
