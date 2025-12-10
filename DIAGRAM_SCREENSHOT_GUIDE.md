# 📸 Architecture Diagram - Screenshot Generation Guide

## Quick View

**🌐 View in Browser**: Open `architecture-viewer.html` in any modern web browser to see the interactive diagram.

**📥 Download PNG**: Click the "Download as PNG" button in the viewer to export a high-resolution image.

---

## Method 1: Interactive HTML Viewer (Recommended) ⭐

### Steps:
1. Open `architecture-viewer.html` in your web browser
2. Wait 2-3 seconds for the diagram to render
3. Click the **"📥 Download as PNG"** button
4. The diagram will be downloaded as `zava-ai-platform-architecture.png`

### Features:
- ✅ No installation required
- ✅ Works in Chrome, Firefox, Safari, Edge
- ✅ Interactive viewing with zoom (Ctrl/Cmd +/-)
- ✅ High-resolution export
- ✅ Color-coded layers with legend
- ✅ Print-friendly design

---

## Method 2: Online Mermaid Editor

### Steps:
1. Go to [https://mermaid.live/](https://mermaid.live/)
2. Copy the contents of `final-architecture.mmd`
3. Paste into the editor
4. The diagram will render automatically
5. Click **"Download PNG"** or **"Download SVG"**

### Advantages:
- ✅ No installation
- ✅ Supports SVG export (scalable)
- ✅ Can edit diagram in real-time
- ✅ Share diagram via URL

---

## Method 3: Mermaid CLI (For Developers)

### Prerequisites:
```bash
# Install Node.js (if not already installed)
# https://nodejs.org/

# Install Mermaid CLI globally
npm install -g @mermaid-js/mermaid-cli
```

### Generate PNG:
```bash
# Basic PNG generation
mmdc -i final-architecture.mmd -o architecture.png

# High-resolution PNG with white background
mmdc -i final-architecture.mmd \
     -o zava-ai-platform-architecture.png \
     -b white \
     -w 2400 \
     -H 3000

# Generate SVG (scalable vector)
mmdc -i final-architecture.mmd \
     -o architecture.svg \
     -b white
```

### Advantages:
- ✅ Command-line automation
- ✅ Can be integrated into CI/CD
- ✅ Customizable dimensions
- ✅ Batch processing support

---

## Method 4: Python Script (Automated)

### Prerequisites:
```bash
# Install required packages
pip install requests
```

### Run the script:
```bash
# The script tries multiple methods automatically
python3 /tmp/render_diagram.py
```

### How it works:
1. Tries **Kroki API** (cloud service, no installation)
2. Falls back to **Mermaid CLI** (if installed)
3. Falls back to **Playwright** (browser automation)

### Advantages:
- ✅ Fully automated
- ✅ Multiple fallback methods
- ✅ Works in CI/CD environments
- ✅ Error handling and retry logic

---

## Method 5: VS Code Extension

### Prerequisites:
1. Install VS Code
2. Install "Markdown Preview Mermaid Support" extension
3. Or install "Mermaid Preview" extension

### Steps:
1. Open `final-architecture.mmd` in VS Code
2. Right-click → "Mermaid Preview"
3. Use browser dev tools to export the rendered SVG/PNG

### Advantages:
- ✅ Great for development workflow
- ✅ Live preview while editing
- ✅ Integrated with code editor

---

## Method 6: Playwright/Puppeteer (Programmatic)

### Prerequisites:
```bash
# Install Playwright
pip install playwright
playwright install chromium

# Or use Puppeteer (Node.js)
npm install puppeteer
```

### Python Example:
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page(viewport={'width': 2400, 'height': 3000})
    
    # Load the HTML viewer
    page.goto('file:///path/to/architecture-viewer.html')
    
    # Wait for rendering
    page.wait_for_timeout(3000)
    
    # Screenshot
    page.screenshot(path='architecture.png', full_page=True)
    
    browser.close()
```

### Node.js Example:
```javascript
const puppeteer = require('puppeteer');

(async () => {
  const browser = await puppeteer.launch();
  const page = await browser.newPage();
  await page.setViewport({ width: 2400, height: 3000 });
  
  await page.goto('file:///path/to/architecture-viewer.html');
  await page.waitForTimeout(3000);
  
  await page.screenshot({ 
    path: 'architecture.png', 
    fullPage: true 
  });
  
  await browser.close();
})();
```

---

## Method 7: Docker (Containerized)

### Using Official Mermaid Docker Image:
```bash
# Pull the image
docker pull minlag/mermaid-cli

# Generate diagram
docker run --rm -v $(pwd):/data \
  minlag/mermaid-cli \
  -i /data/final-architecture.mmd \
  -o /data/architecture.png \
  -b white \
  -w 2400
```

### Advantages:
- ✅ No local installation
- ✅ Consistent environment
- ✅ Easy CI/CD integration

---

## Recommended Approach by Use Case

### For Quick Viewing:
→ **Method 1: HTML Viewer** (Just open the file)

### For Sharing:
→ **Method 2: Mermaid.live** (Get shareable URL)

### For Documentation:
→ **Method 3: Mermaid CLI** (High-quality PNG/SVG)

### For CI/CD:
→ **Method 4: Python Script** or **Method 7: Docker**

### For Development:
→ **Method 5: VS Code Extension** (Live preview)

---

## Troubleshooting

### Issue: Diagram doesn't render in HTML viewer

**Solutions:**
- Ensure JavaScript is enabled in your browser
- Use a modern browser (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+)
- Check browser console for errors (F12)
- Try opening in incognito/private mode

### Issue: Mermaid CLI fails with memory error

**Solutions:**
```bash
# Increase Node.js memory limit
export NODE_OPTIONS="--max-old-space-size=4096"

# Then retry
mmdc -i final-architecture.mmd -o output.png
```

### Issue: Fonts look wrong in exported PNG

**Solutions:**
- Install system fonts used in the diagram
- Or specify custom fonts in mmdc config:
```json
{
  "themeVariables": {
    "fontFamily": "Arial, sans-serif"
  }
}
```

### Issue: Diagram is cut off

**Solutions:**
```bash
# Increase dimensions
mmdc -i final-architecture.mmd -o output.png -w 3000 -H 4000

# Or use SVG (scales infinitely)
mmdc -i final-architecture.mmd -o output.svg
```

---

## Screenshot Quality Comparison

| Method | Resolution | Quality | File Size | Ease of Use |
|--------|-----------|---------|-----------|-------------|
| HTML Viewer | 2400x3000 | ⭐⭐⭐⭐⭐ | ~500KB | ⭐⭐⭐⭐⭐ |
| Mermaid.live | 1920x2400 | ⭐⭐⭐⭐ | ~400KB | ⭐⭐⭐⭐⭐ |
| Mermaid CLI | Custom | ⭐⭐⭐⭐⭐ | ~600KB | ⭐⭐⭐ |
| Python Script | Custom | ⭐⭐⭐⭐ | ~500KB | ⭐⭐⭐⭐ |
| VS Code | 1920x2400 | ⭐⭐⭐⭐ | ~400KB | ⭐⭐⭐⭐ |
| Playwright | Custom | ⭐⭐⭐⭐⭐ | ~600KB | ⭐⭐⭐ |
| Docker | Custom | ⭐⭐⭐⭐⭐ | ~600KB | ⭐⭐⭐ |

---

## Automated CI/CD Integration

### GitHub Actions Example:

```yaml
name: Generate Architecture Diagram

on:
  push:
    paths:
      - 'final-architecture.mmd'

jobs:
  generate-diagram:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Install Mermaid CLI
        run: npm install -g @mermaid-js/mermaid-cli
      
      - name: Generate PNG
        run: |
          mmdc -i final-architecture.mmd \
               -o zava-ai-platform-architecture.png \
               -b white \
               -w 2400 \
               -H 3000
      
      - name: Upload artifact
        uses: actions/upload-artifact@v3
        with:
          name: architecture-diagram
          path: zava-ai-platform-architecture.png
      
      - name: Commit diagram
        run: |
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git add zava-ai-platform-architecture.png
          git commit -m "Auto-generated architecture diagram" || exit 0
          git push
```

---

## File Locations

```
📁 Repository Root
├── 📄 final-architecture.mmd          # Mermaid source file
├── 🌐 architecture-viewer.html        # Interactive HTML viewer
├── 📚 ARCHITECTURE.md                 # Comprehensive documentation
├── 🛠️ IMPLEMENTATION_GUIDE.md         # Developer guide
├── ✅ ARCHITECTURE_REVIEW.md          # Architecture review
├── 📖 DIAGRAM_SCREENSHOT_GUIDE.md     # This file
└── 🖼️ zava-ai-platform-architecture.png # Generated screenshot (after export)
```

---

## Support

For questions or issues:
- **Documentation**: See `ARCHITECTURE.md` for architecture details
- **Implementation**: See `IMPLEMENTATION_GUIDE.md` for development setup
- **Mermaid Syntax**: [Official Mermaid Documentation](https://mermaid.js.org/)

---

**Last Updated**: 2025-12-10  
**Maintained By**: Platform Architecture Team
