import tkinter as tk
from tkinter import messagebox
from datetime import datetime
import sqlite3
import os

class PerpustakaanApp:
    def __init__(self, master):
        self.master = master
        master.title("Sistem Peminjaman Buku")
        master.geometry("800x650")
        master.configure(bg="#121212")

        self.buku_tersedia = [
            "Sejarah Dunia Kuno",
            "Perang Dunia II",
            "Kerajaan Nusantara",
            "Revolusi Industri",
            "Peradaban Mesir Kuno",
            "Indonesia Merdeka"
        ]
        self.buku_dipilih = None

        self.setup_database()

        # Header
        self.header = tk.Label(master, text="Perpustakaan Digital - Buku Sejarah", bg="#121212", fg="#ffffff",
                               font=("Helvetica", 20, "bold"))
        self.header.pack(pady=20, anchor="w", padx=20)

        # Form Frame
        self.form_frame = tk.Frame(master, bg="#1f1f1f", padx=20, pady=20)
        self.form_frame.pack(padx=20, fill="x")

        tk.Label(self.form_frame, text="Nama Peminjam", bg="#1f1f1f", fg="white", font=("Arial", 10, "bold")).pack(anchor="w")
        self.entry_nama = tk.Entry(self.form_frame, width=40, font=("Arial", 10))
        self.entry_nama.pack(pady=5, anchor="w")
        self.entry_nama.insert(0, "Muhammad Aldi")

        tk.Label(self.form_frame, text="Pilih Buku Sejarah", bg="#1f1f1f", fg="white", font=("Arial", 10, "bold")).pack(anchor="w", pady=(10, 5))
        self.grid_buku_frame = tk.Frame(self.form_frame, bg="#1f1f1f")
        self.grid_buku_frame.pack(anchor="w")

        for index, buku in enumerate(self.buku_tersedia):
            row = index // 2
            col = index % 2
            btn = self.bulatkan_tombol(self.grid_buku_frame, buku, lambda b=buku: self.pilih_buku(b))
            btn.grid(row=row, column=col, padx=5, pady=5, sticky="w")

        self.button_frame = tk.Frame(master, bg="#121212")
        self.button_frame.pack(pady=15, anchor="w", padx=20)

        self.btn_pinjam = self.bulatkan_tombol(self.button_frame, "Pinjam Buku", self.pinjam_buku, "#4caf50")
        self.btn_pinjam.pack(anchor="w", pady=5)

        self.btn_kembali = self.bulatkan_tombol(self.button_frame, "Kembalikan Buku", self.kembalikan_buku, "#e53935")
        self.btn_kembali.pack(anchor="w", pady=5)

        self.main_content = tk.Frame(master, bg="#121212")
        self.main_content.pack(fill="both", expand=True, padx=20)

        self.label_daftar = tk.Label(self.main_content, text="Daftar Peminjaman", bg="#121212", fg="white",
                                     font=("Arial", 12, "bold"))
        self.label_daftar.grid(row=0, column=1, sticky="e", pady=(10, 0), padx=(10, 0))

        self.listbox_peminjaman = tk.Listbox(self.main_content, width=50, height=10, font=("Arial", 10), bg="#ffffff", justify="right")
        self.listbox_peminjaman.grid(row=1, column=1, sticky="e", padx=(10, 0), pady=5)

        tk.Label(self.main_content, bg="#121212").grid(row=0, column=0, rowspan=2, sticky="nsew", padx=10)
        self.main_content.grid_columnconfigure(0, weight=1)
        self.main_content.grid_columnconfigure(1, weight=1)

        self.load_data()

    def setup_database(self):
        self.conn = sqlite3.connect("perpustakaan.db")
        self.cursor = self.conn.cursor()
        self.cursor.execute('''
            CREATE TABLE IF NOT EXISTS peminjaman (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nama TEXT NOT NULL,
                buku TEXT NOT NULL,
                tanggal TEXT NOT NULL
            )
        ''')
        self.conn.commit()

    def load_data(self):
        self.listbox_peminjaman.delete(0, tk.END)
        self.cursor.execute("SELECT nama, buku, tanggal FROM peminjaman")
        for nama, buku, tanggal in self.cursor.fetchall():
            data = f"{nama} meminjam '{buku}' pada {tanggal}"
            self.listbox_peminjaman.insert(tk.END, data)

    def bulatkan_tombol(self, parent, text, command, warna="#007acc"):
        return tk.Button(parent, text=text, command=command,
                         bg=warna, fg="white", font=("Arial", 9, "bold"),
                         relief="flat", bd=0, padx=15, pady=10,
                         highlightthickness=0, cursor="hand2", anchor="w", justify="left")

    def pilih_buku(self, buku):
        self.buku_dipilih = buku
        messagebox.showinfo("Buku Dipilih", f"Kamu memilih buku: {buku}")

    def pinjam_buku(self):
        nama = self.entry_nama.get().strip()
        if not nama:
            messagebox.showwarning("Peringatan", "Nama peminjam tidak boleh kosong.")
            return
        if not self.buku_dipilih:
            messagebox.showwarning("Peringatan", "Silakan pilih buku terlebih dahulu.")
            return

        tanggal_pinjam = datetime.now().strftime("%d-%m-%Y %H:%M")

        # Cek apakah sudah dipinjam
        self.cursor.execute("SELECT * FROM peminjaman WHERE nama=? AND buku=?", (nama, self.buku_dipilih))
        if self.cursor.fetchone():
            messagebox.showinfo("Info", "Buku ini sudah dipinjam oleh orang yang sama.")
            return

        self.cursor.execute("INSERT INTO peminjaman (nama, buku, tanggal) VALUES (?, ?, ?)",
                            (nama, self.buku_dipilih, tanggal_pinjam))
        self.conn.commit()

        data = f"{nama} meminjam '{self.buku_dipilih}' pada {tanggal_pinjam}"
        self.listbox_peminjaman.insert(tk.END, data)
        self.buku_dipilih = None

    def kembalikan_buku(self):
        selected_index = self.listbox_peminjaman.curselection()
        if not selected_index:
            messagebox.showwarning("Peringatan", "Pilih peminjaman yang ingin dikembalikan.")
            return

        data = self.listbox_peminjaman.get(selected_index)
        self.listbox_peminjaman.delete(selected_index)

        try:
            nama = data.split(" meminjam ")[0]
            buku = data.split("'")[1]
        except IndexError:
            messagebox.showerror("Error", "Format data tidak dikenali.")
            return

        self.cursor.execute("DELETE FROM peminjaman WHERE nama=? AND buku=?", (nama, buku))
        self.conn.commit()
        messagebox.showinfo("Buku Dikembalikan", f"{data} telah dikembalikan.")

# Jalankan aplikasi
if __name__ == "__main__":
    root = tk.Tk()
    app = PerpustakaanApp(root)
    root.mainloop()
