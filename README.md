<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AX 애자일 리더십 진단</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Pretendard:wght@400;600;700;900&display=swap');
        body { font-family: 'Pretendard', sans-serif; background-color: #f8fafc; color: #1e293b; -webkit-tap-highlight-color: transparent; }
        .radar-chart { filter: drop-shadow(0 4px 10px rgba(79, 70, 229, 0.1)); }
        .animate-in { animation: fadeIn 0.5s ease-out forwards; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(10px); } to { opacity: 1; transform: translateY(0); } }
    </style>
</head>
<body class="min-h-screen flex items-center justify-center p-4">

    <div id="app-container" class="max-w-xl w-full bg-white rounded-[2.5rem] shadow-2xl overflow-hidden min-h-[650px] flex flex-col relative border border-slate-100">
        <!-- JS에 의해 로딩 화면이 먼저 표시됩니다 -->
        <div class="flex items-center justify-center flex-grow">
            <div class="animate-spin rounded-full h-12 w-12 border-b-2 border-indigo-600"></div>
        </div>
    </div>

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, query, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- Configuration ---
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'agile-leadership-app-v2';

        let user = null;
        let userName = "";
        let answers = [];
        let currentIdx = 0;
        let allResults = [];
        let unsubscribeAdmin = null;

        const QUESTIONS = [
            { cat: "협업 & 적응", q: "나는 개인의 성과보다 팀 전체의 협업 결과물을 더 우선시한다." },
            { cat: "협업 & 적응", q: "시장 상황이 변하면 기존 계획이라도 즉시 수정할 용기가 있다." },
            { cat: "협업 & 적응", q: "팀원들이 타 부서와 자유롭게 소통하고 협력하도록 장려한다." },
            { cat: "협업 & 적응", q: "'기존 관습'보다 '현재 상황에 가장 적합한가'를 먼저 고민한다." },
            { cat: "권한 위임", q: "나는 업무의 과정(How)보다 목표와 결과(What/Why)에 집중한다." },
            { cat: "권한 위임", q: "팀원들이 스스로 결정을 내릴 수 있도록 실질적인 권한을 부여한다." },
            { cat: "권한 위임", q: "진행 상황을 일일이 보고받기보다 자율 공유 시스템을 신뢰한다." },
            { cat: "권한 위임", q: "팀원의 실수에 대해 질책하기보다 배운 점을 함께 논의한다." },
            { cat: "서번트 리더십", q: "나의 주 역할은 팀원의 업무 방해 요소를 제거해 주는 것이다." },
            { cat: "서번트 리더십", q: "지시를 내리기 전 팀원들의 의견을 먼저 경청하고 수렴한다." },
            { cat: "서번트 리더십", q: "팀원의 성장이 곧 조직의 경쟁력이라 믿고 발전을 지원한다." },
            { cat: "서번트 리더십", q: "나는 팀의 관리자가 아니라 목표 달성을 돕는 조력자다." },
            { cat: "단기 반복", q: "장기 계획보다 2~4주 단위의 단기 목표(Sprint)를 선호한다." },
            { cat: "단기 반복", q: "완벽을 기다리기보다 빠르게 실행하여 피드백을 받는 편이다." },
            { cat: "단기 반복", q: "데이터에 기반하여 빠른 의사결정과 수정을 반복한다." },
            { cat: "단기 반복", q: "정기적인 회고를 통해 팀의 일하는 방식을 지속 개선한다." },
            { cat: "변화 & AI", q: "새로운 AI 기술 도입을 팀 생산성 향상의 기회로 여긴다." },
            { cat: "변화 & AI", q: "실패를 자산으로 규정하고 새로운 시도를 두려워 않게 한다." },
            { cat: "변화 & AI", q: "과거의 성공이 미래엔 안 통할 수 있음을 인정하고 학습한다." },
            { cat: "변화 & AI", q: "환경 변화 시 팀 전체가 민첩하게 방향을 전환하도록 독려한다." }
        ];

        const container = document.getElementById('app-container');

        // --- Init Auth (Rule 3) ---
        async function init() {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (err) {
                console.error("인증 오류:", err);
            }

            onAuthStateChanged(auth, (u) => { 
                if (u) {
                    user = u;
                    renderStart();
                    setupAdminListener();
                }
            });
        }

        // --- Setup Data Listener after Auth (Rule 3) ---
        function setupAdminListener() {
            if (!user) return;
            if (unsubscribeAdmin) unsubscribeAdmin();

            const q = query(collection(db, 'artifacts', appId, 'public', 'data', 'leadership_results'));
            unsubscribeAdmin = onSnapshot(q, (snapshot) => {
                allResults = snapshot.docs.map(doc => doc.data());
                // 관리자 화면이 열려있는 경우 실시간 업데이트를 위해 재렌더링
                const adminList = document.getElementById('admin-list-container');
                if (adminList) renderAdmin();
            }, (error) => {
                console.error("데이터 수신 오류:", error);
            });
        }

        window.startApp = () => {
            userName = document.getElementById('name-input').value.trim();
            if(!userName) return alert("성함을 입력해주세요.");
            renderQuiz();
        };

        function renderStart() {
            container.innerHTML = `
                <div class="p-8 md:p-12 flex flex-col items-center justify-center flex-grow text-center animate-in">
                    <div class="w-20 h-20 bg-indigo-600 rounded-3xl flex items-center justify-center mb-8 shadow-xl shadow-indigo-100">
                        <svg class="text-white" width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 22c5.523 0 10-4.477 10-10S17.523 2 12 2 2 6.477 2 12s4.477 10 10 10z"/><path d="m9 12 2 2 4-4"/></svg>
                    </div>
                    <h1 class="text-3xl font-black text-slate-800 mb-4">AX 애자일 리더십 진단</h1>
                    <p class="text-slate-500 mb-10 leading-relaxed font-medium">변화에 민첩하게 대응하고 계신가요?<br/>당신의 리더십 스타일을 실시간으로 분석합니다.</p>
                    <div class="w-full space-y-4 max-w-xs">
                        <input id="name-input" type="text" placeholder="성함 입력" class="w-full p-5 bg-slate-50 border-2 border-slate-100 rounded-2xl focus:border-indigo-500 outline-none transition-all font-bold text-center">
                        <button onclick="startApp()" class="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-5 rounded-2xl shadow-lg transition-all">진단 시작하기</button>
                        <button onclick="renderAdmin()" class="w-full text-slate-400 text-xs font-bold hover:text-indigo-600 py-2">실시간 관리자 데이터 확인</button>
                    </div>
                </div>
            `;
        }

        function renderQuiz() {
            const q = QUESTIONS[currentIdx];
            container.innerHTML = `
                <div class="w-full bg-slate-100 h-2"><div class="bg-indigo-600 h-full transition-all" style="width:${((currentIdx+1)/QUESTIONS.length)*100}%"></div></div>
                <div class="p-8 md:p-12 flex-grow flex flex-col animate-in">
                    <div class="flex justify-between items-center mb-10">
                        <span class="px-4 py-1.5 bg-indigo-50 text-indigo-600 text-[10px] font-black rounded-full uppercase tracking-widest">${q.cat}</span>
                        <span class="text-slate-300 font-black text-xs">${currentIdx+1} / ${QUESTIONS.length}</span>
                    </div>
                    <h2 class="text-2xl font-bold text-slate-800 mb-12 leading-snug h-24">${q.q}</h2>
                    <div class="space-y-3 mt-auto">
                        ${[5, 4, 3, 2, 1].map(n => `
                            <button onclick="handleSelect(${n})" class="w-full p-5 border-2 border-slate-50 rounded-2xl hover:border-indigo-600 hover:bg-indigo-50 flex justify-between items-center group transition-all">
                                <span class="font-bold text-slate-700">${['전혀 아니다','아니다','보통이다','그렇다','매우 그렇다'][n-1]}</span>
                                <span class="w-8 h-8 rounded-lg bg-slate-100 group-hover:bg-indigo-600 group-hover:text-white flex items-center justify-center text-xs font-black">${n}</span>
                            </button>
                        `).join('')}
                    </div>
                </div>
            `;
        }

        window.handleSelect = async (n) => {
            answers.push(n);
            if(currentIdx < QUESTIONS.length - 1) {
                currentIdx++;
                renderQuiz();
            } else {
                renderLoading();
                const total = answers.reduce((a,b)=>a+b, 0);
                const catScores = {
                    "협업": answers.slice(0,4).reduce((a,b)=>a+b,0),
                    "권한": answers.slice(4,8).reduce((a,b)=>a+b,0),
                    "서번트": answers.slice(8,12).reduce((a,b)=>a+b,0),
                    "데이터": answers.slice(12,16).reduce((a,b)=>a+b,0),
                    "AI수용": answers.slice(16,20).reduce((a,b)=>a+b,0)
                };

                // Guard before Firestore operation (Rule 3)
                if(user) {
                    try {
                        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'leadership_results'), {
                            name: userName, 
                            totalScore: total, 
                            catScores, 
                            timestamp: serverTimestamp()
                        });
                    } catch (e) {
                        console.error("데이터 저장 실패:", e);
                    }
                }
                renderResult(total, catScores);
            }
        };

        function renderLoading() {
            container.innerHTML = `<div class="p-20 flex flex-col items-center justify-center flex-grow text-center"><div class="w-12 h-12 border-4 border-indigo-600 border-t-transparent rounded-full animate-spin mb-4"></div><p class="font-bold text-slate-600">결과를 저장하고 있습니다...</p></div>`;
        }

        function renderResult(total, cats) {
            let title = total >= 90 ? "애자일 마스터" : total >= 75 ? "애자일 프랙티셔너" : total >= 55 ? "과도기적 리더" : "전통적 관리자";
            container.innerHTML = `
                <div class="p-8 md:p-12 overflow-y-auto max-h-[90vh] animate-in text-center">
                    <p class="text-slate-400 font-bold mb-2">${userName} 님의 점수</p>
                    <h2 class="text-7xl font-black text-slate-900 mb-2">${total}<span class="text-2xl text-slate-300 ml-1">/100</span></h2>
                    <h3 class="text-2xl font-black text-indigo-600 mb-8">${title}</h3>
                    
                    <div class="bg-indigo-50 p-6 rounded-[2rem] text-left mb-6 border border-indigo-100">
                        <h4 class="font-black text-indigo-900 mb-2">💡 AX 리더십 가이드</h4>
                        <p class="text-xs text-slate-600 leading-relaxed font-medium">데이터가 성공적으로 저장되었습니다. AI 시대의 리더는 지시보다 질문을, 통제보다 지원을 우선해야 합니다. 현재 당신은 <strong>${title}</strong> 단계입니다.</p>
                    </div>

                    <button onclick="location.reload()" class="w-full py-5 bg-slate-900 text-white font-bold rounded-2xl shadow-xl transition-all mt-4">진단 완료</button>
                </div>
            `;
        }

        window.renderAdmin = () => {
            container.innerHTML = `
                <div id="admin-list-container" class="p-8 md:p-12 flex flex-col flex-grow animate-in">
                    <div class="flex justify-between items-center mb-8">
                        <h2 class="text-xl font-black text-slate-800">교육생 실시간 현황</h2>
                        <button onclick="renderStart()" class="text-slate-400 font-bold text-xs underline">닫기</button>
                    </div>
                    <div class="space-y-3 overflow-y-auto max-h-[400px] pr-2">
                        ${allResults.length === 0 ? '<p class="text-center text-slate-300 py-20 font-bold">참여자가 없거나 데이터를 불러오는 중입니다.</p>' : 
                          [...allResults].sort((a,b) => (b.timestamp?.seconds || 0) - (a.timestamp?.seconds || 0)).map(r => `
                            <div class="p-4 bg-slate-50 border border-slate-100 rounded-2xl flex justify-between items-center">
                                <span class="font-bold text-slate-700">${r.name}</span>
                                <span class="px-3 py-1 bg-white border border-slate-200 rounded-lg font-black text-indigo-600 text-sm">${r.totalScore}점</span>
                            </div>
                        `).join('')}
                    </div>
                </div>
            `;
        }

        init();
    </script>
</body>
</html>
