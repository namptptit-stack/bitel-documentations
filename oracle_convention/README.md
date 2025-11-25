# 🗄️ Oracle Database Code Convention

## 📦 Files trong package

```
📁 outputs/
├── 📄 oracle-convention-en.html    (English version - 64KB)
├── 📄 oracle-convention-vi.html    (Vietnamese version - 65KB)
└── 📄 oracle-code-convention.md    (Markdown source)
```

---

## ✨ Tính năng

### 🎨 **UI/UX chuyên nghiệp**
- ✅ Responsive design (Mobile, Tablet, Desktop)
- ✅ Smooth scrolling navigation
- ✅ Back to top button
- ✅ Language switcher (EN ↔ VI)
- ✅ **Dark Mode toggle** 🌙 (NEW!)
- ✅ Code syntax highlighting
- ✅ Color-coded examples (Good ✅ / Bad ❌)

### 📚 **Content**
- 10 sections chính về Oracle optimization
- 50+ code examples với giải thích
- Performance comparison tables
- Pre-deploy checklist
- Best practices cho partition, index, query

---

## 🚀 Cách sử dụng

### **Option 1: Mở trực tiếp (Recommended)**
1. Download cả 2 files `.html`
2. Click đúp vào file để mở trong browser
3. Không cần web server, không cần internet!

### **Option 2: Host trên web server**
```bash
# Với Python
python -m http.server 8000

# Với Node.js
npx http-server

# Truy cập: http://localhost:8000/oracle-convention-en.html
```

### **Option 3: Deploy lên GitHub Pages**
1. Push files lên GitHub repo
2. Enable GitHub Pages trong Settings
3. Share link với team

---

## 🎯 Features chi tiết

### 🌓 **Dark Mode**
- Click vào nút **🌙 Dark Mode** ở góc phải trên
- Theme được lưu tự động (localStorage)
- Mở lại page vẫn giữ theme đã chọn
- Responsive - hoạt động tốt trên mobile

**Keyboard shortcuts:** (Có thể thêm sau)
- `Ctrl/Cmd + D` - Toggle dark mode

### 🌐 **Language Switcher**
- Click vào **🇻🇳 Tiếng Việt** để chuyển sang Vietnamese
- Click vào **🇺🇸 English** để chuyển về English
- Files hoạt động độc lập (offline-ready)

### 📖 **Navigation**
- **Table of Contents** - Click để jump tới section
- **Smooth scroll** - Animation mượt mà
- **Back to top** - Nút ↑ xuất hiện khi scroll xuống

---

## 🎨 Dark Mode Preview

### Light Mode 🌞
```
- Background: Gradient blue/white
- Code blocks: Dark background
- Headers: Oracle Red gradient
- Text: Dark gray
```

### Dark Mode 🌙
```
- Background: Deep blue/purple gradient
- Code blocks: Almost black
- Headers: Darker red gradient  
- Text: Light gray
- Callouts: Darker tones with vibrant borders
```

---

## 📱 Responsive Design

### Desktop (> 1024px)
- Full width container (1200px max)
- 2-column comparison grids
- Visible labels on buttons

### Tablet (768px - 1024px)
- Slightly narrower container
- 2-column grids maintained
- Adjusted padding

### Mobile (< 768px)
- Single column layout
- Stacked comparison grids
- Collapsed button labels (icon only)
- Larger touch targets

---

## 🔧 Troubleshooting

### ❓ **Link tiếng Việt không hoạt động**
**Nguyên nhân:** Cả 2 files phải ở cùng thư mục

**Giải pháp:**
```
✅ ĐÚNG:
📁 my-docs/
├── oracle-convention-en.html
└── oracle-convention-vi.html

❌ SAI:
📁 my-docs/
├── 📁 english/
│   └── oracle-convention-en.html
└── 📁 vietnamese/
    └── oracle-convention-vi.html
```

