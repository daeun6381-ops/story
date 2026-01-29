<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>STORY💕</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    
    <!-- Firebase SDK & App Logic -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, doc, setDoc, query } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 1. Firebase 설정
        const firebaseConfig = {
            apiKey: "AIzaSyATTfsDTspXM6iHreZCyO_Apk4GWIk74JM",
            authDomain: "story2-60345.firebaseapp.com",
            projectId: "story2-60345",
            storageBucket: "story2-60345.firebasestorage.app",
            messagingSenderId: "64911818684",
            appId: "1:64911818684:web:fc18e4a06917b8f38435b9",
            measurementId: "G-VVLLW4VWCJ"
        };

        // 2. Firebase 초기화
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'couple-diary-html';
        const ANNIVERSARY_DATE = new Date('2025-05-28');
        const SECRET_CODE = "EVOL";

        let photos = [];
        let currentTheme = 'pink';
        let currentView = 'home';
        let isDataSubscribed = false;

        const THEMES = {
            pink: { primary: 'bg-pink-500', text: 'text-pink-600', light: 'bg-pink-50', border: 'border-pink-200', bg: 'bg-[#FFF9FB]' },
            blue: { primary: 'bg-blue-500', text: 'text-blue-600', light: 'bg-blue-50', border: 'border-blue-200', bg: 'bg-[#F9FBFF]' },
            green: { primary: 'bg-emerald-500', text: 'text-emerald-600', light: 'bg-emerald-50', border: 'border-emerald-200', bg: 'bg-[#F9FFF9]' },
            dark: { primary: 'bg-gray-800', text: 'text-gray-800', light: 'bg-gray-100', border: 'border-gray-300', bg: 'bg-[#F5F5F5]' }
        };

        const getTodayFormatted = () => {
            const now = new Date();
            return `${now.getFullYear()}.${String(now.getMonth() + 1).padStart(2, '0')}.${String(now.getDate()).padStart(2, '0')}`;
        };

        const calculateDDay = (dateStr) => {
            const target = new Date(dateStr);
            const diff = Math.floor((target - ANNIVERSARY_DATE) / (1000 * 60 * 60 * 24)) + 1;
            return diff >= 0 ? `D+${diff}` : `D${diff}`;
        };

        // 메인 화면으로 전환하는 함수
        function enterApp() {
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('app-screen').classList.remove('hidden');
            startSubscribingData();
            updateUI();
        }

        function updateUI() {
            const theme = THEMES[currentTheme];
            document.body.className = `min-h-screen ${theme.bg} transition-colors duration-500 pb-24`;
            
            const ddayEl = document.getElementById('header-dday');
            if(ddayEl) {
                ddayEl.innerText = calculateDDay(new Date());
                ddayEl.className = `text-xl font-black ${theme.text} uppercase`;
            }
            
            // SVG 요소의 클래스 설정 오류 수정 (setAttribute 사용)
            const heartEl = document.getElementById('header-heart');
            if(heartEl) {
                heartEl.setAttribute('class', `${theme.text} fill-current w-[18px] h-[18px]`);
            }

            ['home', 'gallery', 'theme'].forEach(view => {
                const btn = document.getElementById(`tab-${view}`);
                if (btn) {
                    btn.className = (view === currentView) 
                        ? `px-6 py-2.5 rounded-xl text-sm font-black bg-white shadow-md ${theme.text}`
                        : `px-6 py-2.5 rounded-xl text-sm font-black text-gray-400 hover:${theme.text}`;
                }
            });

            renderContent();
            lucide.createIcons();
        }

        window.setView = (view) => {
            currentView = view;
            updateUI();
        };

        window.changeTheme = async (themeId) => {
            currentTheme = themeId;
            updateUI();
            try {
                const themeDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance');
                await setDoc(themeDocRef, { themeId }, { merge: true });
            } catch(e) { console.error("Theme save error:", e); }
        };

        function renderContent() {
            const container = document.getElementById('main-content');
            if(!container) return;
            const theme = THEMES[currentTheme];
            
            if (currentView === 'home') {
                container.innerHTML = `
                    <div class="space-y-12 animate-fade-in">
                        <section class="space-y-6">
                            <div class="flex items-end justify-between px-2">
                                <h2 class="text-2xl font-black text-gray-800 tracking-tighter">STORY</h2>
                                <span class="text-sm font-bold opacity-40">옆으로 밀어보세요</span>
                            </div>
                            <div class="flex overflow-x-auto gap-6 pb-8 px-2 no-scrollbar snap-x">
                                ${photos.map(p => `
                                    <div class="snap-start shrink-0 w-[280px] h-[400px] relative bg-white rounded-3xl overflow-hidden shadow-lg border border-gray-100">
                                        <img src="${p.url}" class="w-full h-full object-cover">
                                        <div class="absolute top-4 left-4 ${theme.primary} text-white px-4 py-1.5 rounded-full text-xs font-black shadow-md">${calculateDDay(p.date)}</div>
                                        <div class="absolute inset-x-0 bottom-0 bg-gradient-to-t from-black/80 p-6">
                                            <p class="text-white font-bold truncate mb-1">${p.caption}</p>
                                            <p class="text-white/60 text-[10px] uppercase font-bold tracking-widest">${p.date}</p>
                                        </div>
                                    </div>
                                `).join('') || `<div class="w-full py-20 text-center opacity-30 font-bold">아직 사진이 없어요</div>`}
                            </div>
                        </section>

                        <section class="max-w-xl mx-auto bg-white rounded-[2.5rem] p-8 shadow-xl border ${theme.border}">
                            <div class="flex items-center gap-3 mb-8">
                                <div class="p-3 ${theme.light} rounded-xl"><i data-lucide="camera" class="${theme.text}"></i></div>
                                <div>
                                    <h3 class="text-lg font-black text-gray-800">새로운 기록 남기기</h3>
                                    <p class="text-xs text-gray-400 font-bold">소중한 순간은 잊기 전에 기록해요</p>
                                </div>
                            </div>
                            <div class="space-y-4">
                                <div class="space-y-2">
                                    <label class="text-[10px] font-black text-gray-400 ml-2 uppercase">날짜 선택</label>
                                    <input type="date" id="upload-date" value="${new Date().toISOString().split('T')[0]}" class="w-full p-4 bg-gray-50 rounded-2xl border-2 border-gray-100 outline-none focus:border-gray-300 font-bold text-gray-600 transition-all">
                                </div>
                                <div class="space-y-2">
                                    <label class="text-[10px] font-black text-gray-400 ml-2 uppercase">코멘트</label>
                                    <input type="text" id="upload-caption" placeholder="기억하고 싶은 내용" class="w-full p-4 bg-gray-50 rounded-2xl border-2 border-gray-100 outline-none focus:border-gray-300 font-bold text-gray-600 transition-all">
                                </div>
                                <label class="block mt-6">
                                    <div id="upload-btn" class="w-full py-5 ${theme.primary} text-white rounded-2xl font-black text-center cursor-pointer hover:scale-[1.02] active:scale-95 transition-all shadow-lg">
                                        사진 올려서 기록하기
                                    </div>
                                    <input type="file" id="file-input" class="hidden" accept="image/*">
                                </label>
                            </div>
                        </section>
                    </div>
                `;
                document.getElementById('file-input').onchange = handleFileUpload;
            } else if (currentView === 'gallery') {
                container.innerHTML = `
                    <div class="grid grid-cols-2 md:grid-cols-4 gap-4 animate-fade-in">
                        ${photos.map(p => `
                            <div class="aspect-square relative bg-white rounded-2xl overflow-hidden shadow-md group">
                                <img src="${p.url}" class="w-full h-full object-cover transition-transform group-hover:scale-110 duration-500">
                                <div class="absolute inset-0 bg-gradient-to-t from-black/60 opacity-0 group-hover:opacity-100 transition-opacity p-4 flex flex-col justify-end">
                                    <p class="text-white text-xs font-bold truncate">${p.caption}</p>
                                    <p class="text-white/60 text-[8px] font-bold">${p.date}</p>
                                </div>
                            </div>
                        `).join('')}
                    </div>
                `;
            } else if (currentView === 'theme') {
                container.innerHTML = `
                    <div class="max-w-md mx-auto space-y-4 animate-fade-in text-center">
                        <h2 class="text-xl font-black text-gray-800">테마 설정</h2>
                        <p class="text-xs text-gray-400 font-bold mb-8 uppercase tracking-widest">Select Vibe</p>
                        ${Object.entries(THEMES).map(([id, t]) => `
                            <button onclick="changeTheme('${id}')" class="w-full p-6 bg-white rounded-[2rem] border-4 ${currentTheme === id ? t.border + ' ' + t.light : 'border-gray-50'} flex items-center justify-between transition-all hover:scale-[1.02]">
                                <div class="flex items-center gap-4">
                                    <div class="w-8 h-8 ${t.primary} rounded-full shadow-inner"></div>
                                    <span class="font-black text-gray-700 uppercase tracking-tighter">${id}</span>
                                </div>
                                ${currentTheme === id ? '<i data-lucide="check" class="text-gray-400"></i>' : ''}
                            </button>
                        `).join('')}
                    </div>
                `;
            }
            lucide.createIcons();
        }

        async function handleFileUpload(e) {
            const file = e.target.files[0];
            if (!file) return;
            
            const btn = document.getElementById('upload-btn');
            const originalText = btn.innerText;
            btn.innerText = "업로드 중...";
            btn.style.opacity = "0.5";

            const reader = new FileReader();
            reader.onloadend = async () => {
                const base64 = reader.result;
                const caption = document.getElementById('upload-caption').value || "행복한 순간";
                const date = document.getElementById('upload-date').value;

                try {
                    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), {
                        url: base64,
                        caption,
                        date,
                        timestamp: Date.now()
                    });
                } catch (err) {
                    alert("Firebase 권한(Rules)을 확인해주세요.");
                }

                btn.innerText = originalText;
                btn.style.opacity = "1";
                document.getElementById('upload-caption').value = "";
                updateUI();
            };
            reader.readAsDataURL(file);
        }

        function startSubscribingData() {
            if (isDataSubscribed) return;
            isDataSubscribed = true;

            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), (snap) => {
                photos = snap.docs.map(d => d.data()).sort((a,b) => new Date(b.date) - new Date(a.date));
                updateUI();
            }, (err) => console.error("Firestore error:", err));

            onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance'), (snap) => {
                if (snap.exists()) {
                    currentTheme = snap.data().themeId || 'pink';
                    updateUI();
                }
            }, (err) => console.error("Theme snapshot error:", err));
        }

        onAuthStateChanged(auth, async (user) => {
            if (user) {
                if (localStorage.getItem('isAuth') === 'true') {
                    enterApp();
                }
            } else {
                try {
                    await signInAnonymously(auth);
                } catch (err) {
                    console.error("익명 로그인 실패:", err.message);
                }
            }
        });

        window.checkCode = () => {
            const val = document.getElementById('pass-input').value.toUpperCase();
            const errEl = document.getElementById('error-msg');
            
            if (val === SECRET_CODE) {
                localStorage.setItem('isAuth', 'true');
                if (errEl) errEl.classList.add('hidden');
                enterApp();
            } else {
                if(errEl) {
                    errEl.innerText = "코드가 올바르지 않아요!";
                    errEl.classList.remove('hidden');
                }
            }
        };

        document.addEventListener('DOMContentLoaded', () => {
            const loginDateEl = document.getElementById('login-date');
            if(loginDateEl) loginDateEl.innerText = `2025.05.28 ~ ${getTodayFormatted()}`;
            lucide.createIcons();
            
            document.getElementById('pass-input').addEventListener('keypress', (e) => {
                if (e.key === 'Enter') window.checkCode();
            });
        });

    </script>
    <style>
        @import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
        .animate-fade-in { animation: fadeIn 0.4s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
        body { font-family: 'Pretendard', sans-serif; -webkit-tap-highlight-color: transparent; }
    </style>
</head>
<body class="bg-gray-50">

    <!-- 로그인 화면 -->
    <div id="login-screen" class="fixed inset-0 bg-pink-50 z-[100] flex items-center justify-center p-6">
        <div class="bg-white p-10 rounded-[3rem] shadow-2xl w-full max-w-sm text-center border-4 border-white">
            <div class="w-20 h-20 bg-pink-50 rounded-full flex items-center justify-center mx-auto mb-6 shadow-inner">
                <i data-lucide="lock" class="text-pink-400 w-10 h-10"></i>
            </div>
            <h1 class="text-2xl font-black mb-1 uppercase tracking-tighter text-gray-800">INPUT CODE</h1>
            <p id="login-date" class="text-gray-400 mb-10 text-[10px] font-bold tracking-widest uppercase"></p>
            
            <div class="space-y-4">
                <input type="password" id="pass-input" placeholder="비밀 코드를 입력하세요" class="w-full px-6 py-4 bg-gray-50 border-2 border-gray-100 rounded-2xl focus:border-pink-200 outline-none text-center font-black tracking-[0.3em]">
                <p id="error-msg" class="text-red-400 text-[10px] font-bold hidden"></p>
                <button onclick="checkCode()" class="w-full bg-pink-500 hover:bg-pink-600 text-white font-black py-5 rounded-2xl shadow-xl transition-all active:scale-95">입장하기</button>
            </div>
            <p class="mt-6 text-[9px] text-gray-300 font-bold uppercase tracking-tight">Our Secret Diary Space</p>
        </div>
    </div>

    <!-- 앱 메인 화면 -->
    <div id="app-screen" class="hidden">
        <header class="bg-white/70 backdrop-blur-lg border-b sticky top-0 z-50 px-6 py-4 flex justify-between items-center">
            <div class="flex items-center gap-2">
                <!-- 하트 아이콘 id 유지 -->
                <i id="header-heart" data-lucide="heart"></i>
                <h1 id="header-dday"></h1>
            </div>
            <button onclick="localStorage.removeItem('isAuth'); location.reload();" class="p-2 text-gray-300 hover:text-gray-600 transition-colors">
                <i data-lucide="log-out" size="18"></i>
            </button>
        </header>

        <main class="max-w-4xl mx-auto p-6 space-y-10">
            <div class="flex bg-gray-100/50 p-1.5 rounded-2xl w-fit mx-auto border border-white shadow-inner">
                <button id="tab-home" onclick="setView('home')" class="px-6 py-2.5 rounded-xl text-sm font-black transition-all">홈</button>
                <button id="tab-gallery" onclick="setView('gallery')" class="px-6 py-2.5 rounded-xl text-sm font-black transition-all">갤러리</button>
                <button id="tab-theme" onclick="setView('theme')" class="px-6 py-2.5 rounded-xl text-sm font-black transition-all">테마</button>
            </div>

            <div id="main-content" class="pb-10"></div>
        </main>
    </div>

</body>
</html>
