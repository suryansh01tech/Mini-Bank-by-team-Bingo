# Mini-Bank-by-team-Bingo
import json
import os
import sys
import time
import hashlib
import secrets
from dataclasses import dataclass, asdict, field
from typing import Dict, List

import tkinter as tk
from tkinter import messagebox, simpledialog

DB_FILE = "bank_data.json"
ADMIN_PASSWORD = "admin123"


def _now():
    return time.strftime("%Y-%m-%d %H:%M:%S")


def hash_pin(pin: str, salt: str):
    return hashlib.sha256((salt + pin).encode()).hexdigest()


@dataclass
class Transaction:
    timestamp: str
    type: str
    amount: float
    description: str
    balance_after: float


@dataclass
class Account:
    acc_no: str
    name: str
    salt: str
    pin_hash: str
    balance: float = 0.0
    transactions: List[Dict] = field(default_factory=list)

    def verify_pin(self, pin: str):
        return self.pin_hash == hash_pin(pin, self.salt)

    def add_transaction(self, t_type: str, amount: float, description: str):
        tx = Transaction(_now(), t_type, amount, description, self.balance)
        self.transactions.append(asdict(tx))


class Bank:
    def __init__(self, db_file=DB_FILE):
        self.db_file = db_file
        self.accounts: Dict[str, Account] = {}
        self._load()

    def _load(self):
        if not os.path.exists(self.db_file):
            self._save()
            return
        try:
            with open(self.db_file, "r") as f:
                data = json.load(f)
            for acc_no, raw in data.get("accounts", {}).items():
                acc = Account(
                    acc_no=raw["acc_no"],
                    name=raw["name"],
                    salt=raw["salt"],
                    pin_hash=raw["pin_hash"],
                    balance=raw.get("balance", 0.0),
                    transactions=raw.get("transactions", []),
                )
                self.accounts[acc_no] = acc
        except:
            pass

    def _save(self):
        data = {"accounts": {acc_no: asdict(acc) for acc_no, acc in self.accounts.items()}}
        with open(self.db_file, "w") as f:
            json.dump(data, f, indent=2)

    def _generate_acc_no(self):
        while True:
            acc_no = str(secrets.randbelow(10**10)).zfill(10)
            if acc_no not in self.accounts:
                return acc_no

    def create_account(self, name: str, pin: str, initial_deposit: float = 0.0):
        acc_no = self._generate_acc_no()
        salt = secrets.token_hex(8)
        pin_hash = hash_pin(pin, salt)
        acc = Account(acc_no, name, salt, pin_hash, 0.0)
        if initial_deposit > 0:
            acc.balance = float(initial_deposit)
            acc.add_transaction("DEPOSIT", initial_deposit, "Initial deposit")
        self.accounts[acc_no] = acc
        self._save()
        return acc

    def get_account(self, acc_no: str):
        return self.accounts.get(acc_no)

    def deposit(self, acc: Account, amount: float):
        if amount <= 0:
            raise ValueError("Deposit amount must be positive")
        acc.balance += float(amount)
        acc.add_transaction("DEPOSIT", amount, "Deposit")
        self._save()

    def withdraw(self, acc: Account, amount: float):
        if amount <= 0:
            raise ValueError("Withdrawal amount must be positive")
        if acc.balance < amount:
            raise ValueError("Insufficient funds")
        acc.balance -= float(amount)
        acc.add_transaction("WITHDRAW", amount, "Withdrawal")
        self._save()

    def transfer(self, from_acc: Account, to_acc_no: str, amount: float):
        if amount <= 0:
            raise ValueError("Transfer amount must be positive")
        to_acc = self.get_account(to_acc_no)
        if to_acc is None:
            raise ValueError("Destination account not found")
        if from_acc.balance < amount:
            raise ValueError("Insufficient funds")
        from_acc.balance -= amount
        from_acc.add_transaction("TRANSFER_OUT", amount, f"To {to_acc_no}")
        to_acc.balance += amount
        to_acc.add_transaction("TRANSFER_IN", amount, f"From {from_acc.acc_no}")
        self._save()

    def close_account(self, acc_no: str):
        if acc_no in self.accounts:
            del self.accounts[acc_no]
            self._save()


