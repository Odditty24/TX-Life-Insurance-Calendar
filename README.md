<Morales Financial, LLC.>
<html lang="en">
<head>
    <meta charset="UTF-T">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Texas Life Insurance Exam Study Plan</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        .tab-active {
            border-color: #3b82f6; /* blue-500 */
            color: #3b82f6;
            font-weight: 600;
        }
        .task-item label:hover {
            text-decoration: line-through;
            cursor: pointer;
        }
        .task-item input[type="checkbox"]:checked + label {
            text-decoration: line-through;
            color: #6b7280; /* gray-500 */
        }
        /* Custom scrollbar for better aesthetics */
        ::-webkit-scrollbar {
            width: 8px;
        }
        ::-webkit-scrollbar-track {
            background: #f1f1f1;
            border-radius: 10px;
        }
        ::-webkit-scrollbar-thumb {
            background: #888;
            border-radius: 10px;
        }
        ::-webkit-scrollbar-thumb:hover {
            background: #555;
        }
        #loading-spinner, #gemini-loading-spinner {
            border: 4px solid rgba(0, 0, 0, 0.1);
            width: 36px;
            height: 36px;
            border-radius: 50%;
            border-left-color: #09f; /* A distinct blue for spinner */
            animation: spin 1s ease infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
        /* Modal styles */
        .modal {
            transition: opacity 0.25s ease;
        }
        .modal-content {
            transition: transform 0.25s ease;
        }
    </style>
