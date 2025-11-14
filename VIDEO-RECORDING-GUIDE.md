# Video Recording Guide for DSP AI Platform Tutorials

## 🎬 Recording Setup & Best Practices

### Required Software

#### Screen Recording Tools
1. **OBS Studio** (Recommended - Free)
   - Download: https://obsproject.com/
   - Best for: Professional recording with scenes
   - Features: Multiple sources, transitions, streaming

2. **Camtasia** (Professional - Paid)
   - Best for: Built-in editing capabilities
   - Features: Annotations, zoom effects, cursor highlights

3. **Windows Game Bar** (Built-in - Free)
   - Press: Win + G
   - Best for: Quick recordings
   - Limitations: Basic features only

#### Video Editing Software
1. **DaVinci Resolve** (Free)
   - Professional-grade editing
   - Color correction
   - Audio mastering

2. **Adobe Premiere Pro** (Paid)
   - Industry standard
   - Advanced effects
   - Creative Cloud integration

3. **OpenShot** (Free)
   - Simple interface
   - Basic editing features
   - Cross-platform

#### Supporting Tools
- **Audacity**: Audio recording and editing
- **draw.io**: Creating diagrams
- **Excalidraw**: Hand-drawn style diagrams
- **PowerPoint/Keynote**: Slide creation
- **VS Code**: Code demonstrations
- **Windows Terminal**: Command-line demos
- **Postman**: API testing demonstrations

---

## 📋 Pre-Recording Checklist

### Environment Setup
- [ ] Clean desktop - remove unnecessary icons
- [ ] Close unnecessary applications
- [ ] Disable notifications (Windows, Slack, Email)
- [ ] Set display resolution to 1920x1080
- [ ] Increase font sizes for readability
  - Terminal: 14-16pt
  - VS Code: 14-16pt
  - Browser: 110-125% zoom
- [ ] Use high-contrast themes for better visibility
- [ ] Prepare demo data (no sensitive information)

### Audio Setup
- [ ] Test microphone levels (peak around -12dB)
- [ ] Use noise cancellation/suppression
- [ ] Find quiet recording environment
- [ ] Use headphones to prevent echo
- [ ] Do a test recording to check audio quality

### Content Preparation
- [ ] Review script thoroughly
- [ ] Prepare all code snippets
- [ ] Set up demo environments
- [ ] Create visual assets (diagrams, slides)
- [ ] Prepare terminal commands in a text file
- [ ] Test all demos to ensure they work

---

## 🎯 OBS Studio Configuration

### Scene Setup

#### Scene 1: Introduction
- **Sources**:
  - Display Capture (full screen)
  - Image: DSP AI Logo (top-right corner)
  - Text: Episode title (bottom third)

#### Scene 2: Screen Share
- **Sources**:
  - Display Capture or Window Capture
  - Webcam (optional, bottom-right corner)

#### Scene 3: Terminal Focus
- **Sources**:
  - Window Capture (Terminal only)
  - Increased scaling for visibility

#### Scene 4: Code Editor
- **Sources**:
  - Window Capture (VS Code)
  - Browser Source (for live preview)

### Recording Settings
```
Video:
- Base Resolution: 1920x1080
- Output Resolution: 1920x1080
- FPS: 30
- Video Bitrate: 8000-10000 Kbps
- Encoder: x264 or NVENC (if available)

Audio:
- Sample Rate: 48 kHz
- Channels: Stereo
- Bitrate: 160 Kbps

Output:
- Recording Format: MP4
- Recording Quality: High Quality, Medium File Size
- Recording Path: D:\dsp-videos\recordings\
```

### Hotkeys Configuration
- Start/Stop Recording: F9
- Pause Recording: F10
- Switch Scenes: F1-F4
- Mute/Unmute Mic: F5

---

## 🎤 Recording Best Practices

### Voice & Narration
1. **Pacing**
   - Speak clearly and at moderate pace
   - Pause between major points
   - Allow time for viewers to absorb information

2. **Tone**
   - Enthusiastic but professional
   - Conversational, not monotone
   - Emphasize key points

3. **Script Reading**
   - Practice beforehand
   - Don't read word-for-word
   - Use script as guide, speak naturally

### Visual Presentation
1. **Mouse Movement**
   - Move cursor deliberately
   - Highlight important areas
   - Avoid unnecessary movement

2. **Transitions**
   - Pause before switching screens
   - Announce what you're about to show
   - Give context for transitions

3. **Code Demonstration**
   - Type at readable pace
   - Explain while typing
   - Show complete code before running

### Common Mistakes to Avoid
- ❌ Recording with notifications on
- ❌ Using personal/sensitive data
- ❌ Moving too fast through content
- ❌ Forgetting to explain keyboard shortcuts
- ❌ Not testing demos beforehand
- ❌ Poor audio quality
- ❌ Inconsistent volume levels

---

## 📹 Recording Workflow

