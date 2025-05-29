import tkinter as tk
from tkinter import ttk, messagebox, filedialog
import datetime
import os
import json
import tempfile
import webbrowser

# File to store user credentials
USER_FILE = "users.json"

# Load users
if os.path.exists(USER_FILE):
    with open(USER_FILE, 'r') as f:
        users = json.load(f)
else:
    users = {"admin": {"password": "admin123", "role": "admin"}}

# Save users
with open(USER_FILE, 'w') as f:
    json.dump(users, f)

# Global Variables
customer_details = {}
current_bill = 0
water_arrears = 0
total_due = 0
total_after_due = 0
payment_due_date = ""
disconnection_due_date = ""
transaction_history = []
total_revenue = 0
average_consumption = 0
transaction_count = 0
current_payment = 0
current_balance = 0
current_status = ""

# App Window
root = tk.Tk()
root.title("Water Billing System")
root.geometry("1000x700")
root.configure(bg="white")
root.resizable(False, False)

# Frames
login_frame = tk.Frame(root, bg="white")
register_frame = tk.Frame(root, bg="white")
billing_frame = tk.Frame(root, bg="white")
for frame in (login_frame, register_frame, billing_frame):
    frame.place(x=0, y=0, width=1000, height=700)


# Switch Frames
def raise_frame(frame):
    frame.tkraise()


# Login Logic
def login():
    username = username_entry.get()
    password = password_entry.get()
    if username in users and users[username]["password"] == password:
        global current_user
        current_user = username
        role = users[username]["role"]
        welcome_label.config(text=f"Welcome, {username.title()}! Role: {role.title()}")
        raise_frame(billing_frame)
    else:
        messagebox.showerror("Login Failed", "Invalid credentials")


# Register Logic
def register():
    username = reg_username_entry.get()
    password = reg_password_entry.get()
    if username in users:
        messagebox.showerror("Error", "Username already exists")
    else:
        users[username] = {"password": password, "role": "user"}
        with open(USER_FILE, 'w') as f:
            json.dump(users, f)
        messagebox.showinfo("Success", "Registration successful")
        raise_frame(login_frame)


# Delete User
def delete_user():
    if users[current_user]['role'] != 'admin':
        messagebox.showerror("Permission Denied", "Only admin can delete users")
        return
    username = delete_user_entry.get()
    if username == "admin":
        messagebox.showerror("Error", "Cannot delete admin account")
    elif username in users:
        del users[username]
        with open(USER_FILE, 'w') as f:
            json.dump(users, f)
        messagebox.showinfo("Deleted", f"User '{username}' deleted successfully")
    else:
        messagebox.showerror("Error", "User not found")


# Login GUI
login_title = tk.Label(login_frame, text="Login", font=("Arial", 24), bg="white")
login_title.pack(pady=20)
username_label = tk.Label(login_frame, text="Username:", font=("Arial", 14), bg="white")
username_label.pack()
username_entry = tk.Entry(login_frame, font=("Arial", 14))
username_entry.pack(pady=5)
password_label = tk.Label(login_frame, text="Password:", font=("Arial", 14), bg="white")
password_label.pack()
password_entry = tk.Entry(login_frame, font=("Arial", 14), show='*')
password_entry.pack(pady=5)
login_button = tk.Button(login_frame, text="Login", font=("Arial", 14), bg="#007bff", fg="white", command=login)
login_button.pack(pady=10)
register_link = tk.Button(login_frame, text="Register", font=("Arial", 12), fg="#007bff", bg="white",
                          command=lambda: raise_frame(register_frame))
register_link.pack()

# Registration GUI
tk.Label(register_frame, text="Register", font=("Arial", 24), bg="white").pack(pady=20)
tk.Label(register_frame, text="Username:", font=("Arial", 14), bg="white").pack()
reg_username_entry = tk.Entry(register_frame, font=("Arial", 14))
reg_username_entry.pack(pady=5)
tk.Label(register_frame, text="Password:", font=("Arial", 14), bg="white").pack()
reg_password_entry = tk.Entry(register_frame, font=("Arial", 14), show='*')
reg_password_entry.pack(pady=5)
tk.Button(register_frame, text="Register", font=("Arial", 14), bg="#28a745", fg="white", command=register).pack(pady=10)
tk.Button(register_frame, text="Back to Login", font=("Arial", 12), fg="#007bff", bg="white",
          command=lambda: raise_frame(login_frame)).pack()

