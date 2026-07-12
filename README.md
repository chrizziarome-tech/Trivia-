<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Live Trivia Challenge</title>
    <style>
        body { font-family: system-ui, sans-serif; background: #f0f2f5; margin: 0; padding: 20px; display: flex; flex-direction: column; align-items: center; }
        .card { background: white; padding: 25px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); width: 100%; max-width: 500px; text-align: center; margin-bottom: 20px; box-sizing: border-box; }
        input, select, button { display: block; width: 100%; margin: 10px 0; padding: 12px; border: 1px solid #ced4da; border-radius: 8px; font-size: 16px; box-sizing: border-box; }
        
        /* NAVIGATION BAR BUTTONS FIX */
        .nav-links { display: flex; justify-content: space-around; width: 100%; max-width: 500px; margin-bottom: 20px; background: white; padding: 10px; border-radius: 8px; box-shadow: 0 2px 6px rgba(0,0,0,0.05); }
        .nav-links button { display: inline-block; width: 30%; background: #e9ecef; color: #007bff; border: none; font-size: 14px; padding: 10px; cursor: pointer; font-weight: bold; margin: 0; }
        .nav-links button:hover { background: #007bff; color: white; }
        
        button { background: #007bff; color: white; cursor: pointer; border: none; font-weight: bold; -webkit-user-select: none; user-select: none; }
        button:hover { background: #0056b3; }
        
        .choice-btn { background: #fff; color: #333; border: 1px solid #ced4da; }
        .choice-btn:hover { background: #e9ecef; }
        .correct { background: #d4edda !important; color: #155724; }
        .wrong { background: #f8d7da !important; color: #721c24; }
        .hidden { display: none; }
        table { width: 100%; border-collapse: collapse; margin-top: 10px; }
        th, td { padding: 10px; border-bottom: 1px solid #dee2e6; text-align: left; }
        th { background: #e9ecef; }
        
        /* PREVENT CHROME TEXT HIGHLIGHTING GLITCH */
        h2, h3, p, th, td, div { -webkit-user-select: none; user-select: none; }
    </style>
</head>
<body>

    <div class="nav-links">
        <button onclick="showView('student-login-view')">Play Game</button>
        <button onclick="showView('leaderboard-view')">Leaderboard</button>
        <button onclick="showView('creator-view')">Dashboard</button>
    </div>

    <!-- STUDENT LOGIN VIEW -->
    <div id="student-login-view" class="card view">
        <h2>Student Registration</h2>
        <input type="text" id="student-name" placeholder="Full Name">
        <input type="text" id="student-grade" placeholder="Grade Level (e.g., Grade 10)">
        <input type="text" id="student-section" placeholder="Section">
        <button onclick="registerStudent()">Start Challenge</button>
    </div>

    <!-- GAME VIEW -->
    <div id="game-view" class="card view hidden">
        <h2 id="welcome-msg"></h2>
        <div id="question-box">
            <h3 id="q-text">Loading Question...</h3>
            <button class="choice-btn" onclick="submitAnswer('A', this)" id="btnA"></button>
            <button class="choice-btn" onclick="submitAnswer('B', this)" id="btnB"></button>
            <button class="choice-btn" onclick="submitAnswer('C', this)" id="btnC"></button>
            <button class="choice-btn" onclick="submitAnswer('D', this)" id="btnD"></button>
            <div id="message" style="margin-top:15px; font-weight:bold;"></div>
        </div>
    </div>

    <!-- PUBLIC LEADERBOARD VIEW -->
    <div id="leaderboard-view" class="card view hidden">
        <h2>⭐ Live Public Leaderboard</h2>
        <table>
            <thead>
                <tr><th>Rank</th><th>Name</th><th>Grade/Sec</th><th>Stars</th></tr>
            </thead>
            <tbody id="leaderboard-data"></tbody>
        </table>
    </div>

    <!-- CREATOR VIEW -->
    <div id="creator-view" class="card view hidden">
        <h2>🛠️ Creator Dashboard</h2>
        <input type="password" id="creator-pass" placeholder="Enter Password to Access">
        <button onclick="verifyCreator()">Verify Password</button>
        
        <div id="creator-form" class="hidden">
            <h3>Add New Trivia Question</h3>
            <input type="text" id="new-q" placeholder="Question Text">
            <input type="text" id="new-a" placeholder="Choice A">
            <input type="text" id="new-b" placeholder="Choice B">
            <input type="text" id="new-c" placeholder="Choice C">
            <input type="text" id="new-d" placeholder="Choice D">
            <select id="new-correct">
                <option value="">Select Correct Choice</option>
                <option value="A">A</option><option value="B">B</option>
                <option value="C">C</option><option value="D">D</option>
            </select>
            <button onclick="addQuestionToServer()" style="background:#28a745;">Post Question</button>
        </div>
    </div>

    <script>
        const API_URL = "https://google.com"; 
        const CREATOR_PASSWORD = "admin";

        let currentQ = null;
        let student = { name: "", grade: "", section: "" };
        let attempted = JSON.parse(localStorage.getItem('attempted_questions') || '[]');

        function showView(viewId) {
            document.querySelectorAll('.view').forEach(v => v.classList.add('hidden'));
            document.getElementById(viewId).classList.remove('hidden');
            if(viewId === 'leaderboard-view') loadLeaderboard();
        }

        function registerStudent() {
            student.name = document.getElementById('student-name').value.trim();
            student.grade = document.getElementById('student-grade').value.trim();
            student.section = document.getElementById('student-section').value.trim();
            if (!student.name || !student.grade || !student.section) return alert("Fill out all fields!");
            
            document.getElementById('welcome-msg').innerText = `Player: ${student.name} (${student.section})`;
            showView('game-view');
            loadQuestion();
        }

        document.addEventListener("visibilitychange", () => {
            if (document.visibilityState === "hidden" && currentQ) {
                lockQuestionLocally(currentQ.id);
                currentQ = null; 
            }
        });
        window.addEventListener("blur", () => {
            if (currentQ) {
                lockQuestionLocally(currentQ.id);
                currentQ = null;
            }
        });

        function lockQuestionLocally(id) {
            if (!attempted.includes(String(id))) {
                attempted.push(String(id));
                localStorage.setItem('attempted_questions', JSON.stringify(attempted));
            }
        }

        async function loadQuestion() {
            document.getElementById('message').innerText = "";
            document.querySelectorAll('.choice-btn').forEach(b => { b.disabled = false; b.className = "choice-btn"; });

            try {
                const response = await fetch(API_URL);
                const questions = await response.json();
                const available = questions.filter(q => q.status === 'open' && !attempted.includes(String(q.id)));

                if (available.length === 0) {
                    document.getElementById('question-box').innerHTML = `<h3>🎉 All Caught Up!</h3><p>No new questions available for you right now.</p>`;
                    return;
                }

                currentQ = available[Math.floor(Math.random() * available.length)];
                document.getElementById('q-text').innerText = currentQ.question;
                document.getElementById('btnA').innerText = `A) ${currentQ.a}`;
                document.getElementById('btnB').innerText = `B) ${currentQ.b}`;
                document.getElementById('btnC').innerText = `C) ${currentQ.c}`;
                document.getElementById('btnD').innerText = `D) ${currentQ.d}`;
            } catch(e) {
                document.getElementById('q-text').innerText = "Database connection error. Check your Google Sheet tabs.";
            }
        }

        async function submitAnswer(reply, btn) {
            lockQuestionLocally(currentQ.id);
            document.querySelectorAll('.choice-btn').forEach(b => b.disabled = true);
            const msg = document.getElementById('message');

            if (reply === currentQ.correct) {
                btn.classList.add('correct');
                msg.innerHTML = "🎉 Correct! +1 Star added to the public board!";
                msg.style.color = "green";
                await fetch(API_URL, {
                    method: "POST",
                    body: JSON.stringify({ id: currentQ.id, action: "solve", ...student })
                });
            } else {
                btn.classList.add('wrong');
                msg.innerHTML = `❌ Wrong! Correct: ${currentQ.correct}. Question locked.`;
                msg.style.color = "red";
            }
            setTimeout(loadQuestion, 3000);
        }

        async function loadLeaderboard() {
            try {
                const res = await fetch(`${API_URL}?type=leaderboard`);
                const data = await res.json();
                data.sort((a,b) => b.stars - a.stars);
                
                const tbody = document.getElementById('leaderboard-data');
                tbody.innerHTML = data.map((user, idx) => `
                    <tr><td>${idx+1}</td><td>${user.name}</td><td>${user.grade} - ${user.section}</td><td>⭐ ${user.stars}</td></tr>
                `).join('');
            } catch(e) {
                document.getElementById('leaderboard-data').innerHTML = "<tr><td colspan='4'>Failed to load scores.</td></tr>";
            }
        }

        function verifyCreator() {
