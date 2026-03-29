# 📧 EmailMessage vs MIMEMultipart in Python

This document explains **email construction in Python**, starting with the **legacy approach (`MIMEMultipart`)**, then the **modern approach (`EmailMessage`)**, and finally a **clear comparison** to help you choose the right one.

--- 

## 1️⃣ MIMEMultipart (Legacy / Traditional Approach)

### 🔹 What is MIMEMultipart?

`MIMEMultipart` is a class from Python’s `email.mime.multipart` module used to create **emails with multiple parts**, such as:

* Plain text
* HTML content
* Attachments (PDF, images, etc.)
* Inline images

It acts as a **container** that can hold multiple MIME objects.

---

### 📦 Import

```python
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.application import MIMEApplication
```

---

### 🧠 How it works (Conceptually)

* You create a `MIMEMultipart` object
* You manually attach parts (`MIMEText`, `MIMEImage`, `MIMEApplication`)
* You manage MIME headers yourself

Think of it as **manual email assembly**.

---

### ✅ Example: Text email

```python
msg = MIMEMultipart()
msg["From"] = "sender@example.com"
msg["To"] = "user@example.com"
msg["Subject"] = "Hello"

msg.attach(MIMEText("This is a plain text email", "plain"))
```

---

### ✅ Example: Text + HTML email

```python
msg = MIMEMultipart("alternative")

msg.attach(MIMEText("Plain text fallback", "plain"))
msg.attach(MIMEText("<h1>Hello</h1>", "html"))
```

---

### ✅ Example: Attachment

```python
with open("report.pdf", "rb") as f:
    attachment = MIMEApplication(f.read(), _subtype="pdf")

attachment.add_header(
    "Content-Disposition",
    "attachment",
    filename="report.pdf"
)

msg.attach(attachment)
```

---

### ⚠️ Downsides of MIMEMultipart

* Verbose and boilerplate-heavy
* Manual MIME management
* Easy to make mistakes
* Harder to read and maintain
* Considered **legacy style**

---

## 2️⃣ EmailMessage (Modern & Recommended)

### 🔹 What is EmailMessage?

`EmailMessage` is part of Python’s **modern email API** (`email.message`).

It simplifies email creation by:

* Handling MIME structure internally
* Providing high-level helper methods
* Reducing boilerplate

> **EmailMessage replaces most uses of MIMEMultipart**

---

### 📦 Import

```python
from email.message import EmailMessage
```

---

### 🧠 How it works

* You define content using `set_content()`
* You add HTML with `add_alternative()`
* You attach files with `add_attachment()`

Python handles MIME types automatically.

---

### ✅ Example: Text email

```python
msg = EmailMessage()
msg["From"] = "sender@example.com"
msg["To"] = "user@example.com"
msg["Subject"] = "Hello"

msg.set_content("This is a plain text email")
```

---

### ✅ Example: Text + HTML email

```python
msg.set_content("Plain text fallback")

msg.add_alternative(
    """
    <html>
        <body>
            <h1>Hello</h1>
            <p>This is an HTML email</p>
        </body>
    </html>
    """,
    subtype="html"
)
```

---

### ✅ Example: Attachment

```python
with open("report.pdf", "rb") as f:
    msg.add_attachment(
        f.read(),
        maintype="application",
        subtype="pdf",
        filename="report.pdf"
    )
```

---

### 🔥 Advantages of EmailMessage

* Minimal boilerplate
* Clean and readable
* Automatic MIME handling
* Built-in support for attachments
* Actively recommended by Python docs

---

## 3️⃣ EmailMessage vs MIMEMultipart (Comparison)

### 📊 Feature comparison

| Feature                  | EmailMessage | MIMEMultipart |
| ------------------------ | ------------ | ------------- |
| API style                | Modern       | Legacy        |
| Boilerplate              | Low          | High          |
| MIME handling            | Automatic    | Manual        |
| Attachments              | Easy         | Verbose       |
| HTML support             | Built-in     | Manual        |
| Readability              | Excellent    | Poor          |
| Recommended for new code | ✅ Yes        | ❌ No          |

---

### 🧠 When should you use which?

#### ✅ Use EmailMessage when:

* Writing new code
* Building APIs / services
* Sending HTML emails
* Attaching files
* Using FastAPI / modern Python

#### ⚠️ Use MIMEMultipart only when:

* Maintaining legacy systems
* Working with very old Python code
* Required by an old library

---

## 4️⃣ Best Practices (Production)

* ✔ Prefer **EmailMessage**
* ✔ Use SMTP with app passwords
* ✔ Keep email logic in a service layer
* ✔ Separate HTML templates (Jinja2)
* ✔ Avoid hardcoding credentials

---

## 🧠 Final Takeaway

> **MIMEMultipart is legacy and verbose. EmailMessage is modern, clean, and the recommended way forward.**

If you are starting today — **use EmailMessage**.

---

## 4️⃣ Real-world Example: OTP Email (Password Reset)

Below is the same **OTP email functionality** implemented using **both approaches**, so you can clearly see the difference.

---

### 🔹 A) Using MIMEMultipart (Legacy Way)

```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText


def send_otp_email(to_email: str, otp: str) -> bool:
    try:
        msg = MIMEMultipart()
        msg['From'] = f"AIChatApp Support <{SENDER_EMAIL}>"
        msg['To'] = to_email
        msg['Subject'] = "AIChatApp Password Reset OTP"

        body = f"Your OTP for password reset is: {otp}. It is valid for 5 minutes."
        msg.attach(MIMEText(body, 'plain'))

        server = smtplib.SMTP(SMTP_SERVER, SMTP_PORT)
        server.starttls()
        server.login(SENDER_EMAIL, SENDER_PASSWORD)
        server.sendmail(SENDER_EMAIL, to_email, msg.as_string())
        server.quit()

        return True
    except Exception as e:
        print(f"Error sending email: {e}")
        return False
```

⚠️ This works, but it is **verbose**, **manual**, and considered **legacy style**.

---

### 🔹 B) Using EmailMessage (Modern & Recommended)

```python
import smtplib
from email.message import EmailMessage


def send_otp_email(to_email: str, otp: str) -> bool:
    try:
        msg = EmailMessage()
        msg['From'] = f"AIChatApp Support <{SENDER_EMAIL}>"
        msg['To'] = to_email
        msg['Subject'] = "AIChatApp Password Reset OTP"

        msg.set_content(
            f"Your OTP for password reset is: {otp}. It is valid for 5 minutes."
        )

        with smtplib.SMTP(SMTP_SERVER, SMTP_PORT) as server:
            server.starttls()
            server.login(SENDER_EMAIL, SENDER_PASSWORD)
            server.send_message(msg)

        return True
    except Exception as e:
        print(f"Error sending email: {e}")
        return False
```

✔ Cleaner
✔ Less error-prone
✔ Modern API
✔ Recommended for new code

---

### 🧠 Key Differences in OTP Example

| Aspect        | MIMEMultipart | EmailMessage |
| ------------- | ------------- | ------------ |
| Lines of code | More          | Fewer        |
| MIME handling | Manual        | Automatic    |
| Readability   | Lower         | Higher       |
| Maintenance   | Harder        | Easier       |
| Recommended   | ❌             | ✅            |

---

## 🔜 Next steps (optional)

* Create an `EmailService` abstraction
* Add HTML email templates (Jinja2)
* Use FastAPI `BackgroundTasks`
* Add retry & structured logging

---

📌 *This document is suitable for internal docs, onboarding, and interviews.*