# Billing Frame
delete_user_entry = tk.Entry(billing_frame)
delete_user_entry.pack(pady=5)
delete_user_button = tk.Button(billing_frame, text="Delete User (Admin Only)", command=delete_user, bg="red",
                               fg="white")
delete_user_button.pack(pady=5)
welcome_label = tk.Label(billing_frame, text="", font=("Arial", 14), bg="white")
welcome_label.pack(pady=10)

# Tabs
tabs = ttk.Notebook(billing_frame)
tab1 = ttk.Frame(tabs);
tab2 = ttk.Frame(tabs);
tab3 = ttk.Frame(tabs)
tab4 = ttk.Frame(tabs);
tab5 = ttk.Frame(tabs);
tab6 = ttk.Frame(tabs)
tabs.add(tab1, text="Billing");
tabs.add(tab2, text="Payment")
tabs.add(tab3, text="History");
tabs.add(tab4, text="Analytics")
tabs.add(tab5, text="Summary");
tabs.add(tab6, text="Receipt")
tabs.pack(expand=1, fill='both')

# Billing Entry Fields
entries = []
labels = ["Customer Code:", "Name:", "Address:", "Block and Lot No:", "Consumption (cu.m):"]
for i, lbl in enumerate(labels):
    l = tk.Label(tab1, text=lbl, font=("Arial", 12))
    l.grid(row=i, column=0, sticky="w", padx=10, pady=5)
    e = tk.Entry(tab1)
    e.grid(row=i, column=1, padx=10)
    entries.append(e)

# Search Customer
search_entry = tk.Entry(tab1)
search_entry.grid(row=6, column=0, padx=10, pady=5)
tk.Button(tab1, text="Search by Code", bg="#ffc107", command=lambda: search_customer(search_entry.get())).grid(row=6,
                                                                                                               column=1)


def search_customer(code):
    for t in transaction_history:
        if t['code'] == code:
            messagebox.showinfo("Found", f"Name: {t['name']}\nTotal: ₱{t['total']:.2f}")
            return
    messagebox.showerror("Not Found", "Customer code not found")


def update_summary():
    global water_arrears, total_due, total_after_due, payment_due_date, disconnection_due_date
    water_arrears = current_bill * 0.10
    total_due = current_bill + water_arrears
    total_after_due = total_due * 1.05
    payment_due_date = (datetime.datetime.now() + datetime.timedelta(days=30)).strftime("%Y-%m-%d")
    disconnection_due_date = (datetime.datetime.now() + datetime.timedelta(days=60)).strftime("%Y-%m-%d")
    billing_summary_text.config(state='normal')
    billing_summary_text.delete(1.0, tk.END)
    billing_summary_text.insert(tk.END, f"Current Bill: ₱{current_bill:.2f}\n")
    billing_summary_text.insert(tk.END, f"Arrears: ₱{water_arrears:.2f}\n")
    billing_summary_text.insert(tk.END, f"Total Due: ₱{total_due:.2f}\n")
    billing_summary_text.insert(tk.END, f"After Due: ₱{total_after_due:.2f}\n")
    billing_summary_text.insert(tk.END, f"Due Date: {payment_due_date}\n")
    billing_summary_text.insert(tk.END, f"Disconnection: {disconnection_due_date}\n")
    billing_summary_text.config(state='disabled')


# Calculation Logic
def calculate_bill():
    try:
        code, name, project, block, cons = [e.get() for e in entries]
        cons = float(cons)
        global current_bill
        if cons <= 10:
            bill = cons * 30
        elif cons <= 20:
            bill = 10 * 30 + (cons - 10) * 35
        elif cons <= 30:
            bill = 10 * 30 + 10 * 35 + (cons - 20) * 40
        else:
            bill = 10 * 30 + 10 * 35 + 10 * 40 + (cons - 30) * 50
        env_fee = bill * 0.10
        vat = bill * 0.12
        current_bill = bill + env_fee + vat + 50
        customer_details.update(
            {"code": code, "name": name, "project": project, "block": block, "cons": cons, "total": current_bill})
        update_summary()
        messagebox.showinfo("Bill Calculated", f"Total: ₱{current_bill:.2f}")
    except:
        messagebox.showerror("Invalid", "Please check your inputs.")


