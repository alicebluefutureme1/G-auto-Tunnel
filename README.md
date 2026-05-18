# G2ray — Auto Codespace

پروژه‌ای برای اجرای خودکار **Xray (VLESS)** روی GitHub Codespace بدون نیاز به باز کردن VS Code.

---

## ساختار پروژه

```
.
├── .devcontainer/
│   ├── Dockerfile          # محیط Codespace
│   └── devcontainer.json   # تنظیمات Codespace
├── .github/
│   └── workflows/
│       └── auto-start.yml  # اجرای خودکار هر ۵ ساعت
├── g2ray.sh                # اسکریپت اصلی
└── README.md
```

---

## راه‌اندازی

### ۱. ساخت GitHub Personal Access Token

برو به [github.com/settings/tokens](https://github.com/settings/tokens) و یک token با این scope ها بساز:
- `codespace` (همه)
- `repo`

### ۲. اضافه کردن Secret

در ریپو برو به:  
**Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
|------|-------|
| `CODESPACE_TOKEN` | توکن ساخته‌شده |

### ۳. ساخت Google Apps Script

در [script.google.com](https://script.google.com) یک پروژه جدید بساز و کد زیر رو paste کن:

```javascript
const GITHUB_TOKEN = "ghp_xxxx";        // توکن GitHub
const GITHUB_REPO  = "username/repo";   // نام ریپو
const GITHUB_FILE  = "configs.txt";     // فایل ذخیره لینک‌ها
const GITHUB_BRANCH = "main";

function doPost(e) {
  try {
    const body = JSON.parse(e.postData.contents);
    const message = body.message;
    if (!message || !message.startsWith("vless://"))
      return ContentService.createTextOutput("Invalid").setMimeType(ContentService.MimeType.TEXT);

    const apiUrl = `https://api.github.com/repos/${GITHUB_REPO}/contents/${GITHUB_FILE}`;
    const getRes = UrlFetchApp.fetch(apiUrl, {
      method: "GET",
      headers: { Authorization: `token ${GITHUB_TOKEN}`, Accept: "application/vnd.github.v3+json" },
      muteHttpExceptions: true,
    });

    let currentContent = "", sha = "";
    if (getRes.getResponseCode() === 200) {
      const fd = JSON.parse(getRes.getContentText());
      sha = fd.sha;
      currentContent = Utilities.newBlob(Utilities.base64Decode(fd.content.replace(/\n/g,""))).getDataAsString();
    }

    const newContent = currentContent + message + "\n";
    const payload = { message: "Add vless config", content: Utilities.base64Encode(newContent, Utilities.Charset.UTF_8), branch: GITHUB_BRANCH };
    if (sha) payload.sha = sha;

    const putRes = UrlFetchApp.fetch(apiUrl, {
      method: "PUT",
      headers: { Authorization: `token ${GITHUB_TOKEN}`, Accept: "application/vnd.github.v3+json", "Content-Type": "application/json" },
      payload: JSON.stringify(payload),
      muteHttpExceptions: true,
    });

    if ([200,201].includes(putRes.getResponseCode()))
      return ContentService.createTextOutput("Appended to GitHub").setMimeType(ContentService.MimeType.TEXT);
    else
      return ContentService.createTextOutput("GitHub error: " + putRes.getContentText()).setMimeType(ContentService.MimeType.TEXT);
  } catch(err) {
    return ContentService.createTextOutput("Error: " + err.message).setMimeType(ContentService.MimeType.TEXT);
  }
}
```

بعد **Deploy → New deployment → Web app** کن و URL رو در `g2ray.sh` داخل `GAS_URL` جایگزین کن.

---

## جریان کار خودکار

```
هر ۵ ساعت
    ↓
GitHub Actions اجرا میشه
    ↓
Codespace start / create میشه
    ↓
g2ray.sh --silent-start اجرا میشه
    ↓
Config نداشت → خودکار Generate
Config داشت  → Xray Start
    ↓
لینک VLESS ساخته میشه
    ↓
به Google Script فرستاده میشه
    ↓
در GitHub ذخیره میشه ✅
```
