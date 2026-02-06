<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Word Chain Game - Peaceful Edition</title>
    <link href="https://fonts.googleapis.com/css2?family=Tajawal:wght@300;500;700&display=swap" rel="stylesheet">
    <style>
        :root {
            --bg-color: #f0f2f5;
            --card-bg: #ffffff;
            --accent-soft: #8e94f2; /* لافندر هادئ */
            --accent-deep: #4a5568; /* رمادي داكن مريح */
            --text-main: #2d3748;
            --success-soft: #81c784;
            --error-soft: #e57373;
            --shadow: 0 10px 25px rgba(0,0,0,0.05);
        }

        body {
            font-family: 'Tajawal', sans-serif;
            background-color: var(--bg-color);
            color: var(--text-main);
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            margin: 0;
        }

        .game-card {
            background: var(--card-bg);
            width: 90%;
            max-width: 550px;
            padding: 2.5rem;
            border-radius: 30px;
            box-shadow: var(--shadow);
            text-align: center;
            border: 1px solid rgba(255,255,255,0.8);
        }

        .stats-bar {
            display: flex;
            justify-content: space-between;
            font-weight: 500;
            color: var(--accent-deep);
            margin-bottom: 2rem;
            background: #f8fafc;
            padding: 10px 20px;
            border-radius: 15px;
        }

        .main-letter-box {
            display: inline-block;
            width: 100px;
            height: 100px;
            line-height: 100px;
            background: var(--accent-soft);
            color: white;
            font-size: 3.5rem;
            font-weight: 700;
            border-radius: 20px;
            margin-bottom: 1.5rem;
            box-shadow: 0 8px 15px rgba(142, 148, 242, 0.3);
            transition: transform 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
        }

        .input-area {
            margin: 20px 0;
        }

        input {
            width: 100%;
            padding: 15px;
            border: 2px solid #e2e8f0;
            border-radius: 15px;
            font-size: 1.2rem;
            text-align: center;
            transition: all 0.3s ease;
            outline: none;
            font-family: inherit;
        }

        input:focus {
            border-color: var(--accent-soft);
            box-shadow: 0 0 0 4px rgba(142, 148, 242, 0.1);
        }

        .history-container {
            margin-top: 25px;
            max-height: 180px;
            overflow-y: auto;
            display: flex;
            flex-wrap: wrap;
            gap: 10px;
            justify-content: center;
            padding: 10px;
        }

        .word-bubble {
            padding: 8px 16px;
            border-radius: 12px;
            font-size: 0.95rem;
            display: flex;
            flex-direction: column;
            animation: slideUp 0.4s ease;
        }

        .player-word { background: #e0e7ff; color: #4338ca; }
        .ai-word { background: #fef3c7; color: #92400e; }

        @keyframes slideUp {
            from { opacity: 0; transform: translateY(10px); }
            to { opacity: 1; transform: translateY(0); }
        }

        .status-text { height: 20px; font-size: 0.9rem; margin-top: 10px; }
    </style>
</head>
<body>

<div class="game-card">
    <div class="stats-bar">
        <span>🏆 الأعلى: <span id="highScore">0</span></span>
        <span>⭐ النقاط: <span id="currentScore">0</span></span>
    </div>

    <p style="color: #718096;">يجب أن تبدأ الكلمة بالحرف:</p>
    <div id="targetLetter" class="main-letter-box">?</div>

    <div class="input-area">
        <input type="text" id="wordInput" placeholder="اكتب كلمتك هنا..." autocomplete="off">
        <div id="statusMsg" class="status-text"></div>
    </div>

    <div class="history-container" id="history"></div>
</div>

<script>
    let score = 0;
    let highScore = localStorage.getItem('chainScore') || 0;
    let usedWords = new Set();
    let currentLetter = "";
    
    // قائمة كلمات احتياطية واسعة لكل حرف لضمان عدم توقف اللعبة
    const backupLexicon = {
        a: ["apple", "amazing", "ancient", "adventure"],
        b: ["bright", "beautiful", "brave", "butterfly"],
        c: ["cloud", "creative", "classic", "courage"],
        d: ["dream", "dolphin", "diamond", "dynamic"],
        e: ["energy", "elegant", "eagle", "element"],
        // يمكن إضافة المزيد هنا...
    };

    const input = document.getElementById('wordInput');
    const targetUI = document.getElementById('targetLetter');
    const historyUI = document.getElementById('history');

    document.getElementById('highScore').innerText = highScore;

    function init() {
        const alphabet = "abcdefghijklmnopqrstuvwxyz";
        currentLetter = alphabet[Math.floor(Math.random() * alphabet.length)];
        targetUI.innerText = currentLetter.toUpperCase();
    }

    input.addEventListener('keypress', (e) => {
        if (e.key === 'Enter') {
            handleTurn(input.value.toLowerCase().trim());
        }
    });

    async function handleTurn(word) {
        if (!word || word[0] !== currentLetter || usedWords.has(word)) {
            flashError();
            return;
        }

        // التحقق من صحة الكلمة (API)
        const isValid = await validateWord(word);
        if (!isValid) return;

        addWordToUI(word, 'player');
        input.value = "";
        
        // دور الكمبيوتر
        setTimeout(() => aiResponse(word.slice(-1)), 800);
    }

    async function aiResponse(letter) {
        currentLetter = letter;
        targetUI.style.transform = "scale(1.1)";
        targetUI.innerText = letter.toUpperCase();
        setTimeout(() => targetUI.style.transform = "scale(1)", 200);

        // محاولة جلب كلمة من API
        let response = await fetch(`https://api.datamuse.com/words?sp=${letter}*&max=20`);
        let data = await response.json();
        let wordObj = data.find(w => !usedWords.has(w.word) && w.word.length > 2);
        
        let chosenWord = wordObj ? wordObj.word : (backupLexicon[letter] ? backupLexicon[letter][0] : letter + "at");

        addWordToUI(chosenWord, 'ai');
        currentLetter = chosenWord.slice(-1);
        targetUI.innerText = currentLetter.toUpperCase();
    }

    async function addWordToUI(word, owner) {
        usedWords.add(word);
        
        // الترجمة
        const trans = await translateWord(word);
        
        const div = document.createElement('div');
        div.className = `word-bubble ${owner}-word`;
        div.innerHTML = `<strong>${word}</strong><small>${trans}</small>`;
        historyUI.prepend(div);

        // نطق بصوت أنثوي
        speak(word);

        if (owner === 'player') {
            score++;
            document.getElementById('currentScore').innerText = score;
            if (score > highScore) {
                highScore = score;
                localStorage.setItem('chainScore', highScore);
                document.getElementById('highScore').innerText = highScore;
            }
        }
    }

    function speak(text) {
        const msg = new SpeechSynthesisUtterance(text);
        const voices = window.speechSynthesis.getVoices();
        // محاولة اختيار صوت أنثوي (غالباً يحتوي الاسم على Google US English أو Microsoft Zira)
        const femaleVoice = voices.find(v => v.name.includes('Female') || v.name.includes('Google US English') || v.name.includes('Zira'));
        if (femaleVoice) msg.voice = femaleVoice;
        msg.pitch = 1.2; 
        msg.rate = 0.9;
        window.speechSynthesis.speak(msg);
    }

    async function translateWord(word) {
        try {
            const res = await fetch(`https://api.mymemory.translated.net/get?q=${word}&langpair=en|ar`);
            const data = await res.json();
            return data.responseData.translatedText;
        } catch { return "..."; }
    }

    async function validateWord(word) {
        const res = await fetch(`https://api.dictionaryapi.dev/api/v2/entries/en/${word}`);
        if (!res.ok) {
            document.getElementById('statusMsg').innerText = "كلمة غير معروفة!";
            document.getElementById('statusMsg').style.color = "var(--error-soft)";
            return false;
        }
        document.getElementById('statusMsg').innerText = "";
        return true;
    }

    function flashError() {
        input.style.borderColor = "var(--error-soft)";
        setTimeout(() => input.style.borderColor = "#e2e8f0", 500);
    }

    // انتظر تحميل الأصوات
    window.speechSynthesis.onvoiceschanged = () => { /* تم تحميل الأصوات */ };
    init();

</script>
</body>
</html>