tk.Button(tab1, text="Calculate", bg="#007bff", fg="white", command=calculate_bill).grid(row=5, column=0, columnspan=2,
                                                                                         pady=10)

# Payment Tab
payment_entry = tk.Entry(tab2)
tk.Label(tab2, text="Enter Payment").pack()
payment_entry.pack(pady=10)


def generate_receipt(paid, bal, status):
    global current_payment, current_balance, current_status
    current_payment = paid
    current_balance = bal
    current_status = status

    receipt_text.config(state='normal')
    receipt_text.delete(1.0, tk.END)

    # Enhanced receipt format with proper header and sections
    receipt_text.insert(tk.END, "=" * 60 + "\n")
    receipt_text.insert(tk.END, "AQUA WATER UTILITIES COMPANY, INC.\n")
    receipt_text.insert(tk.END, "Corrales Ave, Cagayan de Oro, 9000 Misamis Oriental\n")
    receipt_text.insert(tk.END, "Tel: +6392-992-2137 | Email: billing@aquawater.com\n")
    receipt_text.insert(tk.END, "=" * 60 + "\n\n")

    receipt_text.insert(tk.END, "OFFICIAL PAYMENT RECEIPT\n")
    receipt_text.insert(tk.END, "-" * 30 + "\n\n")

    receipt_text.insert(tk.END, "CUSTOMER INFORMATION\n")
    receipt_text.insert(tk.END, "-" * 30 + "\n")
    receipt_text.insert(tk.END, f"Receipt No: {datetime.datetime.now().strftime('%Y%m%d%H%M%S')}\n")
    receipt_text.insert(tk.END, f"Date: {datetime.datetime.now().strftime('%B %d, %Y - %I:%M:%S %p')}\n")
    receipt_text.insert(tk.END, f"Customer Code: {customer_details['code']}\n")
    receipt_text.insert(tk.END, f"Name: {customer_details['name']}\n")
    receipt_text.insert(tk.END,
                        f"Service Address: {customer_details['project']}, Block & Lot: {customer_details['block']}\n\n")

    receipt_text.insert(tk.END, "BILLING DETAILS\n")
    receipt_text.insert(tk.END, "-" * 30 + "\n")
    receipt_text.insert(tk.END, f"Consumption: {customer_details['cons']} cu.m\n")
    receipt_text.insert(tk.END,
                        f"Water Charges: ₱{current_bill - (current_bill * 0.12) - (current_bill * 0.10) - 50:.2f}\n")
    receipt_text.insert(tk.END, f"Environmental Fee (10%): ₱{current_bill * 0.10:.2f}\n")
    receipt_text.insert(tk.END, f"VAT (12%): ₱{current_bill * 0.12:.2f}\n")
    receipt_text.insert(tk.END, f"Other Charges: ₱50.00\n")
    receipt_text.insert(tk.END, f"Current Bill: ₱{current_bill:.2f}\n")
    receipt_text.insert(tk.END, f"Arrears: ₱{water_arrears:.2f}\n")
    receipt_text.insert(tk.END, f"Total Amount Due: ₱{total_due:.2f}\n\n")

    receipt_text.insert(tk.END, "PAYMENT INFORMATION\n")
    receipt_text.insert(tk.END, "-" * 30 + "\n")
    receipt_text.insert(tk.END, f"Amount Paid: ₱{paid:.2f}\n")
    receipt_text.insert(tk.END, f"Balance: ₱{bal:.2f}\n")
    receipt_text.insert(tk.END, f"Payment Status: {status}\n\n")

    receipt_text.insert(tk.END, "IMPORTANT DATES\n")
    receipt_text.insert(tk.END, "-" * 30 + "\n")
    receipt_text.insert(tk.END, f"Due Date: {payment_due_date}\n")
    receipt_text.insert(tk.END, f"Disconnection Date: {disconnection_due_date}\n\n")

    receipt_text.insert(tk.END, "=" * 60 + "\n")
    receipt_text.insert(tk.END, "Thank you for your payment!\n")
    receipt_text.insert(tk.END, "For billing inquiries, please contact our Customer Service.\n")
    receipt_text.insert(tk.END, "This is a computer-generated receipt. No signature required.\n")
    receipt_text.insert(tk.END, "=" * 60)

    receipt_text.config(state='disabled')