bank = Bank()
current_account = None


class App:
    def __init__(self, root):
        self.root = root
        self.root.title("MiniBank GUI")
        self.root.geometry("450x300")
        self.show_main_menu()

    def clear(self):
        for w in self.root.winfo_children():
            w.destroy()

    def show_main_menu(self):
        self.clear()
        tk.Label(self.root, text="MiniBank", font=("Arial", 18)).pack(pady=15)
        tk.Button(self.root, text="Create Account", width=20, command=self.create_account).pack(pady=5)
        tk.Button(self.root, text="Login", width=20, command=self.login).pack(pady=5)
        tk.Button(self.root, text="Admin", width=20, command=self.admin_login).pack(pady=5)
        tk.Button(self.root, text="Exit", width=20, command=self.root.destroy).pack(pady=10)

    def create_account(self):
        self.clear()
        tk.Label(self.root, text="Create Account", font=("Arial", 14)).pack(pady=10)

        name = tk.Entry(self.root, width=30)
        pin = tk.Entry(self.root, width=30, show="*")
        pin2 = tk.Entry(self.root, width=30, show="*")
        initial = tk.Entry(self.root, width=30)

        tk.Label(self.root, text="Full Name").pack()
        name.pack()
        tk.Label(self.root, text="PIN (4-6 digits)").pack()
        pin.pack()
        tk.Label(self.root, text="Confirm PIN").pack()
        pin2.pack()
        tk.Label(self.root, text="Initial Deposit").pack()
        initial.pack()

        def submit():
            n = name.get().strip()
            p1 = pin.get()
            p2 = pin2.get()
            dep = initial.get().strip() or "0"

            if not n or not p1.isdigit() or not (4 <= len(p1) <= 6) or p1 != p2:
                messagebox.showerror("Error", "Invalid input")
                return

            try:
                dep = float(dep)
            except:
                dep = 0

            acc = bank.create_account(n, p1, dep)
            messagebox.showinfo("Success", f"Account created!\nYour number: {acc.acc_no}")
            self.show_main_menu()

        tk.Button(self.root, text="Create", command=submit).pack(pady=10)
        tk.Button(self.root, text="Back", command=self.show_main_menu).pack()

    def login(self):
        self.clear()
        tk.Label(self.root, text="Login", font=("Arial", 14)).pack(pady=10)

        acc_no = tk.Entry(self.root, width=30)
        pin = tk.Entry(self.root, width=30, show="*")

        tk.Label(self.root, text="Account Number").pack()
        acc_no.pack()
        tk.Label(self.root, text="PIN").pack()
        pin.pack()

        def submit():
            global current_account
            a = bank.get_account(acc_no.get().strip())
            if not a or not a.verify_pin(pin.get()):
                messagebox.showerror("Error", "Invalid login")
                return
            current_account = a
            self.account_menu()

        tk.Button(self.root, text="Login", command=submit).pack(pady=10)
        tk.Button(self.root, text="Back", command=self.show_main_menu).pack()

    def account_menu(self):
        self.clear()
        acc = current_account

        tk.Label(self.root, text=f"Welcome {acc.name}", font=("Arial", 14)).pack(pady=10)
        tk.Label(self.root, text=f"Account: {acc.acc_no}").pack()
        bal = tk.StringVar()
        bal.set(f"Balance: {acc.balance:.2f}")
        tk.Label(self.root, textvariable=bal, font=("Arial", 12)).pack(pady=10)

        def refresh():
            bal.set(f"Balance: {acc.balance:.2f}")

        def deposit():
            amt = simpledialog.askfloat("Deposit", "Amount:", minvalue=0.0)
            if amt:
                try:
                    bank.deposit(acc, amt)
                    refresh()
                except Exception as e:
                    messagebox.showerror("Error", str(e))

        def withdraw():
            amt = simpledialog.askfloat("Withdraw", "Amount:", minvalue=0.0)
            if amt:
                try:
                    bank.withdraw(acc, amt)
                    refresh()
                except Exception as e:
                    messagebox.showerror("Error", str(e))

        def transfer():
            to = simpledialog.askstring("Transfer", "Destination account:")
            if not to:
                return
            amt = simpledialog.askfloat("Amount", "Amount:", minvalue=0.0)
            if amt:
                try:
                    bank.transfer(acc, to.strip(), amt)
                    refresh()
                except Exception as e:
                    messagebox.showerror("Error", str(e))

        def history():
            win = tk.Toplevel(self.root)
            win.title("Transactions")
            text = tk.Text(win, width=80, height=20)
            text.pack()
            for t in acc.transactions:
                line = f"{t['timestamp']} | {t['type']} | {t['amount']} | Bal:{t['balance_after']} | {t['description']}\n"
                text.insert("end", line)
            text.config(state="disabled")

        def close():
            if messagebox.askyesno("Confirm", "Close account permanently?"):
                bank.close_account(acc.acc_no)
                messagebox.showinfo("Closed", "Account closed")
                self.show_main_menu()

        tk.Button(self.root, text="Deposit", width=20, command=deposit).pack(pady=3)
        tk.Button(self.root, text="Withdraw", width=20, command=withdraw).pack(pady=3)
        tk.Button(self.root, text="Transfer", width=20, command=transfer).pack(pady=3)
        tk.Button(self.root, text="Transactions", width=20, command=history).pack(pady=3)
        tk.Button(self.root, text="Close Account", width=20, command=close).pack(pady=3)
        tk.Button(self.root, text="Logout", width=20, command=self.show_main_menu).pack(pady=10)

    def admin_login(self):
        pwd = simpledialog.askstring("Admin Login", "Password:", show="*")
        if pwd == ADMIN_PASSWORD:
            self.admin_menu()
        else:
            messagebox.showerror("Error", "Wrong password")

    def admin_menu(self):  
        self.clear()
        tk.Label(self.root, text="ADMIN PANEL", font=("Arial", 16)).pack(pady=10)

        def list_accounts():
            win = tk.Toplevel(self.root)
            text = tk.Text(win, width=80, height=20)
            text.pack()
            for acc_no, acc in bank.accounts.items():
                text.insert("end", f"{acc_no} | {acc.name} | Bal:{acc.balance} | TX:{len(acc.transactions)}\n")
            text.config(state="disabled")

        def details():
            no = simpledialog.askstring("Account", "Account number:")
            acc = bank.get_account(no)
            if not acc:
                messagebox.showerror("Error", "Not found")
                return
            win = tk.Toplevel(self.root)
            text = tk.Text(win, width=80, height=20)
            text.pack()
            text.insert("end", json.dumps(asdict(acc), indent=2))
            text.config(state="disabled")

        def backup():
            fname = f"backup_{int(time.time())}.json"
            with open(fname, "w") as f:
                with open(bank.db_file, "r") as src:
                    f.write(src.read())
            messagebox.showinfo("Backup", f"Saved to {fname}")

        def restore():
            fname = simpledialog.askstring("Restore", "File name:")
            if not os.path.exists(fname):
                messagebox.showerror("Error", "File not found")
                return
            with open(fname, "r") as src:
                data = src.read()
            with open(bank.db_file, "w") as dst:
                dst.write(data)
            bank._load()
            messagebox.showinfo("Restore", "Database restored")

        tk.Button(self.root, text="List Accounts", width=25, command=list_accounts).pack(pady=5)
        tk.Button(self.root, text="Account Details", width=25, command=details).pack(pady=5)
        tk.Button(self.root, text="Backup DB", width=25, command=backup).pack(pady=5)
        tk.Button(self.root, text="Restore DB", width=25, command=restore).pack(pady=5)
        tk.Button(self.root, text="Back", width=25, command=self.show_main_menu).pack(pady=10)


root = tk.Tk()
app = App(root)
root.mainloop()