</head>
<body class="bg-gray-100 text-gray-800 p-4 sm:p-6 md:p-8">
    <div class="max-w-4xl mx-auto bg-white shadow-xl rounded-lg p-6 md:p-8">
        <header class="mb-8 text-center">
            <h1 class="text-3xl sm:text-4xl font-bold text-blue-600">Texas Life Insurance Exam</h1>
            <p class="text-xl text-gray-600 mt-2">4-Week Interactive Study Plan</p>
            <div class="mt-2 text-sm text-gray-500">User ID: <span id="userIdDisplay" class="font-mono">Initializing...</span></div>
        </header>

        <div id="loadingOverlay" class="fixed inset-0 bg-gray-500 bg-opacity-75 flex items-center justify-center z-50">
            <div id="loading-spinner"></div>
            <p class="ml-3 text-white text-lg">Loading study plan...</p>
        </div>
        
        <div class="mb-6">
            <div class="flex justify-between mb-1">
                <span class="text-base font-medium text-blue-700">Overall Progress</span>
                <span id="progressText" class="text-sm font-medium text-blue-700">0%</span>
            </div>
            <div class="w-full bg-gray-200 rounded-full h-4">
                <div id="progressBar" class="bg-blue-600 h-4 rounded-full transition-all duration-500 ease-out" style="width: 0%"></div>
            </div>
        </div>

        <div class="mb-6 border-b border-gray-200">
            <nav id="weekTabs" class="flex flex-wrap -mb-px text-sm font-medium text-center" aria-label="Tabs">
                </nav>
        </div>

        <div id="weekContentContainer">
            </div>

        <div id="uiMessageBox" class="fixed bottom-5 right-5 bg-green-500 text-white p-4 rounded-lg shadow-md transition-opacity duration-300 opacity-0 hidden">
            <p id="uiMessageText"></p>
        </div>

        <div id="explanationModal" class="modal fixed inset-0 bg-gray-800 bg-opacity-75 flex items-center justify-center z-50 hidden opacity-0">
            <div class="modal-content bg-white p-6 rounded-lg shadow-xl max-w-xl w-full max-h-[80vh] overflow-y-auto transform scale-95">
                <div class="flex justify-between items-center mb-4">
                    <h3 id="explanationModalTitle" class="text-xl font-semibold text-blue-700">Topic Explanation</h3>
                    <button id="closeExplanationModalButton" class="text-gray-600 hover:text-gray-800 text-2xl font-bold">&times;</button>
                </div>
                <div id="explanationModalContentContainer" class="text-gray-700 prose max-w-none">
                    <div id="geminiLoadingState" class="flex flex-col items-center justify-center p-4 hidden">
                        <div id="gemini-loading-spinner" class="mb-2"></div>
                        <p>Fetching explanation from Gemini AI...</p>
                    </div>
                    <div id="geminiResponseContent"></div>
                </div>
            </div>
        </div>
    </div>

    <script type="module">
        // Firebase imports
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, setPersistence, browserLocalPersistence } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, onSnapshot, setDoc, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- IMPORTANT: Firebase Configuration ---
        // REPLACE THIS with your own Firebase project's configuration object!
        // You get this from the Firebase console when you add a web app to your project.
        const firebaseConfig = {
            apiKey: "YOUR_API_KEY_HERE", 
            authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
            projectId: "YOUR_PROJECT_ID_HERE",
            storageBucket: "YOUR_PROJECT_ID.appspot.com",
            messagingSenderId: "YOUR_MESSAGING_SENDER_ID_HERE",
            appId: "YOUR_WEB_APP_ID_HERE"
        };
        
        // --- IMPORTANT: Application ID ---
        // For your deployed version, set this to a unique fixed string.
        // This string is used in Firestore paths. Make sure it matches any paths in your security rules.
        // Example: const appId = 'myTexasStudyApp2025';
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-study-app-id'; 
        
        let fbApp;
        let fbAuth;
        let fbDb;
        let currentUserId = null;
        let taskStates = {}; 
        let unsubscribeFirestoreListener = null; 
        const FIRESTORE_PROGRESS_DOC_ID = `texasLifeExamPlanProgress_${appId}`; 

        // --- UI Elements ---
        const weekTabsContainer = document.getElementById('weekTabs');
        const weekContentContainer = document.getElementById('weekContentContainer');
        const progressBar = document.getElementById('progressBar');
        const progressText = document.getElementById('progressText');
        const userIdDisplay = document.getElementById('userIdDisplay');
        const loadingOverlay = document.getElementById('loadingOverlay');
        const uiMessageBox = document.getElementById('uiMessageBox');
        const uiMessageText = document.getElementById('uiMessageText');

        // Gemini Modal Elements
        const explanationModal = document.getElementById('explanationModal');
        const explanationModalTitle = document.getElementById('explanationModalTitle');
        const geminiLoadingState = document.getElementById('geminiLoadingState');
        const geminiResponseContent = document.getElementById('geminiResponseContent');
        const closeExplanationModalButton = document.getElementById('closeExplanationModalButton');

        // --- Study Plan Data ---
        const studyPlanData = [
            // ... (Study plan data remains the same as previous versions - ensure it's complete here) ...
            // Week 1
            {
                week: 1, focus: "Foundations & Core Life Insurance Products",
                days: [
                    { day: 1, topic: "Intro to Insurance & Basic Concepts", tasks: [{ id: "w1d1t1", description: "Read: Core Concepts (Risk, Peril, Hazard, Insurable Interest), Elements of a Legal Contract." }, { id: "w1d1t2", description: "Notes & Flashcards." }], notesAndTips: "Focus on understanding the \"why\" behind these fundamental principles." },
                    { day: 2, topic: "Underwriting & Policy Issue/Delivery", tasks: [{ id: "w1d2t1", description: "Read: Underwriting Process, Risk Classification, Application, Policy Issue & Delivery, Free Look." }, { id: "w1d2t2", description: "Notes & Flashcards." }], notesAndTips: "Understand the agent's role and responsibilities in this process." },
                    { day: 3, topic: "Term Life & Whole Life Insurance", tasks: [{ id: "w1d3t1", description: "Read: Detailed chapters on Term Life (Level, Decreasing, Renewable, Convertible) and Whole Life (Ordinary, Limited-Pay, Single Premium)." }, { id: "w1d3t2", description: "Notes & Flashcards." }, { id: "w1d3t3", description: "Practice Qs." }], notesAndTips: "Create a comparison chart for Term vs. Whole Life." },
                    { day: 4, topic: "Flexible Premium & Specialized Policies", tasks: [{ id: "w1d4t1", description: "Read: Universal Life, Variable Life (conceptual understanding), Joint Life, Survivorship Life, Credit Life, Juvenile Life." }, { id: "w1d4t2", description: "Notes & Flashcards." }], notesAndTips: "Focus on the unique features and uses of each policy type." },
                    { day: 5, topic: "Annuities", tasks: [{ id: "w1d5t1", description: "Read: Purpose, Phases (Accumulation, Annuitization), Types (Fixed, Variable, Immediate, Deferred), Payout Options." }, { id: "w1d5t2", description: "Notes & Flashcards." }, { id: "w1d5t3", description: "Practice Qs." }], notesAndTips: "Diagram the flow of an annuity. Understand its role in providing income." },
                    { day: 6, topic: "Review: Week 1 (Part 1)", tasks: [{ id: "w1d6t1", description: "Review all notes & flashcards from Days 1-5." }, { id: "w1d6t2", description: "Complete a practice quiz covering these topics." }], notesAndTips: "Identify any areas of confusion and re-read those sections." },
                    { day: 7, topic: "Rest or Light Review", tasks: [{ id: "w1d7t1", description: "Lightly review flashcards or challenging concepts." }], notesAndTips: "Don't burn out! Rest is important for retention." }
                ]
            },
            // Week 2
            {
                week: 2, focus: "Policy Details, Options, & General Concepts",
                days: [
                    { day: 1, topic: "Standard Life Policy Provisions", tasks: [{ id: "w2d1t1", description: "Read: Entire Contract, Insuring Clause, Consideration, Owner's Rights, Beneficiaries, Premium Modes, Grace Period, Reinstatement, Incontestability, Misstatement of Age/Sex, Suicide Clause." }, { id: "w2d1t2", description: "Notes & Flashcards." }], notesAndTips: "For each provision, understand its purpose and implications for the policyholder." },
                    { day: 2, topic: "Policy Options & Common Riders", tasks: [{ id: "w2d2t1", description: "Read: Nonforfeiture Options, Dividend Options, Policy Loans. Riders (Waiver of Premium, ADB, Guaranteed Insurability, Accelerated Benefits, LTC, etc.)." }, { id: "w2d2t2", description: "Notes & Flashcards." }, { id: "w2d2t3", description: "Practice Qs." }], notesAndTips: "Create scenarios where specific riders would be crucial." },
                    { day: 3, topic: "Tax Treatment & Group Life Insurance", tasks: [{ id: "w2d3t1", description: "Read: General tax implications (premiums, death benefits, cash values, dividends). Group Life (Master Contract, Certificate, Conversion)." }, { id: "w2d3t2", description: "Notes & Flashcards." }], notesAndTips: "Summarize key tax points. Understand group vs. individual coverage." },
                    { day: 4, topic: "Business Uses & Other Concepts", tasks: [{ id: "w2d4t1", description: "Read: Key Person, Buy-Sell Agreements. Basic understanding of Social Security survivor benefits. Suitability." }, { id: "w2d4t2", description: "Notes & Flashcards." }, { id: "w2d4t3", description: "Practice Qs." }], notesAndTips: "How does life insurance solve business needs? What are ethical considerations?" },
                    { day: 5, topic: "Review: Week 2 (Part 2)", tasks: [{ id: "w2d5t1", description: "Review all notes & flashcards from Days 1-4 of this week." }, { id: "w2d5t2", description: "Complete a practice quiz covering these topics." }], notesAndTips: "Focus on application-based questions for provisions and riders." },
                    { day: 6, topic: "Mid-Plan Review: General Knowledge", tasks: [{ id: "w2d6t1", description: "Review all material from Weeks 1 & 2." }, { id: "w2d6t2", description: "Take a comprehensive practice quiz on all General Knowledge topics." }], notesAndTips: "Identify overall strengths and weaknesses in general insurance concepts." },
                    { day: 7, topic: "Rest or Light Review", tasks: [{ id: "w2d7t1", description: "Lightly review flashcards or challenging concepts." }], notesAndTips: "Prepare for the shift to state-specific laws." }
                ]
            },
            // Week 3
            {
                week: 3, focus: "Texas Laws & Regulations",
                days: [
                    { day: 1, topic: "TDI, Commissioner & Licensing", tasks: [{ id: "w3d1t1", description: "Read: Texas Department of Insurance (TDI), Powers of Commissioner, Licensing (types, qualifications, renewal, CE requirements)." }, { id: "w3d1t2", description: "Notes & Flashcards." }], notesAndTips: "Memorize timelines, fee amounts (if specified in materials), and CE hours." },
                    { day: 2, topic: "Agent Duties & Unfair Trade Practices (1)", tasks: [{ id: "w3d2t1", description: "Read: Fiduciary Duty, Ethics. Unfair Trade Practices: Misrepresentation, False Advertising, Defamation." }, { id: "w3d2t2", description: "Notes & Flashcards." }, { id: "w3d2t3", description: "Practice Qs." }], notesAndTips: "Understand the definitions and provide examples for each unfair practice." },
                    { day: 3, topic: "Unfair Trade Practices (2) & Marketing", tasks: [{ id: "w3d3t1", description: "Read: Unfair Trade Practices: Twisting, Rebating (Texas specifics), Unfair Discrimination, Coercion. Marketing & Advertising rules, Buyer's Guide, Policy Summary." }, { id: "w3d3t2", description: "Notes & Flashcards." }], notesAndTips: "Pay close attention to what constitutes illegal rebating in Texas." },
                    { day: 4, topic: "Texas Policy Provisions & Replacement", tasks: [{ id: "w3d4t1", description: "Read: Texas-specific rules for Free Look, Grace Period, Reinstatement, Policy Loans, Incontestability, Suicide Clause. Detailed regulations for Policy Replacement." }, { id: "w3d4t2", description: "Notes & Flashcards." }, { id: "w3d4t3", description: "Practice Qs." }], notesAndTips: "Create a checklist for compliant policy replacement in Texas." },
                    { day: 5, topic: "Guaranty Association & Privacy", tasks: [{ id: "w3d5t1", description: "Read: Texas Life & Health Insurance Guaranty Association (purpose, limits, advertising prohibition). Privacy Laws (GLBA, Texas specific)." }, { id: "w3d5t2", description: "Notes & Flashcards." }], notesAndTips: "Memorize Guaranty Association coverage limits. Understand disclosure rules." },
                    { day: 6, topic: "Review: Week 3 (Texas Laws)", tasks: [{ id: "w3d6t1", description: "Review all notes & flashcards from Days 1-5 of this week." }, { id: "w3d6t2", description: "Complete a practice quiz focused exclusively on Texas laws." }], notesAndTips: "This section is detail-oriented. Repetition is key." },
                    { day: 7, topic: "Rest or Targeted Texas Law Review", tasks: [{ id: "w3d7t1", description: "Focus on any Texas laws or regulations you find confusing." }], notesAndTips: "Ensure you are comfortable with the nuances of state rules." }
                ]
            },
            // Week 4
            {
                week: 4, focus: "Final Review & Exam Simulation",
                days: [
                    { day: 1, topic: "Comprehensive Review: General Knowledge", tasks: [{ id: "w4d1t1", description: "Quickly review all notes, flashcards, and challenging topics from Weeks 1 & 2." }, { id: "w4d1t2", description: "Practice Qs on weak areas." }], notesAndTips: "Revisit areas where you scored lower in previous quizzes." },
                    { day: 2, topic: "Comprehensive Review: Texas Laws", tasks: [{ id: "w4d2t1", description: "Quickly review all notes, flashcards, and challenging topics from Week 3." }, { id: "w4d2t2", description: "Practice Qs on weak areas." }], notesAndTips: "Drill down on specific timelines, dollar amounts, and prohibited practices." },
                    { day: 3, topic: "Full Practice Exam #1", tasks: [{ id: "w4d3t1", description: "Take a full-length timed practice exam (aim for 80-90 questions in 2 hours)." }, { id: "w4d3t2", description: "Simulate exam conditions." }], notesAndTips: "Find a quiet space, no distractions." },
                    { day: 4, topic: "Analyze Practice Exam #1 & Targeted Review", tasks: [{ id: "w4d4t1", description: "Score the exam." }, { id: "w4d4t2", description: "Thoroughly review every incorrect answer and understand why." }, { id: "w4d4t3", description: "Re-study those topics." }], notesAndTips: "Don't just look at the right answer; understand the concept behind it." },
                    { day: 5, topic: "Full Practice Exam #2", tasks: [{ id: "w4d5t1", description: "Take another full-length timed practice exam (preferably a different one from a different source if possible)." }], notesAndTips: "Focus on improving timing and applying test-taking strategies." },
                    { day: 6, topic: "Analyze Practice Exam #2 & Final Polish", tasks: [{ id: "w4d6t1", description: "Score the exam." }, { id: "w4d6t2", description: "Review incorrect answers." }, { id: "w4d6t3", description: "Do a final quick review of all key concepts and Texas regulations." }], notesAndTips: "Make a short list of \"must-remember\" facts for a final glance." },
                    { day: 7, topic: "Light Review & Rest", tasks: [{ id: "w4d7t1", description: "Lightly glance over your most important notes/flashcards. Relax." }, { id: "w4d7t2", description: "Ensure you know the exam location and what to bring. Get a good night's sleep." }], notesAndTips: "Confidence is key! Trust your preparation." }
                ]
            }
        ];

        // --- UI Functions ---
        function showUIMessage(text, type = 'success') {
            uiMessageText.textContent = text;
            uiMessageBox.classList.remove('hidden', 'bg-green-500', 'bg-red-500', 'opacity-0');
            uiMessageBox.classList.add(type === 'success' ? 'bg-green-500' : 'bg-red-500');
            
            //offsetHeight to trigger reflow for transition
            uiMessageBox.offsetHeight; 
            uiMessageBox.classList.add('opacity-100');

            setTimeout(() => {
                uiMessageBox.classList.remove('opacity-100');
                // Add hidden after transition
                setTimeout(() => uiMessageBox.classList.add('hidden'), 300);
            }, 3000);
        }

        function renderTabs() {
            weekTabsContainer.innerHTML = ''; 
            studyPlanData.forEach((weekData, index) => {
                const tabButton = document.createElement('a');
                tabButton.href = '#';
                tabButton.textContent = `Week ${weekData.week}`;
                tabButton.classList.add('inline-block', 'p-4', 'border-b-2', 'border-transparent', 'rounded-t-lg', 'hover:text-gray-600', 'hover:border-gray-300', 'transition-colors', 'duration-150');
                if (index === 0) { 
                    tabButton.classList.add('tab-active');
                }
                tabButton.addEventListener('click', (e) => {
                    e.preventDefault();
                    renderWeekContent(weekData.week -1); 
                    document.querySelectorAll('#weekTabs a').forEach(btn => btn.classList.remove('tab-active'));
                    tabButton.classList.add('tab-active');
                });
                weekTabsContainer.appendChild(tabButton);
            });
        }

        function renderWeekContent(weekIndex) {
            const weekData = studyPlanData[weekIndex];
            if (!weekData) return;

            weekContentContainer.innerHTML = ''; 

            const focusEl = document.createElement('p');
            focusEl.classList.add('text-lg', 'font-semibold', 'text-gray-700', 'mb-4', 'italic');
            focusEl.textContent = `Focus: ${weekData.focus}`;
            weekContentContainer.appendChild(focusEl);

            weekData.days.forEach(dayData => {
                const dayContainer = document.createElement('div');
                dayContainer.classList.add('mb-6', 'p-4', 'border', 'border-gray-200', 'rounded-lg', 'shadow-sm', 'bg-gray-50');

                const dayHeaderContainer = document.createElement('div');
                dayHeaderContainer.classList.add('flex', 'justify-between', 'items-center', 'mb-3', 'flex-wrap');
                
                const dayHeader = document.createElement('h3');
                dayHeader.classList.add('text-xl', 'font-semibold', 'text-blue-700', 'mr-2'); // Added mr-2 for spacing
                dayHeader.textContent = `Day ${dayData.day}: ${dayData.topic}`;
                dayHeaderContainer.appendChild(dayHeader);

                const explainButton = document.createElement('button');
                explainButton.innerHTML = `Explain Topic ✨`;
                explainButton.classList.add('px-3', 'py-1', 'text-xs', 'font-medium', 'bg-purple-500', 'text-white', 'rounded-md', 'hover:bg-purple-600', 'focus:outline-none', 'focus:ring-2', 'focus:ring-purple-500', 'focus:ring-opacity-50', 'transition-colors', 'whitespace-nowrap');
                explainButton.onclick = () => showExplanationModal(dayData.topic);
                dayHeaderContainer.appendChild(explainButton);
                
                dayContainer.appendChild(dayHeaderContainer);

                const tasksList = document.createElement('ul');
                tasksList.classList.add('space-y-2', 'mb-3');
                dayData.tasks.forEach(task => {
                    const taskItem = document.createElement('li');
                    taskItem.classList.add('flex', 'items-center', 'task-item');
                    
                    const checkbox = document.createElement('input');
                    checkbox.type = 'checkbox';
                    checkbox.id = task.id;
                    checkbox.checked = taskStates[task.id] || false;
                    checkbox.classList.add('h-5', 'w-5', 'text-blue-600', 'border-gray-300', 'rounded', 'focus:ring-blue-500', 'mr-3', 'flex-shrink-0');
                    checkbox.addEventListener('change', () => handleTaskToggle(task.id));
                    
                    const label = document.createElement('label');
                    label.htmlFor = task.id;
                    label.textContent = task.description;
                    label.classList.add('text-gray-700', 'flex-1', 'min-w-0'); // Added min-w-0 for better wrapping with flex

                    taskItem.appendChild(checkbox);
                    taskItem.appendChild(label);
                    tasksList.appendChild(taskItem);
                });
                dayContainer.appendChild(tasksList);

                if (dayData.notesAndTips) {
                    const notesEl = document.createElement('p');
                    notesEl.classList.add('text-sm', 'text-gray-600', 'italic', 'mt-2', 'p-2', 'bg-blue-50', 'rounded');
                    notesEl.innerHTML = `<strong>Notes & Tips:</strong> ${dayData.notesAndTips}`;
                    dayContainer.appendChild(notesEl);
                }
                weekContentContainer.appendChild(dayContainer);
            });
        }

        // --- Firebase & Task Logic ---
        async function handleTaskToggle(taskId) {
            taskStates[taskId] = !taskStates[taskId]; 
            updateProgressBar();
            await saveTaskStatesToFirestore(); 
            
            const currentActiveTab = document.querySelector('#weekTabs a.tab-active');
            if (currentActiveTab) {
                const weekText = currentActiveTab.textContent; 
                const weekNum = parseInt(weekText.split(' ')[1]);
                if (!isNaN(weekNum)) {
                    renderWeekContent(weekNum - 1); // Re-render to apply strikethrough style
                }
            }
        }

        function updateProgressBar() {
            let totalTasks = 0;
            let completedTasks = 0;
            studyPlanData.forEach(week => {
                week.days.forEach(day => {
                    day.tasks.forEach(task => {
                        totalTasks++;
                        if (taskStates[task.id]) {
                            completedTasks++;
                        }
                    });
                });
            });

            const percentage = totalTasks > 0 ? Math.round((completedTasks / totalTasks) * 100) : 0;
            progressBar.style.width = `${percentage}%`;
            progressText.textContent = `${percentage}%`;
        }

        async function initializeFirebaseAndLoadData() {
            try {
                // Check if firebaseConfig has placeholder values
                if (firebaseConfig.apiKey === "YOUR_API_KEY_HERE" || firebaseConfig.projectId === "YOUR_PROJECT_ID_HERE") {
                    console.warn("Firebase is not configured. Using local mode. Progress will not be saved online.");
                    showUIMessage("Firebase not configured. Progress is local only.", "error");
                    userIdDisplay.textContent = "Local Mode";
                    loadingOverlay.style.display = 'none';
                    renderAppContent(); // Render UI with local state
                    return; 
                }

                fbApp = initializeApp(firebaseConfig);
                fbAuth = getAuth(fbApp);
                fbDb = getFirestore(fbApp);
                // setLogLevel('debug'); 

                await setPersistence(fbAuth, browserLocalPersistence); 

                onAuthStateChanged(fbAuth, async (user) => {
                    if (user) {
                        currentUserId = user.uid;
                        console.log("User is signed in with UID:", currentUserId);
                        userIdDisplay.textContent = currentUserId.substring(0, 8) + "..."; // Display partial UID
                        await loadTaskStatesFromFirestore();
                    } else {
                        console.log("User is not signed in. Attempting sign-in.");
                        userIdDisplay.textContent = "Signing in...";
                        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                            try {
                                await signInWithCustomToken(fbAuth, __initial_auth_token);
                                console.log("Signed in with custom token.");
                            } catch (error) {
                                console.error("Error signing in with custom token, falling back to anonymous:", error);
                                await signInAnonymously(fbAuth);
                            }
                        } else {
                            await signInAnonymously(fbAuth);
                        }
                        // onAuthStateChanged will trigger again with the new user state
                    }
                });

            } catch (error) {
                console.error("Error initializing Firebase:", error);
                showUIMessage(`Firebase Init Error: ${error.message}. Progress local.`, 'error');
                userIdDisplay.textContent = "Firebase Error";
                loadingOverlay.style.display = 'none'; 
                renderAppContent();
            }
        }
        
        function renderAppContent() {
            renderTabs();
            // Attempt to get active tab, default to 0 if none (shouldn't happen if renderTabs works)
            const activeTabLink = weekTabsContainer.querySelector('a.tab-active');
            let initialWeekIndex = 0;
            if (activeTabLink) {
                const weekNum = parseInt(activeTabLink.textContent.split(' ')[1]);
                if (!isNaN(weekNum)) initialWeekIndex = weekNum - 1;
            }
            renderWeekContent(initialWeekIndex); 
            updateProgressBar();
        }


        async function loadTaskStatesFromFirestore() {
            if (!currentUserId || !fbDb) {
                console.warn("Cannot load tasks: User not authenticated or Firestore not initialized.");
                loadingOverlay.style.display = 'none';
                renderAppContent(); // Render with default (empty) taskStates
                return;
            }
            
            // Path: artifacts/{appId}/users/{userId}/studyPlanProgress/{docId_with_appId}
            const docPath = `artifacts/${appId}/users/${currentUserId}/studyPlanProgress/${FIRESTORE_PROGRESS_DOC_ID}`;
            console.log("Attempting to load from Firestore path:", docPath);
            const docRef = doc(fbDb, docPath);

            if (unsubscribeFirestoreListener) {
                unsubscribeFirestoreListener(); // Detach previous listener
            }

            unsubscribeFirestoreListener = onSnapshot(docRef, (docSnap) => {
                loadingOverlay.style.display = 'flex'; 
                if (docSnap.exists()) {
                    const data = docSnap.data();
                    taskStates = data.taskStates || {};
                    console.log("Task states loaded from Firestore:", Object.keys(taskStates).length, "states");
                } else {
                    console.log("No task states found in Firestore. Initializing with defaults for path:", docPath);
                    taskStates = {}; // Reset if no doc exists
                    studyPlanData.forEach(week => week.days.forEach(day => day.tasks.forEach(task => taskStates[task.id] = false)));
                }
                renderAppContent();
                loadingOverlay.style.display = 'none'; 
            }, (error) => {
                console.error("Error getting task states from Firestore:", error);
                showUIMessage(`Error loading progress: ${error.message}`, 'error');
                taskStates = {}; // Fallback to empty on error
                studyPlanData.forEach(week => week.days.forEach(day => day.tasks.forEach(task => taskStates[task.id] = false)));
                loadingOverlay.style.display = 'none';
                renderAppContent();
            });
        }

        async function saveTaskStatesToFirestore() {
            if (!currentUserId || !fbDb) {
                console.warn("Cannot save tasks: User not authenticated or Firestore not initialized.");
                showUIMessage("Error: Not connected. Progress not saved.", 'error');
                return;
            }
            const docPath = `artifacts/${appId}/users/${currentUserId}/studyPlanProgress/${FIRESTORE_PROGRESS_DOC_ID}`;
            const docRef = doc(fbDb, docPath);
            try {
                await setDoc(docRef, { taskStates });
                console.log("Task states saved to Firestore path:", docPath);
                showUIMessage("Progress saved!", 'success');
            } catch (error) {
                console.error("Error saving task states to Firestore:", error);
                showUIMessage(`Error saving: ${error.message}`, 'error');
            }
        }

        // --- Gemini API Functions ---
        function showExplanationModal(topic) {
            explanationModalTitle.textContent = `Explanation for: ${topic}`;
            geminiResponseContent.innerHTML = ''; // Clear previous content
            geminiLoadingState.classList.remove('hidden'); 
            
            explanationModal.classList.remove('hidden', 'opacity-0');
            explanationModal.querySelector('.modal-content').classList.remove('scale-95');
            explanationModal.classList.add('opacity-100');
            explanationModal.querySelector('.modal-content').classList.add('scale-100');

            fetchExplanationFromGemini(topic);
        }

        function hideExplanationModal() {
            explanationModal.classList.add('opacity-0');
            explanationModal.querySelector('.modal-content').classList.add('scale-95');
            setTimeout(() => {
                explanationModal.classList.add('hidden');
                explanationModal.querySelector('.modal-content').classList.remove('scale-100'); 
            }, 250); 
        }

        async function fetchExplanationFromGemini(topic) {
            const prompt = `Explain the topic "${topic}" for someone preparing for the Texas Life Insurance Exam. Focus on the most important key concepts, definitions, and principles an aspiring agent needs to know for this specific topic to pass the exam. Keep the explanation concise (around 200-300 words), easy to understand, and well-formatted for web display (use paragraphs, and bullet points or numbered lists if appropriate for clarity).`;
            
            let chatHistory = [{ role: "user", parts: [{ text: prompt }] }];
            const payload = { contents: chatHistory };
            const geminiApiKey = ""; // This will be handled by the environment if applicable
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${geminiApiKey}`;

            try {
                const response = await fetch(apiUrl, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                geminiLoadingState.classList.add('hidden'); 

                if (!response.ok) {
                    const errorData = await response.json().catch(() => ({ error: { message: "Unknown API error, non-JSON response" }})); // Catch if response is not JSON
                    console.error("Gemini API Error. Status:", response.status, "Response:", errorData);
                    geminiResponseContent.innerHTML = `<p class="text-red-500">Error fetching explanation: ${errorData.error?.message || response.statusText}</p><p class="text-xs text-gray-500">Make sure API key is valid and model is accessible.</p>`;
                    return;
                }

                const result = await response.json();
                
                if (result.candidates && result.candidates.length > 0 &&
                    result.candidates[0].content && result.candidates[0].content.parts &&
                    result.candidates[0].content.parts.length > 0 &&
                    result.candidates[0].content.parts[0].text) {
                    let text = result.candidates[0].content.parts[0].text;
                    // Basic formatting for web display
                    text = text.replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>'); // Bold
                    text = text.replace(/\*(.*?)\*/g, '<em>$1</em>');     // Italics (careful not to over-match with bold)
                    text = text.replace(/^- /gm, '&bull; ').replace(/^\* /gm, '&bull; '); // Basic bullets
                    text = text.replace(/\n\n/g, '</p><p>').replace(/\n/g, '<br>');
                    geminiResponseContent.innerHTML = `<p>${text}</p>`;
                } else {
                    geminiResponseContent.innerHTML = "<p>No explanation received or unexpected response format from Gemini.</p>";
                    console.warn("Unexpected Gemini response structure:", result);
                }
            } catch (error) {
                console.error("Error calling Gemini API:", error);
                geminiLoadingState.classList.add('hidden');
                geminiResponseContent.innerHTML = `<p class="text-red-500">Failed to fetch explanation. Check console for details: ${error.message}</p>`;
            }
        }

        // Event listener for closing the modal
        closeExplanationModalButton.addEventListener('click', hideExplanationModal);
        explanationModal.addEventListener('click', (event) => {
            if (event.target === explanationModal) { // Click on overlay
                hideExplanationModal();
            }
        });


        // --- Initial Load ---
        document.addEventListener('DOMContentLoaded', () => {
            loadingOverlay.style.display = 'flex'; 
            initializeFirebaseAndLoadData();
        });

    </script>
</body>
</html>