def pay():
    try:
        paid = float(payment_entry.get())
        bal = customer_details['total'] - paid
        status = "Paid" if bal <= 0 else "Partial Payment"

        if bal > 0:
            messagebox.showinfo("Lacking Payment",
                                f"Payment is lacking by ₱{bal:.2f}\nPlease add more payment to complete the transaction.")
        else:
            messagebox.showinfo("Status", f"{status}. Change: ₱{abs(bal) if bal < 0 else 0:.2f}")
        global total_revenue, transaction_count
        total_revenue += customer_details['total']
        transaction_count += 1
        average = total_revenue / transaction_count
        analytics_text.config(state='normal')
        analytics_text.delete(1.0, tk.END)
        analytics_text.insert(tk.END,
                              f"Revenue: ₱{total_revenue:.2f}\nAverage: ₱{average:.2f}\nCount: {transaction_count}")
        analytics_text.config(state='disabled')
        transaction_history.append(customer_details.copy())
        generate_receipt(paid, max(bal, 0), status)
    except:
        messagebox.showerror("Error", "Enter valid payment")


tk.Button(tab2, text="Submit Payment", bg="#007bff", fg="white", command=pay).pack()

# History Tab
history_text = tk.Text(tab3, height=20, width=70)
history_text.pack()


def show_history():
    history_text.delete(1.0, tk.END)
    for t in transaction_history:
        history_text.insert(tk.END, f"Code: {t['code']} | Name: {t['name']} | Total: ₱{t['total']:.2f}\n")


def export_history():
    filename = filedialog.asksaveasfilename(defaultextension=".txt", filetypes=[("Text files", "*.txt")])
    if filename:
        with open(filename, 'w') as f:
            for t in transaction_history:
                f.write(f"Code: {t['code']} | Name: {t['name']} | Total: ₱{t['total']:.2f}\n")
        messagebox.showinfo("Exported", "History exported successfully")


show_history_btn = tk.Button(tab3, text="Show History", bg="#007bff", fg="white", command=show_history)
show_history_btn.pack()
tk.Button(tab3, text="Export History", bg="#28a745", fg="white", command=export_history).pack(pady=5)

# Analytics and Summary
analytics_text = tk.Text(tab4, state='disabled', height=10)
analytics_text.pack(pady=10)
billing_summary_text = tk.Text(tab5, state='disabled', height=10)
billing_summary_text.pack(pady=10)

# Receipt Tab
receipt_text = tk.Text(tab6, state='disabled', height=20, width=70)
receipt_text.pack(pady=10)