### ❓ **Dark mode không lưu**
- Kiểm tra browser có enable localStorage
- Clear browser cache và thử lại
- Thử browser khác (Chrome, Firefox, Edge)

### ❓ **Scroll không smooth**
- Một số browser cũ không support smooth scroll
- Vẫn hoạt động, chỉ không có animation

---

## 📤 Share với Team

### **Email**
```
Subject: [Oracle] Database Code Convention - NEW!

Hi team,

Tôi đã tạo một bộ convention mới cho Oracle development.

📄 Files đính kèm:
- oracle-convention-en.html (English)
- oracle-convention-vi.html (Tiếng Việt)

✨ Features:
- 🌙 Dark mode
- 📱 Mobile friendly
- 🎨 Color-coded examples
- ⚡ Performance tips

Mở file HTML trong browser để xem (không cần internet).

Best regards,
```

### **Slack/Teams**
```
🗄️ NEW: Oracle Database Code Convention

📖 Comprehensive guide for writing high-performance SQL
🎯 Covers: Index strategy, Partition optimization, Clean code
🌙 Dark mode included!
📱 Mobile friendly

Download & open in browser: [Attach files]
```

### **Confluence/Internal Wiki**
1. Upload cả 2 files lên Confluence
2. Tạo page mới
3. Insert macro "HTML"
4. Link tới files

---

## 🛠️ Customization

### **Thay đổi màu sắc**
Edit CSS variables trong `<style>` tag:

```css
:root {
    --primary-color: #F80000;     /* Oracle Red */
    --secondary-color: #C74634;   /* Darker Red */
    --accent-color: #00758F;      /* Oracle Blue */
    /* ... thay đổi theo brand của bạn */
}
```

### **Thêm logo công ty**
Tìm phần `<header>` và thêm:

```html
<header>
    <img src="company-logo.png" alt="Logo" style="height: 50px;">
    <h1>🗄️ Oracle Database Code Convention</h1>
    ...
</header>
```

### **Thêm version number**
Edit phần `<footer>`:

```html
<footer>
    <p><strong>Document Version:</strong> 2.0</p>
    <p><strong>Last Updated:</strong> 2024-11-25</p>
    ...
</footer>
```

---

## 📊 Statistics

- **Total Sections:** 12
- **Code Examples:** 50+
- **Tables:** 5
- **Callout Boxes:** 30+
- **File Size:** ~65KB each
- **Load Time:** < 1 second

---

## 🎓 Use Cases

### **1. Onboarding - Fresher/Junior Devs**
- Share files trong welcome package
- Yêu cầu đọc trước khi code review đầu tiên
- Quiz based on content (có thể thêm sau)

### **2. Code Review Reference**
- Mở trong browser khi review code
- Copy link tới specific section
- Show examples trực tiếp

### **3. Training Sessions**
- Display trên projector/screen share
- Walk through examples
- Interactive discussion

### **4. Quick Reference**
- Bookmark trong browser
- Search (Ctrl+F) khi cần
- Print-friendly (có thể print thành PDF)

---

## 🔄 Updates

### Version 1.0 (2024-11)
- ✅ Initial release
- ✅ English & Vietnamese versions
- ✅ 10 main sections
- ✅ 50+ examples

### Version 1.1 (2024-11-25)
- ✅ **Added Dark Mode toggle** 🌙
- ✅ Fixed Vietnamese link
- ✅ Enhanced responsive design
- ✅ localStorage for theme persistence

### Planned (Future)
- ⏳ Interactive quiz
- ⏳ Search functionality
- ⏳ Keyboard shortcuts
- ⏳ PDF export option
- ⏳ Code playground integration

---

## 📞 Support

Nếu có vấn đề hoặc suggestions:
1. Create merge request với ví dụ cụ thể
2. Discuss với team lead
3. Update document sau khi approved

---

## 📄 License

Internal use only - [Your Company Name]

---

**Made with ❤️ for better Oracle development**

*"Write code that your future self will thank you for."* 🚀
