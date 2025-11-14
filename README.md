<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>John's AI Chat</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom scrollbar for a cleaner look */
        .chat-container::-webkit-scrollbar {
            width: 8px;
        }
        .chat-container::-webkit-scrollbar-thumb {
            background-color: #4b5563; /* Gray 600 */
            border-radius: 10px;
        }
        .chat-container::-webkit-scrollbar-track {
            background-color: #1f2937; /* Gray 800 */
        }
        /* Mobile-friendly fixed height for chat window */
        .chat-container {
            height: calc(100vh - 120px); /* Full height minus header and input bar */
            max-height: 80vh; /* Max height limit */
        }
    </style>
    <script>
        tailwind.config = {
            theme: {
                extend: {
                    fontFamily: {
                        sans: ['Inter', 'sans-serif'],
                    },
                }
            }
        }
    </script>
</head>
<body class="bg-gray-900 text-gray-100 font-sans p-2 h-screen overflow-hidden flex items-center justify-center">

    <div id="app" class="w-full max-w-lg mx-auto bg-gray-800 rounded-xl shadow-2xl flex flex-col h-full sm:h-auto">
        
        <!-- Header -->
        <header class="p-4 border-b border-gray-700 flex justify-between items-center">
            <h1 class="text-xl font-bold flex items-center text-indigo-400">
                <svg class="w-6 h-6 mr-2" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 3v2m6-2v2m-6 14v2m6-2v2M5 9H3m2 6H3m18-6h-2m2 6h-2M7 19h10a2 2 0 002-2V7a2 2 0 00-2-2H7a2 2 0 00-2 2v10a2 2 0 002 2zM9 9h6v6H9V9z"></path></svg>
                John's AI
            </h1>
            <!-- NEW: Clear History Button -->
            <button id="clear-btn" class="text-sm px-3 py-1 bg-gray-700 text-gray-300 rounded-lg hover:bg-red-700 hover:text-white transition duration-200 shadow-md flex items-center">
                <svg class="w-4 h-4 mr-1" fill="none" stroke="currentColor" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"></path></svg>
                Clear
            </button>
        </header>
        <p class="text-sm text-gray-400 px-4 pt-1">Ask me anything! I use Google Search to stay current.</p>

        <!-- Chat History Container -->
        <div id="chat-container" class="flex-grow overflow-y-auto p-4 space-y-4 chat-container">
            <!-- Initial welcome message -->
            <!-- The welcome message will be inserted by JS on load -->
        </div>

        <!-- Input Area -->
        <div class="p-4 border-t border-gray-700 flex items-center">
            <input type="text" id="user-input" placeholder="Type your message..." class="flex-grow p-3 bg-gray-700 border border-gray-600 rounded-lg focus:outline-none focus:ring-2 focus:ring-indigo-500 text-white placeholder-gray-400 transition" autocomplete="off">
            <button id="send-btn" class="ml-3 p-3 bg-indigo-600 hover:bg-indigo-700 text-white rounded-lg shadow-md transition duration-200 disabled:opacity-50 flex items-center justify-center" disabled>
                <span id="send-text">Send</span>
                <span id="loading-spinner" class="hidden">
                    <svg class="animate-spin h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
                    </svg>
                </span>
            </button>
        </div>
    </div>

    <script>
        // --- Core Configuration and Utility Functions ---

        const API_URL_BASE = "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent";
        // API Key is automatically provided by the Canvas environment if left empty
        const API_KEY = ""; 

        const systemPrompt = "You are a friendly, concise, and highly effective AI assistant named John's AI. Your goal is to provide clear, accurate, and helpful information. Use Google Search grounding to ensure factual accuracy when answering current event or knowledge-based questions. Always cite your sources concisely below your answer if grounding is used. Your response should be in Markdown format.";

        const chatContainer = document.getElementById('chat-container');
        const userInput = document.getElementById('user-input');
        const sendBtn = document.getElementById('send-btn');
        const loadingSpinner = document.getElementById('loading-spinner');
        const sendText = document.getElementById('send-text');
        // NEW: Clear Button reference
        const clearBtn = document.getElementById('clear-btn'); 

        let chatHistory = [];
        
        // Helper to convert Markdown to HTML (basic implementation)
        function formatText(text) {
            // Basic markdown-to-html conversion for a cleaner output
            text = text.replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>'); // Bold
            text = text.replace(/\*(.*?)\*/g, '<em>$1</em>'); // Italic
            text = text.replace(/^- (.*)/gm, '<li>$1</li>'); // List items (if any)
            if (text.includes('<li>')) {
                text = '<ul>' + text + '</ul>';
            }
            return text.replace(/\n/g, '<br>'); // New lines
        }

        // Add initial welcome message
        function addWelcomeMessage() {
            addMessage("Hello! I'm **John's AI**. I can help you summarize information, answer questions, and even write drafts. What's on your mind?", false);
        }

        // Add a message to the chat container
        function addMessage(text, isUser, sources = []) {
            const messageWrapper = document.createElement('div');
            messageWrapper.className = `flex ${isUser ? 'justify-end' : 'justify-start'}`;

            const messageBubble = document.createElement('div');
            // User message: indigo background, rounded-tr-none
            // AI message: gray background, rounded-tl-none
            messageBubble.className = `p-3 rounded-xl max-w-xs sm:max-w-md shadow-lg break-words ${
                isUser 
                ? 'bg-indigo-500 text-white rounded-tr-none' 
                : 'bg-gray-700 text-gray-100 rounded-tl-none'
            }`;

            messageBubble.innerHTML = formatText(text);

            if (!isUser && sources.length > 0) {
                const sourceList = document.createElement('div');
                sourceList.className = 'mt-2 pt-2 border-t border-gray-600 text-xs text-gray-400';
                sourceList.innerHTML = '<strong>Sources:</strong><ul>' + sources.map(source => 
                    `<li><a href="${source.uri}" target="_blank" class="text-indigo-300 hover:text-indigo-200 truncate block">${source.title}</a></li>`
                ).join('') + '</ul>';
                messageBubble.appendChild(sourceList);
            }

            messageWrapper.appendChild(messageBubble);
            chatContainer.appendChild(messageWrapper);
            
            // Scroll to the bottom of the chat
            chatContainer.scrollTop = chatContainer.scrollHeight;
        }

        // NEW: Function to clear the history
        function clearHistory() {
            // 1. Reset the chat history array
            chatHistory = [];
            // 2. Remove all chat bubbles from the DOM
            chatContainer.innerHTML = '';
            // 3. Re-add the welcome message
            addWelcomeMessage();
            // 4. Focus the input field
            userInput.focus();
        }

        // Exponential backoff fetch helper
        async function exponentialBackoffFetch(url, options, maxRetries = 5, delay = 1000) {
            for (let i = 0; i < maxRetries; i++) {
                try {
                    const response = await fetch(url, options);
                    if (response.ok) {
                        return response;
                    }
                    // Handle rate limiting (429) or server errors (5xx)
                    if (response.status === 429 || response.status >= 500) {
                        throw new Error(`Server error or rate limit hit: ${response.status}`);
                    }
                    // Handle other non-ok responses immediately (e.g., 400 Bad Request)
                    const errorJson = await response.json();
                    console.error("API Error Response:", errorJson);
                    return response; // Return the non-retriable error
                } catch (error) {
                    if (i === maxRetries - 1) {
                        throw error; // Re-throw the error on the last attempt
                    }
                    const waitTime = delay * Math.pow(2, i) + Math.random() * 1000;
                    // console.log(`Retry ${i + 1} of ${maxRetries} after ${waitTime}ms`);
                    await new Promise(resolve => setTimeout(resolve, waitTime));
                }
            }
        }

        // Call the Gemini API
        async function callGeminiAPI(query) {
            
            // Add user query to history
            chatHistory.push({ role: "user", parts: [{ text: query }] });
            
            const payload = {
                contents: chatHistory,
                // Enable Google Search grounding tool
                tools: [{ "google_search": {} }],
                // System instruction sets the AI's persona and rules
                systemInstruction: {
                    parts: [{ text: systemPrompt }]
                }
            };

            const url = `${API_URL_BASE}?key=${API_KEY}`;
            
            try {
                const response = await exponentialBackoffFetch(url, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(payload)
                });

                if (!response.ok) {
                    const errorText = await response.text();
                    throw new Error(`API call failed with status ${response.status}: ${errorText}`);
                }

                const result = await response.json();
                const candidate = result.candidates?.[0];
                let responseText = "Sorry, I couldn't generate a response. Please try again.";
                let sources = [];
                
                if (candidate && candidate.content?.parts?.[0]?.text) {
                    responseText = candidate.content.parts[0].text;

                    // Extract grounding sources
                    const groundingMetadata = candidate.groundingMetadata;
                    if (groundingMetadata && groundingMetadata.groundingAttributions) {
                        sources = groundingMetadata.groundingAttributions
                            .map(attribution => ({
                                uri: attribution.web?.uri,
                                title: attribution.web?.title,
                            }))
                            .filter(source => source.uri && source.title);
                    }
                }

                // Add AI response to history
                chatHistory.push({ role: "model", parts: [{ text: responseText }] });
                
                addMessage(responseText, false, sources);

            } catch (error) {
                console.error("Gemini API Error:", error);
                addMessage("Oops! There was an issue connecting to the AI. Please check the console for details.", false);
            }
        }

        // Main function to handle sending the message
        async function sendMessage() {
            const query = userInput.value.trim();
            if (query === "") return;

            // 1. Clear input and disable button
            userInput.value = '';
            sendBtn.disabled = true;
            sendText.classList.add('hidden');
            loadingSpinner.classList.remove('hidden');

            // 2. Display user message
            addMessage(query, true);

            // 3. Call the AI
            await callGeminiAPI(query);

            // 4. Re-enable input and button
            sendBtn.disabled = false;
            sendText.classList.remove('hidden');
            loadingSpinner.classList.add('hidden');
            userInput.focus();
        }

        // Setup event listeners
        function setupChat() {
            // Initial welcome message
            addWelcomeMessage();
            
            // Enable button only when input has content
            userInput.addEventListener('input', () => {
                sendBtn.disabled = userInput.value.trim() === '';
            });
            
            // Enter key to send
            userInput.addEventListener('keypress', (e) => {
                if (e.key === 'Enter' && !sendBtn.disabled) {
                    e.preventDefault();
                    sendMessage();
                }
            });

            // Send button click
            sendBtn.addEventListener('click', sendMessage);

            // NEW: Clear button click
            clearBtn.addEventListener('click', clearHistory);

            // Initial focus
            userInput.focus();
            // Enable button initially if there's any pre-filled content (shouldn't be, but safe practice)
            sendBtn.disabled = userInput.value.trim() === '';
        }

        // Initialize the app when the window loads
        window.onload = setupChat;

    </script>
</body>
</html>