def print_receipt():
    receipt = receipt_text.get(1.0, tk.END)
    if not receipt.strip():
        messagebox.showerror("Error", "No receipt to print.")
        return

    # Create a HTML version of the receipt for better printing
    html_receipt = f"""<!DOCTYPE html>
<html>
<head>
    <title>Water Billing Receipt</title>
    <style>
        body {{
            font-family: Arial, sans-serif;
            margin: 20px;
            line-height: 1.6;
        }}
        .receipt {{
            width: 80%;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            border: 1px solid #ccc;
        }}
        .header {{
            text-align: center;
            border-bottom: 2px solid #000;
            padding-bottom: 10px;
            margin-bottom: 20px;
        }}
        .title {{
            font-size: 24px;
            font-weight: bold;
            text-align: center;
            margin: 15px 0;
        }}
        .section {{
            margin-bottom: 20px;
        }}
        .section-title {{
            font-weight: bold;
            border-bottom: 1px solid #ccc;
            margin-bottom: 10px;
        }}
        .row {{
            display: flex;
            justify-content: space-between;
            margin: 5px 0;
        }}
        .footer {{
            text-align: center;
            margin-top: 30px;
            font-size: 14px;
            border-top: 2px solid #000;
            padding-top: 10px;
        }}
    </style>
</head>
<body>
    <div class="receipt">
        <div class="header">
            <h1>AQUA WATER UTILITIES COMPANY, INC.</h1>
            <p>Corrales Ave, Cagayan de Oro, 9000 Misamis Oriental</p>
            <p>Tel: +6392-992-2137 | Email: billing@aquawater.com</p>
        </div>

        <div class="title">OFFICIAL PAYMENT RECEIPT</div>

        <div class="section">
            <div class="section-title">CUSTOMER INFORMATION</div>
            <p>Receipt No: {datetime.datetime.now().strftime('%Y%m%d%H%M%S')}</p>
            <p>Date: {datetime.datetime.now().strftime('%B %d, %Y - %I:%M:%S %p')}</p>
            <p>Customer Code: {customer_details.get('code', 'N/A')}</p>
            <p>Name: {customer_details.get('name', 'N/A')}</p>
            <p>Service Address: {customer_details.get('project', 'N/A')}, Block & Lot: {customer_details.get('block', 'N/A')}</p>
        </div>

        <div class="section">
            <div class="section-title">BILLING DETAILS</div>
            <div class="row">
                <span>Consumption:</span>
                <span>{customer_details.get('cons', 0)} cu.m</span>
            </div>
            <div class="row">
                <span>Water Charges:</span>
                <span>₱{current_bill - (current_bill * 0.12) - (current_bill * 0.10) - 50:.2f}</span>
            </div>
            <div class="row">
                <span>Environmental Fee (10%):</span>
                <span>₱{current_bill * 0.10:.2f}</span>
            </div>
            <div class="row">
                <span>VAT (12%):</span>
                <span>₱{current_bill * 0.12:.2f}</span>
            </div>
            <div class="row">
                <span>Other Charges:</span>
                <span>₱50.00</span>
            </div>
            <div class="row">
                <span>Current Bill:</span>
                <span>₱{current_bill:.2f}</span>
            </div>
            <div class="row">
                <span>Arrears:</span>
                <span>₱{water_arrears:.2f}</span>
            </div>
            <div class="row">
                <span>Total Amount Due:</span>
                <span>₱{total_due:.2f}</span>
            </div>
        </div>

        <div class="section">
            <div class="section-title">PAYMENT INFORMATION</div>
            <div class="row">
                <span>Amount Paid:</span>
                <span>₱{current_payment:.2f}</span>
            </div>
            <div class="row">
                <span>Balance:</span>
                <span>₱{current_balance:.2f}</span>
            </div>
            <div class="row">
                <span>Payment Status:</span>
                <span>{current_status}</span>
            </div>
        </div>

        <div class="section">
            <div class="section-title">IMPORTANT DATES</div>
            <div class="row">
                <span>Due Date:</span>
                <span>{payment_due_date}</span>
            </div>
            <div class="row">
                <span>Disconnection Date:</span>
                <span>{disconnection_due_date}</span>
            </div>
        </div>

        <div class="footer">
            <p>Thank you for your payment!</p>
            <p>For billing inquiries, please contact our Customer Service.</p>
            <p>This is a computer-generated receipt. No signature required.</p>
        </div>
    </div>
</body>
</html>"""

    # Save HTML to temporary file
    with tempfile.NamedTemporaryFile(delete=False, suffix=".html", mode='w', encoding='utf-8') as temp:
        temp.write(html_receipt)
        temp_path = temp.name

    # Open HTML file in default browser for printing
    try:
        webbrowser.open('file://' + temp_path)
        messagebox.showinfo("Print", "Receipt opened in browser for printing")
    except Exception as e:
        # Fallback to plain text if browser opening fails
        with tempfile.NamedTemporaryFile(delete=False, suffix=".txt", mode='w', encoding='utf-8') as temp:
            temp.write(receipt)
            temp_path = temp.name
        try:
            os.startfile(temp_path, "print")  # Windows
        except Exception as e:
            messagebox.showerror("Print Error", f"Could not print receipt:\n{str(e)}")


print_button = tk.Button(tab6, text="Print Receipt", bg="#007bff", fg="white", command=print_receipt)
print_button.pack(pady=5)


# Logout functionality
def logout():
    raise_frame(login_frame)
    username_entry.delete(0, tk.END)
    password_entry.delete(0, tk.END)


logout_button = tk.Button(billing_frame, text="Logout", bg="#dc3545", fg="white", command=logout)
logout_button.pack(side=tk.BOTTOM, pady=10)

# Start App
raise_frame(login_frame)
root.mainloop()