### Step 1: Setup (10 minutes)
1. Launch OBS Studio
2. Check all scenes are configured
3. Do audio level check
4. Clear terminal history
5. Open required applications
6. Position windows correctly

### Step 2: Test Recording (5 minutes)
1. Record 30-second test clip
2. Review video and audio quality
3. Adjust settings if needed
4. Delete test recording

### Step 3: Main Recording
1. Take a deep breath, relax
2. Start recording (F9)
3. Wait 2 seconds before speaking
4. Follow script naturally
5. If you make a mistake:
   - Pause briefly
   - Repeat the section
   - Edit out mistakes later
6. End with 2 seconds of silence
7. Stop recording (F9)

### Step 4: Review
1. Watch entire recording
2. Note timestamps for edits
3. Check audio levels throughout
4. Verify all demonstrations work

---

## ✂️ Post-Production Workflow

### Basic Editing Tasks
1. **Trim beginning and end**
2. **Remove mistakes and long pauses**
3. **Add intro/outro sequences**
4. **Insert title cards between sections**
5. **Add background music (subtle, -20dB)**
6. **Include captions/subtitles**

### Visual Enhancements
1. **Zoom effects** for important areas
2. **Highlight boxes** around key elements
3. **Arrows and annotations**
4. **Blur sensitive information**
5. **Speed up repetitive tasks**
6. **Add transitions between scenes**

### Audio Processing
1. **Normalize audio levels**
2. **Remove background noise**
3. **Add compression for consistent volume**
4. **EQ for voice clarity**
5. **Remove mouth sounds/breathing**

### Export Settings
```
Format: MP4 (H.264)
Resolution: 1920x1080
Frame Rate: 30 FPS
Bitrate: 8-10 Mbps
Audio: AAC, 48kHz, 160kbps
```

---

## 📊 Quality Checklist

### Before Publishing
- [ ] Video is 1080p quality
- [ ] Audio is clear and consistent
- [ ] All demonstrations work correctly
- [ ] No sensitive information visible
- [ ] Captions are accurate
- [ ] Timestamps in description
- [ ] Links to resources work
- [ ] Thumbnail is attractive
- [ ] Title is descriptive
- [ ] Description includes key points

### Accessibility
- [ ] Closed captions available
- [ ] Good color contrast
- [ ] Font size readable on mobile
- [ ] Audio description for visual elements
- [ ] Transcript available

---

## 🎨 Visual Assets Templates

### Thumbnail Template
- Size: 1280x720px
- Include: Episode number, title, DSP logo
- Colors: Consistent with brand
- Font: Clear, readable at small size

### Title Card Template
- Duration: 3-5 seconds
- Include: Episode title, number, series name
- Animation: Subtle fade or slide

### Lower Third Template
- Position: Bottom 20% of screen
- Include: Speaker name, topic
- Duration: 5-10 seconds when introduced

---

## 📝 Sample Recording Script Format

```markdown
# Episode: [EPISODE_CODE]
# Title: [EPISODE_TITLE]
# Duration: [TARGET_DURATION]

## [TIMESTAMP] Section Name
**Visual**: [What should be on screen]
**Action**: [What to do/click/type]
**Script**: "[What to say]"

### Key Points:
- Point 1
- Point 2
- Point 3

### Commands to Run:
```bash
command 1
command 2
```
```

---

## 🚀 Quick Start Commands

### Windows Terminal Setup
```powershell
# Set font size
# Settings > Profiles > Defaults > Appearance > Font size: 16

# Clear terminal
Clear-Host

# Set window title
$host.UI.RawUI.WindowTitle = "DSP AI Platform Demo"
```

### Demo Environment
```bash
# Start all services
docker-compose up -d

# Check services are running
docker ps

# Tail logs
docker-compose logs -f

# Clean environment after recording
docker-compose down -v
```

---

## 📚 Additional Resources

### Learning Resources
- [OBS Studio Tutorials](https://obsproject.com/wiki/)
- [DaVinci Resolve Training](https://www.blackmagicdesign.com/products/davinciresolve/training)
- [YouTube Creator Academy](https://creatoracademy.youtube.com/)

### Asset Libraries
- [Unsplash](https://unsplash.com/) - Free stock images
- [Freesound](https://freesound.org/) - Free audio effects
- [YouTube Audio Library](https://studio.youtube.com/channel/UC/music) - Free music

### Tools
- [Handbrake](https://handbrake.fr/) - Video compression
- [ShareX](https://getsharex.com/) - Screenshot tool
- [KeyCastOW](https://github.com/brookhong/KeyCastOW) - Show keypresses

---

## 📧 Support & Feedback

For questions or feedback about video production:
- Create an issue in the repository
- Contact the DSP AI Platform team
- Share your recordings for review

---

**Happy Recording! 🎬**

Remember: The goal is to create helpful, clear tutorials that empower users to successfully use the DSP AI Platform. Quality over quantity!
